# Provenance Guard

> ### 👋 Start here
>
> **New to this repo? Read [RUNNING.md](RUNNING.md) first.**
>
> Once `python test.py` passes, in one terminal:
>
> ```bash
> python app.py
> ```
>
> and in another:
>
> ```bash
> curl -X POST http://127.0.0.1:5000/ping \
>   -H "Content-Type: application/json" \
>   -d '{"message": "hello"}'
> ```
>
> That's a web service. Something sent it text and it sent something back.
> Everything this unit is filling in the middle.
>
> **The rest of this file is your submission.** Fill it in as you go.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     Unit 7 asks for the first five sections. Unit 8 adds the five below.

     Everything is pasted as TEXT. A pasted curl response gets full credit;
     a picture of your terminal gets none.
     ───────────────────────────────────────────────────────────────────────── -->

<!-- ═══════════════════════ UNIT 7 — THE BUILD ═══════════════════════ -->

## What This Does

Provenance Guard is a small Flask backend a writing site could plug into. It takes a piece of text and returns a guess about whether a person or AI wrote it, a confidence score, and a plain-English label. A writer who disagrees can appeal. A submission takes this path (`app.py::submit`):

1. **Request arrives.** A client sends `POST /submit` with JSON `{"text": "...", "creator_id": "..."}`. If `text` is missing, empty or not a string, the service returns a 400 error and stops.
2. **Signal one, the model signal.** `detector.model_signal` runs the text through a local GPT-2 model and returns a 0–1 score for how predictable it is. Higher means more likely AI.
3. **Signal two, the style signal.** `stylometry.style_signal` measures the shape of the text (sentence length spread, vocabulary variety, punctuation) without reading its meaning, and returns a 0–1 score in the same direction.
4. **Combine.** `scoring.combine_signals` takes a weighted average, 0.6 × model + 0.4 × style, into one combined score (returned as `confidence`).
5. **Label.** `scoring.score_to_label` turns the combined score into a guess (`human`, `unsure` or `ai`) and the sentence a reader sees. Scores up to 0.35 are human, 0.65 and above are AI, and everything in between is unsure.
6. **Log.** `audit.log_decision` writes a new `content_id`, the creator, both signal scores, the combined score, the guess, the label and a status of `decided` to `logs/audit.jsonl`.
7. **Respond.** The service returns the `content_id`, guess, confidence, label and both signal scores as JSON.

If the writer disagrees, they send `POST /appeal` with the `content_id` and their reasoning. The service checks the id exists, logs the appeal next to the original decision, and sets the status to `under_review`. Nothing is re-checked automatically.

---

## Signals and Scoring

<!-- For each signal: what property it measures, why that might differ between
     human and AI writing, and WHAT IT CAN'T SEE.

     The blind spot is not optional. If you can't name it you don't understand
     the signal yet — and unit 8's attack set is built out of blind spots. -->

### Signal one — the model signal

**What it measures:** How predictable the text is to a small language model (GPT-2) running locally (`detector.py::model_signal`). Text the model finds easy to predict (low perplexity) scores closer to 1 (more likely AI).

**Why that might differ between human and AI writing:** Generated text is built by picking likely next words, so it tends to be more predictable to another model. Human writing has more surprising word choices, odd phrasing and personal detail.

**What it can't see:** It only measures predictability, not authorship. Formal, conventional or heavily edited human writing is also predictable, so it can be read as AI (my formal-human test scored 0.727). Lightly edited or paraphrased AI text becomes less predictable and can slip through. It also only reads the first ~400 tokens, and GPT-2 is a small, old model, so text from newer models may not look unusual to it.

### Signal two — the style signal

**What it measures:** The shape of the text without understanding it (`stylometry.py::style_signal`). It averages three measures, each scaled to 0–1 with higher meaning more AI-like: sentence length spread (stdev ÷ mean; low = AI), type-token ratio (unique words ÷ total words; low = AI), and punctuation density (commas, semicolons, dashes, etc. per word; low = AI).

**Why that might differ between human and AI writing:** AI prose tends toward even, mid-length sentences, a narrower vocabulary and plain punctuation. Human writing is lumpier, with a three-word sentence next to a forty-word one, and more varied punctuation.

**What it can't see:** It cannot read, so it has no idea what the text means. Anything that changes the shape without changing the substance moves it: adding typos or dashes, splitting sentences, or pasting in a quotation makes AI text look human. Plain, short-sentence human writing looks AI-like. Type-token ratio is also unreliable on short texts (it is near 1 for almost anything under ~50 words) and drops as text gets longer, and a one-sentence submission has no spread, so that measure returns a neutral 0.5. In my tests it barely moved on AI text (0.391 on a clearly-AI paragraph), so it adds little on its own.

### The combining rule

<!-- Write the rule in words, then give the numbers. What happens when the two
     signals disagree? -->

**The rule:** A plain weighted average of the two signals. Both point the same way (higher = more likely AI), so the combined score is also 0 (confidently human) to 1 (confidently AI). If the signals disagree, the average lands in the middle, which I treat as "unsure" rather than picking a winner.

**The numbers:** `combined = 0.6 × model_score + 0.4 × style_score` (`WEIGHT_MODEL_SIGNAL = 0.6`, `WEIGHT_STYLE_SIGNAL = 0.4` in `config.py`). I weight the model signal higher because it reads meaning and separated my test inputs more than the style signal, which only sees the shape of the text and, on short inputs, barely moves.

**Where it lives:** `scoring.py::combine_signals`

<!-- ⚠️ The grader checks your code against that line. If your rule lives
     somewhere else, say where. -->

**Four calibration inputs (both signals printed separately):**

| Input | model_score | style_score | combined |
|---|---|---|---|
| Clearly AI (generic business-AI paragraph) | 0.889 | 0.391 | 0.690 |
| Clearly human (casual, messy anecdote) | 0.232 | 0.000 | 0.139 |
| Formal human (committee-minutes style) | 0.727 | 0.512 | 0.641 |
| Lightly-edited AI (same AI text, simplified) | 0.714 | 0.441 | 0.605 |

(Combined scores use the 0.6 / 0.4 weights.) Clearly-AI and clearly-human text land 0.55 apart (0.690 vs 0.139), so the score separates them. The two cases that matter are not well separated: formal human writing (0.641) is just under the 0.65 AI threshold, close to clearly AI text (0.690) and above lightly-edited AI (0.605). That is a false-positive risk, and it comes mostly from the model signal, which reads formal prose as predictable. Giving that signal more weight (0.5 to 0.6) made this slightly worse for formal human writing (0.619 to 0.641) in exchange for pushing clearly-AI text over the AI threshold.

**A case where my two signals split, and what my rule does with it:** On the clearly-AI paragraph, the model signal is confident (0.889) but the style signal is only 0.391, a gap of about 0.5. The style signal is weak here because the paragraph has varied sentence lengths and plenty of commas, so it doesn't look "evenly shaped". With the 0.6 / 0.4 weights the average is 0.690, just over the 0.65 AI threshold, so it gets the high-confidence AI label, but only narrowly: the disagreement pulls it toward the middle, and with equal weights (0.640) it would have been "unsure". The same weighting leaves formal human writing at 0.641, one step below the AI line, so the margin against a false accusation is thin.



---

## Label Variants

<!-- The exact text of all three labels, as a reader would see them, and the
     score range each covers.

     Write them for someone who has never heard the word "threshold". -->

| Label | Score range | The exact text a reader sees |
|---|---|---|
| high-confidence human | 0.00 – 0.35 | We think this was probably written by a person. |
| unsure | 0.35 – 0.65 | We can't tell whether this was written by a person or by AI. Nothing is being held against you. If you'd like, you can tell us more about how you wrote it. |
| high-confidence AI | 0.65 – 1.00 | Our checks suggest this was probably written with the help of AI. We can be wrong. If you wrote it yourself, you can appeal and a person will take a look. |

The ranges come from `scoring.label_ranges()` and `config.py` (`HUMAN_THRESHOLD = 0.35`, `AI_THRESHOLD = 0.65`). I kept the starter's values. All three labels are reachable: in my runs the clearly-human sample scored 0.318, the clearly-AI sample 0.602 and a stronger AI paragraph 0.690.

**Why I worded the "unsure" one this way:** The person reading it may have written every word themselves, so it says plainly that we don't know and that nothing is being held against them, and it never says "AI" or "suspicious". The last sentence gives them a next step without implying they did something wrong. Even the AI label says "probably", admits we can be wrong, and points to the appeal.



---

## Sample Run

<!-- One submission and one appeal, pasted as text: the request, the response,
     and the log entries. -->

**A submission**

<!-- The response needs content_id, guess, confidence, label, AND both
     signal scores - model_score and style_score. Unit 8's tools read those
     two off the response, so leaving them out costs you next unit's
     diagnostics. -->

Request body (`samples/clearly_ai.json`):

```json
{
  "text": "The importance of effective time management cannot be overstated in today's fast-paced work environment. Professionals who develop strong organizational habits are better positioned to meet deadlines and reduce stress. There are several key strategies that can help individuals improve their productivity. First, prioritization is essential. By identifying the most important tasks and addressing them first, workers can ensure that critical objectives are met. Second, minimizing distractions plays a significant role. Setting aside dedicated blocks of time for concentrated work is widely considered beneficial. Finally, regular breaks are important for maintaining performance.",
  "creator_id": "demo_instructor"
}
```

```bash
$ curl.exe -X POST http://127.0.0.1:5000/submit -H "Content-Type: application/json" -d @samples/clearly_ai.json
```

```json
{
  "confidence": 0.6021432885507968,
  "content_id": "c_dff3bb55f902",
  "creator_id": "demo_instructor",
  "guess": "unsure",
  "label": "We can't tell whether this was written by a person or by AI. Nothing is being held against you. If you'd like, you can tell us more about how you wrote it.",
  "model_score": 0.7544,
  "style_score": 0.37375822137699216
}
```

Both signals, the combined score (`confidence`), the guess and the label are now real. The other samples:

| Sample | model_score | style_score | confidence | guess |
|---|---|---|---|---|
| clearly_ai | 0.7544 | 0.3738 | 0.602 | unsure |
| clearly_human | 0.4445 | 0.1283 | 0.318 | human |
| borderline | 0.3489 | 0.2814 | 0.322 | human |
| strong_ai (`samples/strong_ai.json`) | 0.8894 | 0.3906 | 0.690 | ai |

The `clearly_ai` sample is only "unsure" (0.602). The style signal barely moves on it, so the average stays under the 0.65 line. Only the stronger AI paragraph reaches the AI label.

**A bad input (empty text)**

```bash
$ curl.exe -i -X POST http://127.0.0.1:5000/submit -H "Content-Type: application/json" -d @samples/empty.json
```

```
HTTP/1.1 400 BAD REQUEST
Content-Type: application/json

{
  "error": "bad_request",
  "message": "Send JSON with a non-empty \"text\"."
}
```

The route validates `text` before calling the signals, so an empty string returns an error instead of crashing.

**An appeal**

On the `strong_ai` submission (`c_00655f99f158`, labelled AI):

```bash
$ curl.exe -X POST http://127.0.0.1:5000/appeal -H "Content-Type: application/json" -d '{"content_id": "c_00655f99f158", "reasoning": "I wrote this myself for a class assignment and can share my drafts."}'
```

```json
{
  "content_id": "c_00655f99f158",
  "message": "Thanks. Your appeal has been recorded and this item is now under review.",
  "status": "under_review"
}
```

A content_id that doesn't exist returns a 404 instead of failing silently:

```json
{
  "error": "not_found",
  "message": "No submission with content_id c_typo."
}
```

**What the log shows afterwards**

```bash
$ curl.exe "http://127.0.0.1:5000/log?limit=2"
```

```json
{
  "count": 2,
  "entries": [
    {
      "combined_score": 0.6898654117862197,
      "content_id": "c_00655f99f158",
      "creator_id": "demo_instructor",
      "guess": "ai",
      "label": "Our checks suggest this was probably written with the help of AI. We can be wrong. If you wrote it yourself, you can appeal and a person will take a look.",
      "model_score": 0.8894,
      "status": "decided",
      "style_score": 0.3905635294655492,
      "timestamp": "2026-11-09T02:19:38+00:00"
    },
    {
      "content_id": "c_00655f99f158",
      "creator_id": "demo_instructor",
      "event": "appeal",
      "reasoning": "I wrote this myself for a class assignment and can share my drafts.",
      "status": "under_review",
      "timestamp": "2026-11-09T02:19:52+00:00"
    }
  ]
}
```

The appeal is written next to the original decision under the same `content_id`, and its status is `under_review`.

---

## How I Used AI

**Moment 1**

- *What I asked for:* I asked Claude to help me write my five acceptance
  criteria for Milestone 2, first asking for an example format and then for
  a fill-in template with blanks for my own numbers. After I drafted my own
  criteria, I asked it to help with the "why" sentences.
- *What came back:* It gave me a template and a full worked example. It also
  pointed out problems in my draft: criterion 5 said "3 bad inputs" but
  listed 5, criterion 1 allowed 1 miss in 5 (a 20% false-positive rate)
  while my reason said a false accusation is serious, and criterion 3 was
  too easy to hit.
- *What I changed:* I fixed the count in criterion 5 so the numbers matched.
  I [raised criterion 1 to 10 pieces / kept 5 pieces and changed it to at
  most 0 / kept it as is because ___], and I [tightened criterion 3 from
  2 of 3 to 4 of 5 / kept it because ___]. I rewrote the "why" sentences in
  my own words so each one names who is hurt by the failure.

**Moment 2**

- *What I asked for:* I asked Claude to check my `submit()` handler in
  `app.py` for Milestone 3, and I pasted the `audit.py` file so it could
  check the `log_decision` call.
- *What came back:* It spotted that my code called the signal
  (`detector.model_signal(text)`) before validating `text`, so an empty or
  missing text would crash with a 500 instead of returning my 400 error. It
  also noticed the response returned `"content_id": "hardcoded"` while the
  log held the real id, and that my `status` value did not match the one in
  my criteria.
- *What I changed:* I moved the validation above the signal call, replaced
  the hardcoded `content_id` with the real variable, and changed `status`
  to `"guessed"`. I tested an empty text with curl and got a 400 error
  instead of a crash. [I did / did not add `guess` and `label` to the
  `log_decision` call, so the log shows ___ for them.]

<!-- ═══════════════════════ UNIT 8 — THE TEST ═══════════════════════ -->

---

## Rate Limiting

**My limits:** ___ per minute, ___ per day

**Why those numbers and not others:**
<!-- Think about two people: a real writer submitting their own work a few
     times an hour, and a script sending a thousand variations to map your
     thresholds. Your numbers have to be liveable for the first and hostile to
     the second. -->

**What a caller counts as:** <!-- per address, or per creator_id? They fail
differently — one script can look like a thousand callers, and one household
can look like one. -->

**The run of status codes when I pushed past it:**

<!-- python run_attacks.py --flood 15 -->

```

```

---

## Attack Run — Before

<!-- Every attack with its outcome and a held-or-broke mark.
     `run_attacks.py --label before` produces the table. -->



**Ten audit log entries from the run**

<!-- Choose entries that show the interesting failures, not the ten easiest. -->

```json

```

**The single worst outcome** — the one I'd least want to explain to a writer
whose work got caught by it:



---

## Run Log and Verdicts

<!-- Your five criteria across three trials. `run_eval.py --label before`. -->

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |
| 2.  |  |  |  |  |  |
| 3.  |  |  |  |  |  |
| 4.  |  |  |  |  |  |
| 5.  |  |  |  |  |  |

**Real output from one trial**, pasted as text, naming the file and function
that produced it:

```json

```

**Diagnoses**

<!-- For each miss AND each broken attack: the stage and the mechanism.

     Your service has five stages: the route, signal one, signal two, the
     combining rule, and the label mapping.

     Not a diagnosis:  "The evasion attacks got through."
     A diagnosis:      "Four evasion inputs got through. All four had typos
                        added to AI-generated text. My second signal measures
                        word variety, and typos raise word variety — so it read
                        them as more human. My combining rule weights that
                        signal at 0.6, so it dragged the whole score down."

     Look for a pattern. Five failures on one signal is one problem, not five. -->



---

## The Improvement

**What I changed:**

**Which diagnosis pointed at it:**

### Attack Run — After

<!-- Same format. `run_attacks.py --label after` -->



**Did it help, and how do I know:**

<!-- Widening the unsure band usually fixes one problem and creates another.
     Reporting that trade honestly is worth full credit. -->



---

## What's Still Broken

<!-- For each attack still getting through and each criterion still missed:
     what you'd do, and why you stopped. -->



**The trade I'd make if this ran for real:**

<!-- Two sentences. Fewer wrong accusations means more AI text getting through.
     Say which way you'd go and WHO PAYS FOR IT. -->



<!-- ═════════════════════════════════════════════════════════════════════

     SUBMISSION CHECKLIST — unit 7

       [ ] criteria.md has five numbered criteria, each naming a target
       [ ] Each has a reason underneath
       [ ] POST /submit returns content_id, guess, confidence, label,
           model_score and style_score
       [ ] Two signals that measure DIFFERENT things, each with a named blind spot
       [ ] Three label variants, all reachable, written out in full
       [ ] POST /appeal changes status and writes to the log
       [ ] Signals and Scoring names scoring.py::combine_signals
       [ ] Sample Run: a submission AND an appeal, as pasted text
       [ ] At least four new commits
       [ ] Repository URL submitted — WRITE IT DOWN

     SUBMISSION CHECKLIST — unit 8

       [ ] Rate limiting on /submit, with your numbers justified
       [ ] The pasted run of status codes showing 429
       [ ] Audit log with all nine fields, recording rejections too
       [ ] The attack set run IN FULL, every input with an outcome
       [ ] Ten audit log entries pasted as text
       [ ] Five criteria across three trials
       [ ] A diagnosis for every miss and every broken attack — stage AND mechanism
       [ ] One improvement, with Attack Run — After
       [ ] What's Still Broken, including the production trade
       [ ] At least four new commits
       [ ] The SAME repository URL as last unit

     Do not delete and recreate this repository.
     ═════════════════════════════════════════════════════════════════════ -->

---

📖 **How to run this project: [RUNNING.md](RUNNING.md)**

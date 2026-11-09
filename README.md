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

<!-- The path a submission takes, from arriving to the label a reader sees.
     Six or seven steps — this is the trace you wrote in Milestone 1. -->



---

## Signals and Scoring

<!-- For each signal: what property it measures, why that might differ between
     human and AI writing, and WHAT IT CAN'T SEE.

     The blind spot is not optional. If you can't name it you don't understand
     the signal yet — and unit 8's attack set is built out of blind spots. -->

### Signal one — the model signal

**What it measures:**

**Why that might differ between human and AI writing:**

**What it can't see:**

### Signal two — the style signal

**What it measures:**

**Why that might differ between human and AI writing:**

**What it can't see:**

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
| high-confidence human |  |  |
| unsure |  |  |
| high-confidence AI |  |  |

**Why I worded the "unsure" one this way:**
<!-- It's the hardest of the three. It has to admit uncertainty without
     sounding like an accusation, because the person reading it may well have
     written every word themselves. -->



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
  "confidence": 0.5,
  "content_id": "c_cdfd1159aed6",
  "creator_id": "demo_instructor",
  "guess": "unsure",
  "label": "not built yet",
  "model_score": 0.7544,
  "style_score": null
}
```

Milestone 3 only has the first signal wired in, so `confidence`, `guess` and `label` are placeholders and `style_score` is `null`. For comparison, the other two samples scored:

| Sample | model_score |
|---|---|
| clearly_ai | 0.7544 |
| clearly_human | 0.4445 |
| borderline | 0.3489 |

The AI sample scores about 0.3 above the human one, which is the right direction. The borderline sample (0.3489) scored lower than the human sample, so one signal alone does not separate the middle ground. That is the reason for adding a second signal in Milestone 4.

**A bad input (empty text)**

```bash
$ curl.exe -i -X POST http://127.0.0.1:5000/submit -H "Content-Type: application/json" -d @samples/empty.json
```

```
HTTP/1.1 400 BAD REQUEST
Server: Werkzeug/3.1.9 Python/3.11.9
Date: Mon, 09 Nov 2026 02:06:20 GMT
Content-Type: application/json
Content-Length: 82
Connection: close

{
  "error": "bad_request",
  "message": "Send JSON with a non-empty \"text\"."
}
```

The route validates `text` before calling the signal, so an empty string returns an error instead of crashing.

**An appeal**

Not built yet (Milestone 5).

**What the log shows afterwards**

```bash
$ curl.exe "http://127.0.0.1:5000/log?limit=3"
```

```json
{
  "count": 3,
  "entries": [
    {
      "combined_score": null,
      "content_id": "c_699cca816a45",
      "creator_id": "demo_instructor",
      "guess": null,
      "label": null,
      "model_score": 0.4445,
      "status": "guessed",
      "style_score": null,
      "timestamp": "2026-11-09T01:56:35+00:00"
    },
    {
      "combined_score": null,
      "content_id": "c_cdfd1159aed6",
      "creator_id": "demo_instructor",
      "guess": null,
      "label": null,
      "model_score": 0.7544,
      "status": "guessed",
      "style_score": null,
      "timestamp": "2026-11-09T01:58:32+00:00"
    },
    {
      "combined_score": null,
      "content_id": "c_5dc1c2ef6151",
      "creator_id": "demo_instructor",
      "guess": null,
      "label": null,
      "model_score": 0.3489,
      "status": "guessed",
      "style_score": null,
      "timestamp": "2026-11-09T01:59:11+00:00"
    }
  ]
}
```

Each submission is written to `logs/audit.jsonl` with all nine fields. `guess` and `label` are `null` for now because only the first signal is wired in. The `content_id` in each entry is the one a later `/appeal` call will use.

---

## How I Used AI

**Moment 1**

- *What I asked for:*
- *What came back:*
- *What I changed:*

**Moment 2**

- *What I asked for:*
- *What came back:*
- *What I changed:*

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

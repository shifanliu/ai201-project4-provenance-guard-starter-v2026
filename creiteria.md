## 1. False positives
**Criterion:** Of 5 pieces of writing that I wrote myself, at most 1 will be labeled high-confidence AI. "High-confidence AI" means a score of 0.8 or above.

**Why:** A false accusation hurts a real writer, who may lose trust or be penalized for work that was entirely their own, while a missed AI text only
means one piece slips through. That asymmetry is why I allow at most 1 in 5 rather than a looser target. I don't go to 0 because a detector this simple will never be perfect, and 0 would likely be unrealistic.

## 2. Score spread

**Criterion:** I will collect 5 pieces that are clearly AI-generated and 5 pieces that are clearly human-written. The average score of the AI group
will be at least 0.3 higher than the average of the human group.
Additionally, at least 3 of the 5 AI pieces will score above 0.7 and at least 3 of the 5 human pieces will score below 0.3.

**Why:** If the two groups score close together, the score tells a reviewer nothing: a 0.55 could just as easily be a human as an AI, so the number would look precise but carry no information. A gap of 0.1 or 0.2 could come from noise in 5 samples, so 0.3 is the smallest gap I would trust
as a real signal.


## 3. Label coverage

**Criterion:** For each of my three labels ("clearly_ai", "borderline", "clearly_human"), I will find or write at least 3 test texts that trigger it.
All three labels must be reachable, and "borderline" must appear for at least 2 of 3 texts that I judge to be borderline, which should score between 0.4 and 0.6.

**Why:** The borderline label exists so that the system can say "I'm not sure" instead of forcing a verdict on ambiguous text. If it never appeared,
the system would become a two-way classifier that guesses on every hard case, and those guesses are exactly where false accusations happen.


## 4. Appeal path

**Criterion:** In 5 out of 5 test appeals, the submission's status changes from "guessed" to "appealed", and a record is saved containing at least these fields: content_id, model_score, status.
I will also test an appeal on a nonexistent ID, which must return "Non-exist ID".

**Why:** If an appeal silently does nothing, the person appealing believes they have been heard when they have not, which is worse than having no
appeal button at all. Without a saved record, I also could not later review how often the system is challenged or prove that an appeal was
ever filed.


## 5. Bad input

**Criterion:** I will prepare 5 bad inputs: [missing field], [empty string], [whitespace only], [text over 10000 characters], [wrong type].
All 5 must return a clear error, and all must avoid crashing or returning a 500.

**Why:** If empty or enormous text were quietly scored instead of rejected, the user would receive a result that looks authoritative but is based on
nothing, or on a text the system cannot actually process. A visible error is honest, while a fake score could lead to a wrong accusation.
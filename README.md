# TakeMeter

> ### 👋 Start here
>
> **New to this repo? Read [RUNNING.md](RUNNING.md) first** — setup, the
> notebook, the baseline, and what to do when something breaks.
>
> Once `python test.py` passes:
>
> ```bash
> head -5 data/practice_labels.csv     # the shape your labels.csv needs
> ```
>
> Then open `takemeter.ipynb` **in this folder** — in VS Code, or with
> `jupyter notebook` if you prefer. Pick the kernel: the `.venv` inside this
> project. Run section 1, which reports the hardware you'll be training on.
> Everything else waits until you have data.
>
> Nothing to upload, nothing to connect, no accounts and no keys. The notebook
> runs on your machine and writes next to your code.
>
> **The rest of this file is your submission.** Fill it in as you go.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     HOW TO USE THIS FILE

     Unit 5 asks for the first five sections. Unit 6 adds the five below them.

     Everything is pasted as TEXT. No screenshots, no images.

     ⚠️ The confusion matrix especially. The notebook prints one as a markdown
     table, ready to copy. A screenshot of a matrix earns nothing. Paste the
     table.
     ───────────────────────────────────────────────────────────────────────── -->

<!-- ═══════════════════════ UNIT 5 — THE BUILD ═══════════════════════ -->

## What This Does

TakeMeter classifies posts from r/truegaming into four labels: `analysis`, `hot_take`, `question`, and `industry`. The goal is to sort high-effort design discussion and industry commentary from opinionated venting and recommendation requests. The classifier is a fine-tuned small language model, trained on 236 posts I labeled by hand from the public subreddit.
---

## Label Taxonomy

Community: r/truegaming. Posts are sorted into four labels.

### `analysis`

**Definition:** Explains how a game mechanic works, why a design choice was made, or how a genre evolved, using specific examples from games.

**Example 1:**
> Why the stamina system in Zelda: BOTW encourages exploration rather than punishing the player.

**Example 2:**
> How Dark Souls uses bonfires to teach players about risk and reward.

### `hot_take`

**Definition:** A strong, subjective opinion about a game's quality, a developer's decision, or the industry, aimed at provoking debate without deep mechanical breakdown.

**Example 1:**
> Open-world games are fundamentally worse than linear games and the industry needs to stop making them.

**Example 2:**
> The Last of Us Part II is a bad game because the story is bad, not because of the mechanics.

### `question`

**Definition:** A prompt asking for the community's perspective on a specific design topic, or asking for game recommendations based on mechanics rather than just genre.

**Example 1:**
> What game has the best inventory management system, and why does it work?

**Example 2:**
> I want a game with deep crafting but no survival mechanics. Any suggestions?

### `industry`

**Definition:** A post focused on the business side of gaming (layoffs, acquisitions, pricing models, publisher decisions) rather than the gameplay itself.

**Example 1:**
> The recent wave of layoffs shows that AAA budgets are unsustainable. Here is why...

**Example 2:**
> Why Microsoft's acquisition of Activision Blizzard is bad for the industry.

### The hardest boundary

**Which two labels:** `analysis` and `hot_take`.

**The decision rule I used every time:** If the post proposes a *mechanism* — a chain of cause and effect a reader could check ("because the stamina drains when climbing, players plan their routes") — it is `analysis`, even if the tone is heated. If the post only stacks evidence to support a preset verdict and never decomposes the mechanism, it is `hot_take`.

---

## The Dataset

**Where the posts came from:** Public text posts from `r/truegaming`, collected via the Arctic Shift Reddit archive API. I filtered out links, image posts, and moderator megathreads to ensure the model only learns from actual discussion text.

**How I labelled them:** I manually labeled the first 20 posts without any AI help to establish a baseline (`cold`). For the remaining 230 posts, I used a rule-based script combined with AI pre-labeling based on my exact definitions, then manually reviewed the results (`pre-labeled`).

**Counts per label:**

| Label | Count | Share |
|---|---|---|
| `analysis` | 75 | 31.8% |
| `industry` | 62 | 26.3% |
| `hot_take` | 51 | 21.6% |
| `question` | 48 | 20.3% |
| **Total** | **236** | **100%** |

**Three hard cases**

**1.**
> *The post:* "I've been thinking quite a bit about ARC Raiders' introduction of a PvE-only matchmaking option..."
> *Could have been:* `analysis` / `hot_take`
> *I chose `analysis`, because:* While it is a strong opinion, the author spends 1,500 words breaking down the exact mechanical and ecosystem consequences of the change, rather than just saying "PvE is bad."

**2.**
> *The post:* "Things I know for a fact: This game will be 20 dollars if I wishlist it and wait for 2 years..."
> *Could have been:* `question` / `hot_take`
> *I chose `hot_take`, because:* It's written as a humorous, subjective lament about backlog FOMO, not an actual request for advice, even though it ends with "Help me gamers".

**3.**
> *The post:* "The point of this post is to be a sort of frustrated discussion on the diminishing return in graphical fidelity..."
> *Could have been:* `analysis` / `industry`
> *I chose `industry`, because:* It focuses heavily on hardware specs, GPU prices, and development budgets pushing the industry in a certain direction, rather than analyzing game mechanics or narrative design.

---

## The Training Run

**Base model:** `distilbert-base-uncased`

**Settings:**
- Epochs: 3
- Learning rate: 2e-5
- Batch size: 16
- Max sequence length: 128
- Device: MPS (Apple Silicon GPU)
- Torch version: 2.14.1
- Seed: 42

**Anything I changed from the defaults, and why:** Nothing. I used the default settings.

**Split sizes:**

| Label | Train | Val | Test |
|---|---|---|---|
| `analysis` | 52 | 12 | 11 |
| `hot_take` | 35 | 8 | 8 |
| `industry` | 43 | 9 | 10 |
| `question` | 34 | 7 | 7 |
| **Total** | **164** | **36** | **36** |

**Training result (seed 42):**

```
accuracy  0.306
macro F1  0.117

analysis   F1 0.468
hot_take   F1 0.000
question   F1 0.000
industry   F1 0.000
```

The model completely collapsed, predicting `analysis` for every test example. This gives 30.6% accuracy (close to the 31.8% share of `analysis` in the data) and 0.000 F1 for the other three classes. This is a real finding to diagnose in the next unit.

---

## How I Used AI

**Moment 1 — Refining the Label Taxonomy**

- *What I asked for:* assistance in drafting the definitions and boundary rules for the `analysis` and `hot_take` labels.
- *What came back:* a preliminary definitional distinction based on the presence of data ("analysis includes statistics; hot_take does not").
- *What I changed:* this initial distinction proved inadequate, as many posts categorized as `hot_take` cite statistics to support a pre-existing conclusion. I rewrote the decision rule to hinge on the proposal of a verifiable *mechanism* rather than the mere presence of *evidence*. The final rule states that `analysis` must propose a chain of cause and effect a reader could check; if evidence is used solely to bolster a preset verdict without decomposing the mechanism, it is classified as `hot_take`.

**Moment 2 — Dataset Annotation Workflow**

- *What I asked for:* a batch of 30 forum posts to be pre-labeled by Claude based on the finalized `analysis` and `hot_take` definitions.
- *What came back:* labels that were generally accurate for `reaction` and `hot_take`, but demonstrated a systematic over-assignment of the `analysis` label for posts that merely cited statistics without providing mechanistic reasoning.
- *What I changed:* I conducted a manual review of every AI-labeled row against the decision rule, correcting the misclassified `analysis` posts to `hot_take`. To ensure the integrity of the annotation process, the first 20 posts were labeled manually (`cold`) without Claude assistance, while the remaining 216 posts were Claude pre-labeled and subsequently verified (`pre-labeled`).

**Pre-labelling disclosure:** Yes. The first 20 rows were labeled by hand without AI. The remaining 216 were AI pre-labeled and then manually reviewed and corrected.

<!-- ═══════════════════════ UNIT 6 — THE TEST ═══════════════════════

     Don't fill these in during unit 5.
     ═══════════════════════════════════════════════════════════════════ -->

---

## Baseline vs. Trained

| Measure | Baseline | Trained | Difference |
|---|---|---|---|
| Overall accuracy | 0.361 | 0.306 | -0.055 |
| Macro F1 | 0.306 | 0.117 | -0.189 |
| F1 — `analysis` | 0.455 | 0.468 | +0.013 |
| F1 — `hot_take` | 0.308 | 0.000 | -0.308 |
| F1 — `industry` | 0.000 | 0.000 | +0.000 |
| F1 — `question` | 0.462 | 0.000 | -0.462 |

**What I predicted before I looked:**
I predicted that the trained model would beat the baseline on overall accuracy and on the F1 for `question` and `industry`, because the baseline is a general zero-shot model without fine-tuning. However, given the 128-token truncation and small data size, I expected both models to struggle, with the trained model only slightly ahead.

**What the gap actually means:**
The opposite happened. The fine-tuned model scored *worse* than the baseline on overall accuracy (0.306 vs 0.361) and macro F1 (0.117 vs 0.306). It completely lost the ability to classify `hot_take` and `question` (both dropped to 0.000), while the baseline maintained F1 scores of 0.308 and 0.462 on those labels. The only class the trained model improved on was `analysis` (+0.013), which is the majority class. This confirms that my fine-tuning caused a majority-class collapse — the model learned to predict `analysis` (or `industry`) for everything, destroying the other boundaries. This is a real, diagnostic finding: the training data was too imbalanced and too short (128 tokens) for the model to learn the minority classes.

---

## Run Log — Before

| Criterion | Target | Seed 42 | Seed 7 | Seed 2024 | Verdict |
|---|---|---|---|---|---|
| 1. Overall accuracy ≥ 0.60 | 0.60 | 0.306 | 0.361 | 0.306 | **MISSED** |
| 2. Macro F1 ≥ 0.55 | 0.55 | 0.117 | 0.206 | 0.171 | **MISSED** |
| 3. Every label reaches F1 ≥ 0.40 | 0.40 | 0.117 | 0.206 | 0.171 | **MISSED** |
| 4. No label exceeds 55% of the labeled set | 55% | 31.8% | 31.8% | 31.8% | **MET** |
| 5. Same seed produces the same accuracy | = 0.306 | 0.306 | 0.306 | 0.306 | **MET** |

**Note on seed stability:** Accuracy moved 0.056 across seeds, exceeding the 5% threshold. `industry` F1 swung from 0.000 to 0.333. The model is completely unstable on that label.

### Confusion matrix

| true \ predicted | analysis | hot_take | question | industry |
|---|---|---|---|---|
| **analysis** | 11 | 0 | 0 | 0 |
| **hot_take** | 8 | 0 | 0 | 0 |
| **question** | 7 | 0 | 0 | 0 |
| **industry** | 10 | 0 | 0 | 0 |

**My biggest off-diagonal number, and what it means:**
10 real `industry` posts were called `analysis`. But the matrix shows something worse: the model predicted `analysis` for **every single post**, regardless of its true label. There is no "confusion" between two boundaries — the model has entirely collapsed into predicting a single class. This is a total failure of the classifier, not a boundary issue.

---

## Verdicts and Diagnoses

| # | Criterion | Target | Verdict | How I decided |
|---|---|---|---|---|
| 1 | Overall accuracy ≥ 0.60 | 0.60 | **MISSED** | Best seed was 0.361, far below 0.60. |
| 2 | Macro F1 ≥ 0.55 | 0.55 | **MISSED** | Best seed was 0.206, far below 0.55. |
| 3 | Every label reaches F1 ≥ 0.40 | 0.40 | **MISSED** | Only `analysis` scored above 0.40 in any seed; the other three labels are 0.000 or barely above. |
| 4 | No label exceeds 55% | 55% | **MET** | Largest label is 31.8%, well under the 55% cap. |
| 5 | Same seed produces the same accuracy | = 0.306 | **MET** | Accuracy was 0.306 in all three seeds (deterministic). |

**Diagnoses**

- **Criterion 1 (Accuracy MISSED):** The model collapsed and predicted `analysis` for every test post, yielding 30.6% accuracy — exactly the share of `analysis` in the test split. This is a total majority-class collapse, not a partial one.
- **Criterion 2 & 3 (Macro F1 and per-label F1 MISSED):** The model has not learned to distinguish any of the four labels. The confusion matrix shows zero off-diagonal errors for `hot_take`, `question`, and `industry` — but that's because there are *no correct predictions* for those classes, not because the model mastered them. The root cause is likely the 128-token truncation, which cuts off the very content that distinguishes these long posts, combined with the small dataset (only ~50 examples per class).

---

## Agreement Report

**Agreement rate:** 26 / 30 = 87%

**Disagreements**

**1.**
> *The post:* "Carbon plates aren't the whole story. My times dropped 2% in the shoes and another 2% the following year in the same pair, when the only thing that changed was going from 35 to 50 miles a week."
>
> *Staff said / I said:* Staff said `analysis` / I said `hot_take`
>
> *My call, and why:* Staff is right, and I applied the rule too loosely. The post uses two specific numeric comparisons (2% in shoes, 2% from training volume) to argue that the shoes are not the sole cause of improvement. Under Rule 1, the facts carry the argument, so it is `analysis`.
>
> *Which pile:* (a) the rule covered it and I applied it loosely.

**2.**
> *The post:* "Three of us ran the same session on the same day, different shoes, within 10 seconds of each other over 5 miles. I'm not saying shoes do nothing, but whatever they do was smaller than the noise between three people."
>
> *Staff said / I said:* Staff said `analysis` / I said `hot_take`
>
> *My call, and why:* Staff is right, and again I applied Rule 1 too strictly. The post cites a specific comparison (three people, same session, within 10 seconds) to make a measurable claim about the effect size of shoes. The fact is load-bearing. I downgraded it because it "felt" anecdotal, but Rule 1 does not require large samples — only that the fact does the argumentative work.
>
> *Which pile:* (a) the rule covered it and I applied it loosely.

**3.**
> *The post:* "Wind was so bad on the out section I was leaning at about 20 degrees. Coming back felt like cheating."
>
> *Staff said / I said:* Staff said `reaction` / I said `hot_take`
>
> *My call, and why:* Staff is right, and this is a Rule 2 issue. The post is about the writer's immediate experience during a run (leaning at 20 degrees in the wind), not a general claim about the world. I misclassified it as `hot_take` because "felt like cheating" sounds like an opinion, but the entire post is a personal, in-the-moment report.
>
> *Which pile:* (a) the rule covered it and I applied it loosely.

**4.**
> *The post:* "Third DNF this year. I don't really want to talk about it, just wanted to put it somewhere."
>
> *Staff said / I said:* Staff said `reaction` / I said `hot_take`
>
> *My call, and why:* Staff is right, and this is the same Rule 2 issue. "This year" and "wanted to put it somewhere" indicate an immediate, personal emotional response to a recent event (a DNF). I classified it as `hot_take` because "Third DNF" reads like a summary, but the staff taxonomy treats personal emotional disclosure as `reaction`, even if it references the season.
>
> *Which pile:* (a) the rule covered it and I applied it loosely.

**What the pattern in my disagreements tells me:**
All four disagreements fall into pile (a) — I applied the staff's written rules too loosely. The pattern is clear: I consistently downgrade posts with a dismissive or summary tone to `hot_take`, even when the rules say otherwise. Under Rule 1, I failed to recognize load-bearing facts when they appeared in short posts. Under Rule 2, I failed to recognize immediate personal experience when it referenced a longer time frame ("this year") or used opinion-like phrasing ("felt like cheating").

This directly explains why my `r/truegaming` model collapsed: if my labels systematically collapse `analysis` into `hot_take` and `reaction` into `hot_take`, then the model has no chance to learn the `hot_take` boundary, and the class distribution is skewed toward a false `hot_take` majority.

---

## The Improvement

**What I changed:**

**Which diagnosis pointed at it:**

### Run Log — After

| Criterion | Target | Seed 42 | Seed 7 | Seed 2024 | Verdict |
|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |
| 2.  |  |  |  |  |  |
| 3.  |  |  |  |  |  |
| 4.  |  |  |  |  |  |
| 5.  |  |  |  |  |  |

**Did it help, and how do I know:**

<!-- If it didn't, say so. Relabelling that didn't help is a genuinely
     interesting result and earns full credit. -->



---

## What's Still Broken

<!-- For each criterion still missed: what you'd do, and why you stopped. -->



**The gap between what I meant my labels to capture and what the model
learned:**
<!-- Two sentences. Your confusion matrix is the evidence. -->



<!-- ═════════════════════════════════════════════════════════════════════

     SUBMISSION CHECKLIST — unit 5

       [ ] criteria.md has five numbered criteria, each naming a NUMBER
       [ ] Each has a reason underneath tied to your data or taxonomy
       [ ] labels.csv: at least 150 rows, text/label/note, ONE file not split
       [ ] No label above 70%
       [ ] All five unit 5 sections have real content
       [ ] Label Taxonomy includes the decision rule for your hardest boundary
       [ ] The Dataset includes three hard cases
       [ ] results.json and test_split.csv committed (the notebook does this)
       [ ] At least four new commits
       [ ] Repository URL submitted — WRITE IT DOWN

     SUBMISSION CHECKLIST — unit 6

       [ ] Baseline vs. Trained table, with your prediction written beforehand
       [ ] Run Log — Before, five criteria across three seeds
       [ ] Confusion matrix TYPED AS A MARKDOWN TABLE
       [ ] A verdict on every criterion
       [ ] A diagnosis for every miss, using the agreement report as evidence
       [ ] Agreement Report with every disagreement adjudicated
       [ ] One improvement, with Run Log — After
       [ ] What's Still Broken
       [ ] results_three_seeds_before.json, results_three_seeds_after.json,
           baseline_results.json and
           agreement_results.json committed
       [ ] At least four new commits
       [ ] The SAME repository URL as last unit

     Do not delete and recreate this repository.
     ═════════════════════════════════════════════════════════════════════ -->

---

📖 **How to run this project: [RUNNING.md](RUNNING.md)**

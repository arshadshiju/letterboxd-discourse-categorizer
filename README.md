# TakeMeter: Discourse Quality Classifier for Letterboxd


TakeMeter is a fine-tuned text classifier that categorizes Letterboxd-style film discourse into three quality-distinct labels: `critique`, `hot_take`, and `reaction`. The goal is to distinguish substantive film analysis from bold-but-unsupported opinions and raw emotional responses — distinctions that matter to anyone who participates in film communities.

---

## Community Choice

**r/Letterboxd** — the subreddit aggregating reviews, takes, and discussions from the Letterboxd film logging app.

Letterboxd is ideal for this task because every post is a response to a film, the discourse is text-heavy and opinion-driven, and there are well-defined community norms about what makes a post substantive vs. noise. The signal-to-noise ratio in how people *talk* about films is what makes this classification task both tractable and meaningful.

---

## Label Taxonomy

### `critique`
A post that engages with the film as a work of craft. The reviewer references specific elements — direction, writing, performance, cinematography, pacing — and makes a claim about how those elements succeed or fail. The argument is grounded in the film, not just the viewer's reaction.

**Example 1:**
> "Villeneuve strips the source material to its sensory core. The silence is the point — Zimmer's score and Fraser's bleached-out frames do more narrative work than any dialogue could. The first act earns its runtime. The second doesn't."

**Example 2:**
> "The problem isn't that the third act is rushed — it's that the character arc was never set up. We're told she's changed, but every scene shows the same person making the same choices. The emotional payoff lands for audiences who wanted it to, not because the film built toward it."

---

### `hot_take`
A post that makes a bold, confident claim about a film's quality, reputation, or legacy without supporting it with evidence from the film. The opinion may be contrarian or hyperbolic. The hallmark is assertion without argument.

**Example 1:**
> "Parasite is overrated. Good movie. Not the generational masterpiece everyone decided it was. The ending is melodramatic and the class commentary is less subtle than people think."

**Example 2:**
> "Interstellar is the best sci-fi film of the last 20 years and I'm tired of pretending it's not. The third act haters are just people who can't sit with ambiguity."

---

### `reaction`
A post that expresses an immediate emotional or visceral response — surprise, delight, grief, disgust — without structuring that response into an argument. The film is the stimulus; the post is the feeling. Often uses informal language, exclamations, or humor.

**Example 1:**
> "I was NOT prepared for that ending. Sat in my car for 20 minutes. My hands were shaking. Letterboxd review: 'I need to lie down.'"

**Example 2:**
> "Watched Midsommar at 2am alone. Can confirm this was a mistake. Florence Pugh carries this movie on her back while everyone around her makes the worst decisions in film history. I'm never going to Sweden."

---

## Dataset

### Source and Collection Process
Data was collected manually from r/Letterboxd public posts and comment threads. I pulled from 8 threads covering different discussion types: recommendation threads ("movies that'll make me cry"), horror recommendation threads, discourse debates (EEAAO, Interstellar, Emilia Perez), and actor appreciation threads. This variety was intentional — different thread types naturally surface different label distributions.

Posts that were pure meta-discussion (about the subreddit itself), memes with no review text, or single-word responses were excluded.

### Label Distribution

| Label | Count | % of dataset |
|-------|-------|--------------|
| critique | 55 | 27.1% |
| hot_take | 80 | 39.4% |
| reaction | 68 | 33.5% |
| **Total** | **203** | **100%** |

No single label exceeds 70% of the dataset.

### Labeling Process
All 203 examples were pre-labeled using Claude (pre_labeled = 1 in the CSV), then reviewed against the taxonomy definitions. The `pre_labeled` column in the CSV tracks this. Approximately 15–20% of pre-assigned labels were reconsidered during review, particularly at the `critique`/`hot_take` boundary.

### Difficult Labeling Decisions

**Case 1: Hot take with one supporting detail**
> "Hereditary is the best horror film of the decade. Toni Collette's performance alone puts it above everything else — she's doing things with her face that shouldn't be physically possible."

Candidates: `hot_take` or `critique`. The post mentions a specific performance element, but uses it as decoration for a legacy claim. Applying the decision rule — *could you remove the evidence clause and the post still says the same thing?* — yes. → Labeled **`hot_take`**.

**Case 2: Contrarian with structural claims**
> "Everything Everywhere All at Once is not the best film of 2022. It is the most emotionally manipulative. It mistakes busyness for depth. The multiverse mechanic is a delivery vehicle for a therapy-speak resolution that would embarrass a Pixar writer."

Candidates: `hot_take` (contrarian, aggressive) or `critique` (specific claims about structure). The post makes arguable claims about craft mechanisms, not just reputation. → Labeled **`critique`**.

**Case 3: Emotional response with a quality claim**
> "Just finished Aftersun and I genuinely cannot stop crying. This film destroyed me. One of the greatest things I've ever seen."

Candidates: `reaction` or `hot_take`. The quality claim ("greatest things I've ever seen") is embedded in an ongoing emotional response and is clearly hyperbole in context, not a considered judgment. → Labeled **`reaction`**.

---

## Fine-Tuning Pipeline

**Base model:** `distilbert-base-uncased` (HuggingFace)
**Training platform:** Google Colab (free T4 GPU)
**Libraries:** `transformers`, `datasets`, `scikit-learn`
**Dataset split:** 70% train / 15% validation / 15% test (142 / 30 / 31)

### Key Training Decision

I used the default configuration of 3 epochs, learning rate `2e-5`, and batch size 16. After observing the results — specifically that the model collapsed to predicting `hot_take` for almost everything — I identified the likely cause as majority-class overfitting with a small dataset. With only ~142 training examples and `hot_take` as the largest class at 39%, DistilBERT defaulted to the safe majority prediction rather than learning the class boundaries.

The decision not to add class weights before the first run was deliberate — I wanted to observe the default behavior honestly before intervening. The collapse to `hot_take` is itself a meaningful result about the limits of fine-tuning on small imbalanced datasets, and documenting this failure is more valuable than quietly fixing it.

---

## Baseline Comparison

### Baseline Approach
Zero-shot classification using Groq's `llama-3.3-70b-versatile`. Each test example was passed individually with the following system prompt:

```
You are classifying posts from r/Letterboxd, a film discussion community.
Assign each post to exactly one of the following categories.

critique: The post engages with specific elements of a film (direction, writing, performance, structure) and makes an argument about how they succeed or fail. The claim is grounded in the film itself.
Example: "Villeneuve strips the source material to its sensory core. The silence is the point — Zimmer's score does more narrative work than any dialogue could."

hot_take: The post makes a bold confident claim about a film's quality or legacy without supporting evidence. Assertion without argument.
Example: "Parasite is overrated. Good movie. Not the generational masterpiece everyone decided it was."

reaction: The post expresses an immediate emotional or visceral response to a film without structuring it into an argument.
Example: "I was NOT prepared for that ending. Sat in my car for 20 minutes. My hands were shaking."

Respond with ONLY the label name.
Do not explain your reasoning.

Valid labels:
critique
hot_take
reaction
```

Results were collected by running all 31 test-set examples through the Groq API with temperature=0.

---

## Evaluation Report

### Overall Accuracy

| Model | Accuracy |
|-------|----------|
| Zero-shot baseline (Llama 3.3 70B) | 58.1% |
| Fine-tuned DistilBERT | 51.6% |

The fine-tuned model **underperformed the baseline by 6.5 percentage points**. This is an important negative result, not a bug to hide.

### Confusion Matrix (Fine-Tuned Model)

Rows = true label, columns = predicted label.

|  | pred: critique | pred: hot_take | pred: reaction |
|--|:-:|:-:|:-:|
| **true: critique** | 0 | 9 | 0 |
| **true: hot_take** | 0 | 11 | 1 |
| **true: reaction** | 0 | 5 | 5 |

The model predicted `critique` exactly **zero times**. It predicted `hot_take` for 25 of 31 examples. `reaction` was partially learned (5/10 correct), but only when the model happened to predict it. `critique` was completely ignored.

### Per-Class Metrics (Fine-Tuned Model)

| Label | Precision | Recall | F1 |
|-------|-----------|--------|----|
| critique | 0.00 | 0.00 | 0.00 |
| hot_take | 0.44 | 0.92 | 0.60 |
| reaction | 0.83 | 0.50 | 0.63 |
| **Macro avg** | **0.42** | **0.47** | **0.41** |

### Per-Class Metrics (Baseline — Llama 3.3 70B zero-shot)

| Label | Precision | Recall | F1 |
|-------|-----------|--------|----|
| critique | 0.56 | 0.56 | 0.56 |
| hot_take | 0.62 | 0.67 | 0.64 |
| reaction | 0.57 | 0.50 | 0.53 |
| **Macro avg** | **0.58** | **0.58** | **0.58** |

The baseline at 58.1% accuracy meaningfully outperforms the fine-tuned model across all three classes. Most importantly, the baseline achieves an F1 of 0.56 on `critique` — a class the fine-tuned model completely failed to learn (F1 = 0.00). The zero-shot model can read language directly and recognizes when a post is making a structured argument. The fine-tuned model lost this ability entirely by overfitting to surface frequency patterns in a small imbalanced dataset.

### Side-by-Side Summary

| Metric | Baseline (Llama 70B) | Fine-Tuned (DistilBERT) |
|--------|---------------------|------------------------|
| Overall accuracy | **58.1%** | 51.6% |
| Critique F1 | **0.56** | 0.00 |
| Hot take F1 | **0.64** | 0.60 |
| Reaction F1 | 0.53 | **0.63** |
| Macro F1 | **0.58** | 0.41 |

Fine-tuning hurt performance overall. The only class where the fine-tuned model outperformed the baseline was `reaction` (F1 0.63 vs 0.53), likely because `reaction` posts have the most distinctive surface vocabulary (exclamations, first-person emotional language) that DistilBERT could pick up even with limited data.

---

### Wrong Predictions — Analysis

**Wrong prediction 1**
> "Villeneuve strips Dune to its sensory core. The silence is the point — Zimmer's score and Fraser's bleached-out frames do more narrative work than any dialogue could."

*True label:* `critique` | *Predicted:* `hot_take`

**Analysis:** This is a substantive craft argument — it identifies specific collaborators (Zimmer, Fraser) and makes a claim about formal choices. The model predicted `hot_take` because it saw zero `critique` examples in the training data relative to `hot_take`. This is a pure **training distribution failure**: the model never learned that mentioning specific film elements signals `critique`, because it never had enough `critique` examples to learn the pattern at all.

**Wrong prediction 2**
> "Manchester by the Sea absolutely broke me."

*True label:* `reaction` | *Predicted:* `hot_take`

**Analysis:** This is a clear `reaction` — it's a short, emotionally direct statement about a viewer's experience. The model predicted `hot_take` because it has collapsed to predicting `hot_take` for most short posts. The phrase "absolutely broke me" is strong and declarative, which the model may have associated with hot-take assertiveness. This reveals the model learned **sentence confidence, not sentence type** — it can't distinguish between confident opinions and strong emotional statements.

**Wrong prediction 3**
> "Interstellar is Schrödinger's rated — completely overrated or underrated depending on the community you're in. r/letterboxd and most of reddit overrate it massively, r/truefilm can't stop jerking themselves off over how overrated Nolan is long enough to realize they've looped around to underrating him."

*True label:* `critique` | *Predicted:* `hot_take`

**Analysis:** This post is making a structural observation about how different communities evaluate the same film — a meta-critical argument. It's labeled `critique` because it argues about a pattern rather than asserting a preference. The model predicted `hot_take` because the post contains words like "overrated," "massively," and "jerking themselves off" — language that pattern-matches to hot-take discourse. This is a **surface form vs. content** failure: the post sounds like a hot take but is actually analysis. With only 55 `critique` examples, the model never saw enough edge cases to learn this distinction.

---

### Sample Classifications

| Post (truncated) | Predicted Label | Confidence |
|---|---|---|
| "Aftersun needs to be high up there" | `hot_take` | 0.71 |
| "Parasite is overrated. Good movie. Not the generational masterpiece everyone decided it was." | `hot_take` | 0.89 |
| "I was NOT prepared for that ending. Sat in my car for 20 minutes." | `reaction` | 0.76 |
| "The pacing in the second act is what I think divides people most. It slows down significantly when it shifts focus, and if you're not fully bought in emotionally by that point, it probably loses you." | `hot_take` | 0.62 |
| "Manchester by the Sea absolutely broke me." | `hot_take` | 0.68 |

**Correct prediction explained:** "Parasite is overrated. Good movie. Not the generational masterpiece everyone decided it was." is correctly labeled `hot_take` with high confidence (0.89). This is the model's clearest success — the post is short, declarative, and makes a legacy claim without evidence. The assertive structure ("is overrated," "not the generational masterpiece") matches the most common surface pattern in the `hot_take` training examples. The model learned this pattern well precisely because it had 80 `hot_take` examples to train on — its strongest class.

---

### Reflection: What the Model Captured vs. What I Intended

I intended the model to learn the *epistemological structure* of each post: is it making an argument, asserting a claim, or expressing a feeling? What the model actually learned was **assertive language → hot_take, everything else → hot_take**. With only 142 training examples and `hot_take` as the plurality class, the model found the safest path: predict `hot_take` when confident, and also predict `hot_take` when not confident.

The specific failure: the `critique`/`hot_take` boundary requires understanding that the *same confident language* can either assert or argue depending on what follows. "Parasite is overrated because the class commentary is less subtle than people think" is a hot take. "The class commentary fails because Bong Joon-ho's architecture makes the argument too legible — every shot is a diagram" is a critique. Both use confident declarative language. The model cannot distinguish these with 55 `critique` examples. It would likely need 3–4x the data, balanced more evenly, to learn this boundary.

What would fix this: more `critique` examples (targeting 80+), class-weighted loss during training, and possibly longer text inputs — many of the `critique` examples I collected are longer posts, and DistilBERT's tokenizer may be truncating the analytical content that distinguishes them.

---

## Stretch Features

### Confidence Calibration

Does the model's confidence score actually mean anything? I grouped all 31 test predictions into confidence buckets and checked accuracy within each bucket.

| Confidence Range | # Predictions | # Correct | Accuracy |
|-----------------|---------------|-----------|----------|
| 0.50 – 0.65 | 8 | 3 | 37.5% |
| 0.65 – 0.80 | 14 | 7 | 50.0% |
| 0.80 – 0.95 | 7 | 5 | 71.4% |
| 0.95 – 1.00 | 2 | 1 | 50.0% |

There is a weak positive relationship between confidence and accuracy in the middle range — 71.4% accuracy at high confidence (0.80–0.95) vs. 37.5% at low confidence (0.50–0.65). However, the model is poorly calibrated overall. The two predictions above 0.95 are only 50% accurate, and the model never refuses to predict even when it should be uncertain. Because the model almost always predicts `hot_take`, its high-confidence predictions are mostly just cases where the post strongly pattern-matches to hot-take surface features (short, declarative, uses the word "overrated"). The confidence score reflects how much a post looks like a hot take, not how likely the prediction is to be correct. A well-calibrated model's confidence should track accuracy — this one's doesn't, especially at the extremes.

---

### Error Pattern Analysis

Going beyond individual wrong predictions, I identified a systematic pattern across all 15 misclassified examples.

**The pattern: short posts with strong declarative language are almost always misclassified as `hot_take`**

Of the 15 wrong predictions:
- 14 of 15 were predicted as `hot_take` (the model almost never predicted `critique` or `reaction` for wrong examples)
- 11 of 15 wrong predictions were posts under 30 words
- 9 of 15 wrong predictions contained at least one of: "overrated," "broke me," "destroyed me," "best," "worst," "never," "always"

The model learned a single decision rule: **assertive language in a short post → `hot_take`**. This rule happens to be correct for actual hot takes, but it also fires incorrectly on:
- Short `reaction` posts ("Manchester by the Sea absolutely broke me" → predicted `hot_take`)
- Short `critique` posts that open with a confident claim before arguing ("The problem isn't pacing — it's that the arc was never set up" → predicted `hot_take`)

The failure mode is not random. It is a specific, consistent over-trigger on declarative sentence structure regardless of whether the post is making an argument, expressing a feeling, or asserting a reputation claim. With more `critique` and `reaction` examples containing this surface form, the model could learn to look past the sentence structure to the underlying content.

---

### Inter-Annotator Reliability

I had a classmate independently label 35 examples from my dataset using only the label definitions (no decision rules or edge case guidance). We then compared our labels.

| Metric | Value |
|--------|-------|
| Total examples compared | 35 |
| Agreement count | 26 |
| Simple agreement rate | 74.3% |
| Cohen's kappa | 0.61 |

Cohen's kappa of 0.61 indicates substantial agreement. The most common disagreements were on the `critique`/`hot_take` boundary (7 of 9 disagreements). In every case, my classmate labeled the post as `hot_take` while I labeled it `critique` — suggesting the `critique` label is harder to apply without the decision rule ("could you remove the evidence clause and the post still says the same thing?"). This confirms that the decision rule is doing real work, and that a model trained without it would produce noisier annotations.

---



### Deployed Interface

`interface.html` in the repo root is a standalone HTML file that accepts a post as input, calls Groq's `llama-3.3-70b-versatile` with the same system prompt used for the baseline evaluation, and displays the predicted label, confidence percentage, and a bar chart of all three class scores.

**To run it:** Open `interface.html` in any browser. Enter your Groq API key (used client-side only, never stored), paste a post, and click Classify.

The interface uses the zero-shot baseline prompt rather than the fine-tuned DistilBERT model because the fine-tuned model requires a Python runtime to serve. The zero-shot approach is deployable as a static HTML file with no server required.

---

## Spec Reflection

**One way the spec helped:** The checkpoint structure — specifically the instruction to "find the hardest edge case before you annotate 200 examples" — was the most valuable piece of guidance. I identified the `hot_take-with-one-detail` pattern before annotating, wrote a decision rule for it, and applied it consistently. Without that forcing function, I would have labeled those cases inconsistently and made the `critique`/`hot_take` boundary even noisier than it already was.

**One way implementation diverged:** The spec implies a clean design → collect → train → evaluate sequence. In practice, data collection drove label refinement. After seeing what r/Letterboxd posts actually looked like in bulk, I realized the `critique` label was harder to find than expected — the community skews heavily toward hot takes and reactions. If I had designed labels from memory rather than reading 30–40 posts first (as the spec instructs), I would have expected a much more even distribution. The spec was right to insist on reading first; I just didn't fully anticipate how much it would reshape my data collection strategy.

---

## AI Usage

**Instance 1: Label stress-testing**
I gave Claude my three label definitions and both edge cases and asked it to generate 10 posts sitting at label boundaries — specifically between `critique`/`hot_take` and `hot_take`/`reaction`. Three of the generated posts I couldn't classify confidently, which prompted me to add the "decorative vs. argumentative evidence" distinction and the *remove-the-clause* decision rule to my `hot_take` definition. The other seven were clearly classifiable.

**Instance 2: Dataset annotation**
All 203 examples were pre-labeled by Claude using my taxonomy definitions. I disclosed this in the `pre_labeled` column (all values = 1). I reviewed every label and reconsidered approximately 15–20% of cases, particularly at the `critique`/`hot_take` boundary. Claude's pre-labels were most reliable for clear `reaction` examples and least reliable for borderline `critique` cases, which it tended to classify as `hot_take`.

**Instance 3: Failure pattern analysis**
After getting my wrong predictions, I pasted all 15 misclassified examples into Claude and asked it to identify common features. It identified two patterns: (1) most `critique → hot_take` errors involved posts with strong declarative language regardless of content, and (2) several `reaction → hot_take` errors involved short posts with no hedging. I verified both by re-reading the examples. Both patterns held up. Claude also suggested that sarcasm might be a factor, but I found no sarcastic posts in the error set and discarded that claim.

---

*Dataset CSV, `evaluation_results.json`, `confusion_matrix.png`, and `interface.html` are committed in the repo root.*

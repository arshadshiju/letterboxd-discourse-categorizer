# TakeMeter Planning Document
## AI201 Project 3 | Arshad Shiju

---

## 1. Community Choice

**Community: r/Letterboxd (and the Letterboxd app review ecosystem)**

Letterboxd is a social film logging platform where users write diary entries and reviews for every film they watch. The r/Letterboxd subreddit aggregates the most notable, funny, or controversial reviews — and discussions about film opinions more broadly.

This community is an excellent fit for a classification task for three reasons:

1. **Discourse is inherently opinion-heavy and varied.** Every post is a take on a film. But the *form* of those takes ranges enormously — from a 200-word close-reading of cinematography to a one-liner like "this slaps." That variance is exactly what makes classification meaningful.
2. **The community has strong implicit norms about what makes a review "good."** Regular Letterboxd users recognize the difference between a thoughtful critique and a hot-take clout post. These norms are consistent enough to annotate reliably.
3. **Text is clean and bounded.** Letterboxd reviews are short-to-medium prose with a single subject (one film), which keeps the classification signal focused.

---

## 2. Label Taxonomy

### Labels (3 total)

---

**`critique`**
A post that engages substantively with the film as a work of craft or art. The reviewer references specific elements — direction, writing, performance, cinematography, score, pacing, structure — and makes a claim about how those elements succeed or fail. The argument is grounded in the film itself, not just the viewer's emotional reaction.

*Example 1:*
> "Villeneuve strips the source material down to its sensory core. The silence is the point — Zimmer's bass-heavy score and Greig Fraser's bleached-out frames do more narrative work than any dialogue could. The first act earns its runtime. The second doesn't, but by then you're already in."

*Example 2:*
> "The problem isn't that the third act is rushed — it's that the character arc was never set up to begin with. We're told she's changed, but every scene shows the same person making the same choices. The emotional payoff lands for audiences who wanted it to, not because the film built toward it."

---

**`hot_take`**
A post that makes a bold, confident claim about a film's quality, reputation, or legacy — without supporting the claim with evidence from the film itself. The opinion may be contrarian, hyperbolic, or simply stated as fact. The hallmark is assertion without argument.

*Example 1:*
> "Parasite is overrated. Good movie. Not the generational masterpiece everyone decided it was. The ending is melodramatic and the class commentary is less subtle than people think."

*Example 2:*
> "Interstellar is the best sci-fi film of the last 20 years and I'm tired of pretending it's not. The third act haters are just people who can't sit with ambiguity."

---

**`reaction`**
A post that expresses an immediate emotional or visceral response to a film — surprise, delight, grief, disgust — without structuring that response into an argument. Reactions are about the viewer's experience in the moment. They often use informal language, exclamations, or humor. The film itself is the stimulus; the post is the feeling.

*Example 1:*
> "I was NOT prepared for that ending. Sat in my car for 20 minutes. My hands were shaking. Letterboxd review: 'I need to lie down.'"

*Example 2:*
> "Watched Midsommar at 2am alone. Can confirm this was a mistake. Florence Pugh carries this movie on her back while everyone around her makes the worst decisions in film history. I'm never going to Sweden."

---

### Why These Three

The `critique` / `hot_take` / `reaction` taxonomy captures the three distinct *modes* of discourse that appear in Letterboxd spaces:

- **Critique** = argument from evidence
- **Hot take** = assertion without evidence
- **Reaction** = emotional response without argument

These are mutually exclusive in intent even when they blur in practice. They also reflect what the community *values* — long-form critiques get upvoted, hot takes get debated, reactions get related to. The labels matter here.

---

## 3. Hard Edge Cases

### The Primary Hard Case: Hot Take with One Supporting Detail

The most common ambiguous post looks like this:

> "Hereditary is the best horror film of the decade. Toni Collette's performance alone puts it above everything else — she's doing things with her face that shouldn't be physically possible."

Is this `critique` (mentions a specific performance element) or `hot_take` (bold claim about legacy, no actual argument about craft)?

**Decision rule:** If the post references a specific film element but uses it as *evidence for a ranking or legacy claim* rather than as the subject of genuine analysis, label it `hot_take`. The test: *could you remove the evidence clause and the post still says the same thing?* If yes — hot take. The example above reduces to "Hereditary is the best horror film of the decade" with a decorative justification. → `hot_take`.

**The counterexample that would be `critique`:** "Toni Collette's performance in Hereditary is remarkable because she never plays grief — she plays dissociation. The scene at the dinner table is the best-acted moment in modern horror." Here, the performance IS the subject. The claim is specific and argued. → `critique`.

### Secondary Edge Case: Emotional Reaction with an Opinion

> "Just finished Aftersun and I genuinely cannot stop crying. This film destroyed me. One of the greatest things I've ever seen."

Is this `reaction` (crying, visceral) or `hot_take` (strong legacy claim)?

**Decision rule:** If the claim ("greatest thing I've ever seen") is clearly emotionally driven and appears in the *context* of an ongoing emotional response rather than as a considered judgment, label it `reaction`. The hyperbole here is part of the feeling, not a standalone take. If the post leads with the claim and the emotion is secondary, lean `hot_take`.

---

## 4. Data Collection Plan

**Source:** r/Letterboxd on Reddit (public posts), plus Letterboxd review screenshots shared in the subreddit.

**Target distribution:**
| Label | Target count | Minimum |
|-------|-------------|---------|
| critique | 70–80 | 60 |
| hot_take | 70–80 | 60 |
| reaction | 60–70 | 50 |

Total: ~200–230 examples

**Collection method:** Manual collection from:
- r/Letterboxd "top posts this month" and "hot" sort
- Search for specific film titles in the subreddit (Dune, Hereditary, Everything Everywhere, Oppenheimer, Aftersun, Midsommar) to get varied content
- Letterboxd review screenshot posts (text extracted manually)

**If a label is underrepresented after 150 examples:** Search specifically for that type. For `critique`, search film titles known for discourse (Tár, The Master, Annihilation). For `hot_take`, search contrarian keywords ("overrated", "actually", "unpopular opinion"). For `reaction`, look for first-time watch threads and "just finished" posts.

**What I will NOT collect:** Posts that are just memes, polls, meta-discussion about Letterboxd the app, or posts with no review text at all.

---

## 5. Evaluation Metrics

**Primary metrics:**
- **Per-class F1** for all three labels. Accuracy alone is insufficient because my dataset may have slight imbalance, and a model that predicts `hot_take` for everything could get reasonable accuracy. F1 penalizes both over- and under-prediction.
- **Confusion matrix** to identify directional confusion — specifically whether the model blurs the `critique`/`hot_take` boundary (the hardest one) or the `hot_take`/`reaction` boundary.

**Secondary:**
- **Macro-averaged F1** as a single summary number that weights all three classes equally.
- **Per-class recall** for `critique` specifically — because misclassifying a genuine critique as a hot take is the most consequential error for community tool use.

**Why not just accuracy?** If the dataset ends up 40% `hot_take`, a dumb model that predicts `hot_take` always gets 40% accuracy. F1 forces the model to actually learn all three classes.

---

## 6. Definition of Success

**Minimum acceptable (deploy-worthy):**
- Fine-tuned model macro F1 ≥ 0.65
- No single class has F1 below 0.50
- Fine-tuned model outperforms zero-shot baseline by at least 10 percentage points in macro F1

**Good performance:**
- Macro F1 ≥ 0.75
- `critique` recall ≥ 0.70 (the model correctly identifies substantive posts)

**Too-good-to-trust threshold:** If any single class F1 exceeds 0.95, investigate for label leakage or overly easy label separation before celebrating.

**Rationale:** A macro F1 of 0.65 means the model is meaningfully better than random (0.33) and better than a naive majority-class predictor. For a community moderation or recommendation tool, that threshold is enough to surface patterns even if individual predictions aren't perfect.

---

## 7. AI Tool Plan

### Label Stress-Testing
After finalizing definitions above, I will give Claude my three label definitions and the two edge cases, and ask it to generate 10 posts that sit at the boundaries — specifically posts that could plausibly be `critique` or `hot_take`, and posts that could be `hot_take` or `reaction`. If any generated post stumps me, I will use it to sharpen the decision rule before annotating. This is the highest-value AI use in the whole project.

### Annotation Assistance
I will use an LLM (Claude or Groq) to pre-label batches of 30–40 examples at a time using my label definitions. I will review and correct every pre-assigned label — I will not skim. I will track which examples were pre-labeled in a `pre_labeled` column in my CSV (value: 1 if AI-assisted, 0 if manually labeled from scratch). This will be disclosed in the README AI usage section.

### Failure Pattern Analysis
After getting my wrong predictions from the notebook, I will paste all misclassified examples into Claude and ask: "Do these wrong predictions share any common features — post length, use of hyperbole, mentions of specific actors, sarcasm, or other patterns?" I will then verify each claimed pattern by re-reading the examples myself. I will include both what the AI found and what I confirmed or rejected.

---

## Hard Annotation Decisions (updated during Milestone 3)

*To be filled in during annotation — minimum 3 examples required.*

**Example 1:**
> "The Banshees of Inisherin made me feel things I don't have words for. Martin McDonagh writes grief like he's lived inside it. I don't know if it's a masterpiece but it felt like one."

*Candidate labels:* `reaction` (emotional, "don't have words for") or `hot_take` ("masterpiece" claim)
*Decision:* `reaction` — the masterpiece claim is hedged ("don't know if") and the post is clearly processing a feeling. The opinion is subordinate to the emotional experience.

**Example 2:**
> "Everything Everywhere All at Once is not the best film of 2022. It is the most emotionally manipulative. It mistakes busyness for depth. The multiverse mechanic is a delivery vehicle for a therapy-speak resolution that would embarrass a Pixar writer."

*Candidate labels:* `hot_take` (contrarian, bold) or `critique` (specific claims about structure and writing)
*Decision:* `critique` — the post makes specific, arguable claims about mechanism ("multiverse mechanic as delivery vehicle") and craft ("mistakes busyness for depth"). These are analytical observations, not just assertions. The contrarian framing is packaging; the content is analysis.

**Example 3:**
> "Oppenheimer is a 3-hour IMAX experience that somehow felt like 90 minutes. Nolan is just built different."

*Candidate labels:* `reaction` (experiential claim about runtime feel) or `hot_take` ("Nolan is built different")
*Decision:* `reaction` — the post is describing the experience of watching, not making a critical claim about the film. "Built different" is slang for "impressive," functioning as an emotion-laden compliment rather than a standalone reputation claim.

---

---

---

## Stretch Feature Plans (updated before starting)

### Inter-Annotator Reliability
I will ask a classmate to independently label 35 examples using only the label definitions — no decision rules or edge case guidance. I will track disagreements and compute Cohen's kappa. My hypothesis is that most disagreements will fall on the `critique`/`hot_take` boundary, which would confirm that the decision rule is doing real work and that annotators without it would produce inconsistent labels.

### Confidence Calibration
After fine-tuning, I will group test predictions into confidence buckets (0.50–0.65, 0.65–0.80, 0.80–0.95, 0.95–1.00) and check accuracy within each bucket. A well-calibrated model should have higher accuracy at higher confidence. I will report whether this holds and what it reveals about the model's self-assessment.

### Error Pattern Analysis
After getting wrong predictions, I will look for a systematic pattern — not just list individual failures. My hypothesis going in: the model will over-predict `hot_take` for any short, assertive post regardless of whether it is making an argument, expressing a feeling, or asserting a reputation. I will check whether post length and presence of declarative language correlates with misclassification.

### Deployed Interface
I will build a React interface that accepts a post as input, calls the fine-tuned model, and displays the predicted label and confidence score. The interface will be committed to the repo and documented in the README.



### Actual Results vs. Success Criteria

My definition of success required macro F1 ≥ 0.65 and no single class below F1 0.50. The actual results fell short of both thresholds:

| Metric | Target | Actual |
|--------|--------|--------|
| Fine-tuned macro F1 | ≥ 0.65 | 0.41 |
| Critique F1 | ≥ 0.50 | 0.00 |
| Fine-tuned accuracy | > baseline | 51.6% (baseline was 58.1%) |

The model did not meet the minimum acceptable threshold. This is a meaningful negative result.

### Why It Failed

The model collapsed to predicting `hot_take` for 25 of 31 test examples. `critique` got zero correct predictions. The cause is majority-class overfitting: with only 142 training examples and `hot_take` as the largest class (39%), DistilBERT found it safer to always predict the majority class than to learn the class boundaries.

The evaluation metrics I chose — per-class F1 and confusion matrix — were the right ones. They revealed exactly this failure pattern. Accuracy alone would have hidden it: a model that always predicts `hot_take` would still achieve ~39% accuracy, which looks reasonable but reveals nothing about whether the model learned anything.

### What Would Fix It

1. More `critique` examples (targeting 80+, currently only 55)
2. Class-weighted loss during training to penalize majority-class collapse
3. Larger total dataset — 200 examples is at the lower bound for DistilBERT fine-tuning on a 3-class subjective task

*Document last updated: after Milestone 5 results.*

# 99 — Interview Q&A: Claim Extraction & Tagging

Read this last, after chapters 00–06 are fresh. Questions are grouped behavioral → technical deep-dive
→ system design → retrospective. Model answers are written in first person as a template — adapt the
specifics (numbers, exact tools) to what you actually remember from the real project, and treat the
STAR summary in `00-README.md` as your 90-second version of question 1.

---

### 1. (Behavioral) Tell me about the Claim Extraction & Tagging project.

**A.** At Indegene, pharma marketing content has to go through medical-legal review before
publication — every claim needs to trace back to an approved source, and every asset needs its
mandatory Important Safety Information section verified present. That review was manual and didn't
scale with content volume. I built four NLP modules to automate the first pass: Claim Extraction &
Classification (find claim sentences, tag their type), Proof Reading (flag/correct grammatical
issues), ISI Classification (detect whether the ISI section is present and complete), and a Content
Comparator (check new claims against an approved-claims library for support or risky paraphrasing).
I fine-tuned BERT-family models on Sagemaker for the classification tasks, built an automated
retraining pipeline so models stayed current as new labeled data came in, and deployed everything
behind auto-scaled Sagemaker endpoints with Lambda/API Gateway routing and EventBridge health checks.
The system pre-triaged content so human reviewers spent their time on genuinely flagged items instead
of reading every asset end-to-end.

*(Follow-up to expect: "what was your specific role vs. the team's?" — have a clear, honest answer
ready about which module(s) you owned most directly.)*

---

### 2. (Behavioral) Describe a time a model you built made a mistake that mattered. How did you handle it?

**A.** In a compliance-adjacent system like this, false negatives (missing a genuinely unsupported
claim) are more costly than false positives (flagging something that turns out to be fine) — a
missed claim can reach publication, while an over-flag just costs a reviewer a few extra minutes. So
when tuning classification/similarity thresholds, I biased toward higher recall at the cost of more
manual review volume, and treated the human-in-the-loop review queue as the actual safety net, not
the model. When I saw a case the model missed, the response wasn't just "retrain" — it was to add
that case (and similar near-miss cases) to the evaluation set specifically, so the deployment gate
(chapter 04) would catch regressions in that exact failure mode going forward, not just look at
aggregate accuracy.

---

### 3. (Technical) Walk me through how you'd build a text classifier from scratch, before jumping to BERT.

**A.** Start simple to establish a baseline and understand task difficulty: TF-IDF vectorize the
text, train a Logistic Regression (or Naive Bayes) classifier, and measure accuracy/F1 on a held-out
set. This is fast to iterate on, interpretable (you can inspect which words drive which predictions),
and gives you a floor — if a fine-tuned BERT model can't clearly beat this baseline, the extra
complexity/cost isn't justified. Only after establishing that baseline would I move to contextual
embeddings/fine-tuning, specifically because claim classification needs context sensitivity
(negation, qualifiers) that bag-of-words representations structurally can't capture. See chapter 01
and `notebooks/01_text_classification_baseline.ipynb` for the concrete version of this.

---

### 4. (Technical deep-dive) How would you handle severe class imbalance in ISI classification, where most documents ARE compliant?

**A.** This is a classic rare-positive-class problem — if 95%+ of documents have a present and
complete ISI section, a model that always predicts "compliant" gets 95% accuracy while being useless.
A few things I'd combine:

- **Never evaluate on accuracy alone.** Use precision/recall/F1 on the *minority* class (missing or
  incomplete ISI) specifically, and look at the confusion matrix, not just an aggregate score.
- **Resampling.** Oversample the minority class (or undersample the majority) during training so the
  model sees a more balanced signal — SMOTE-style synthetic oversampling for tabular features, or
  simple duplication/augmentation for text examples.
- **Class-weighted loss.** Weight the loss function inversely proportional to class frequency
  (e.g., `class_weight="balanced"` in scikit-learn, or a weighted `BCEWithLogitsLoss`/cross-entropy in
  a neural setup) so misclassifying the rare class is penalized more heavily during training.
- **Threshold tuning, not just 0.5.** For a sigmoid/probability output, move the decision threshold
  based on the precision/recall trade-off you actually want — for ISI specifically, I'd bias the
  threshold toward higher recall on "non-compliant," since a missed non-compliant document is a real
  compliance risk and an over-flagged compliant one just costs a reviewer a quick check.
- **Treat the human-in-the-loop queue as the real safety net.** Even with all of the above, I
  wouldn't fully trust the model to auto-approve on the rare-but-critical side — anything below a
  confident-compliant threshold routes to review rather than being auto-passed.

---

### 5. (Technical deep-dive) How do you fine-tune BERT with a small labeled claims dataset without overfitting?

**A.** A few concrete levers, roughly in order of what I'd reach for first: (1) a small learning rate
(1e-5 to 5e-5) so fine-tuning nudges the pretrained weights rather than overwriting them; (2) few
epochs (2–4) with early stopping on a validation set, since a large pretrained model will start
memorizing a small fine-tuning set quickly; (3) dropout in the classification head; (4) weight decay
as regularization; (5) if data is really scarce, consider freezing the lower encoder layers (which
tend to encode generic syntax rather than task-specific signal) and only fine-tuning the upper layers
plus the head, which reduces the number of trainable parameters relative to full fine-tuning; (6)
data augmentation for text — back-translation or synonym replacement to synthetically expand the
training set; (7) k-fold cross-validation instead of a single train/val split, if the labeled set is
small enough that a single split gives a noisy performance estimate. Chapter 02 covers the mechanics
of why the small learning rate specifically matters — it's protecting the transfer-learning benefit
you're paying for by starting from a pretrained model in the first place.

---

### 6. (Technical deep-dive) How does a Sagemaker multi-model endpoint reduce cost vs. one endpoint per model?

**A.** With one endpoint per model, you pay for dedicated always-on compute sized to each model's
peak load individually — so if Claim Classification, ISI Classification, and Proof Reading each have
different, non-overlapping traffic peaks, you're still paying for four sets of idle capacity most of
the time. A multi-model endpoint hosts multiple models on shared compute, loading each model into
memory on demand and evicting less-recently-used ones under memory pressure, so you're provisioning
for the *combined* traffic pattern rather than the sum of each model's individual peak. The trade-off
is a cold-load latency hit the first time a request needs a model that isn't currently cached in
memory, and less strict per-model resource isolation — a memory-hungry model can push others out of
cache more aggressively. For this project, that trade-off favors MME: the four modules have related
but distinct traffic patterns, and a bit of occasional cold-load latency is an acceptable cost for
materially lower steady-state hosting cost.

---

### 7. (Technical deep-dive) How do you decide when to trigger automated retraining?

**A.** I'd combine two triggers rather than relying on just one: a scheduled cadence (e.g., monthly)
to absorb accumulated labeled data from human review corrections, plus a performance-degradation
trigger as a safety net — specifically, monitoring the rate at which human reviewers override the
model's classification/ISI decisions, and forcing an earlier retrain if that override rate spikes
between scheduled runs. I'd be cautious about a pure data/concept-drift statistical trigger (e.g., PSI
on the embedding distribution) as the *only* signal, because drift metrics can fire on benign
distribution shifts that don't actually hurt the model's accuracy — reviewer override rate is a more
directly meaningful, ground-truth-adjacent signal for whether the model still matches current
judgment. See chapter 04 for the full trigger taxonomy (scheduled / drift / degradation) and how they
map onto EventBridge + CloudWatch.

---

### 8. (Technical deep-dive) Your retraining pipeline just produced a new model. How do you decide whether to deploy it?

**A.** Never auto-deploy purely because a new model finished training. Evaluate it on a fixed,
held-out test set that's never used for training across retraining cycles, compute the same metrics
(per-class precision/recall/F1, not just accuracy) as the currently-deployed model, and only promote
automatically if it beats the current model by a meaningful margin on those metrics. If it doesn't
clearly win, hold it in the model registry and alert a human rather than silently discarding it or
silently deploying a possible regression — for a compliance-adjacent system, an undetected accuracy
drop (say, in ISI classification) is a real risk, not just an inconvenience. This "deployment gate"
stage is the difference between a retraining pipeline and a *safe* retraining pipeline. See chapter 04.

---

### 9. (Technical deep-dive) What's the difference between multi-class and multi-label classification, and which applies here?

**A.** Multi-class means each example gets exactly one label from a mutually-exclusive set (softmax
output, cross-entropy loss). Multi-label means each example can get zero, one, or many labels from a
non-exclusive set (independent sigmoid outputs per label, binary cross-entropy per label, evaluated
per-class and often averaged as macro/micro F1 rather than plain accuracy). Claim tagging is
naturally multi-label — a single sentence like "reduces symptom severity by 42% with a favorable
safety profile" is simultaneously an efficacy claim and touches on safety, and forcing one label per
claim would lose information. ISI classification, by contrast, is closer to multi-class/binary — a
document's ISI section is present, absent, or incomplete, one mutually-exclusive outcome per document.
Getting this framing right up front matters because it changes the loss function, the output layer,
and how you evaluate — treating a genuinely multi-label problem as multi-class silently discards
valid label combinations.

---

### 10. (Technical deep-dive) Explain the [CLS] token and why it's used for classification.

**A.** Every BERT input sequence is prepended with a special `[CLS]` token. Because BERT's
transformer encoder layers let every token attend to every other token, by the final encoder layer
the hidden state at the `[CLS]` position has effectively absorbed contextual information from the
whole input sequence. During fine-tuning, you attach a classification head (typically dropout + a
linear layer) on top of that final `[CLS]` hidden state, and train the whole thing so `[CLS]` becomes
a learned, task-specific pooled summary of the input suitable for whatever classification task you're
fine-tuning for — claim-type tagging, ISI presence, etc. It's a convention that works because the
architecture already gives every token global context by the final layer; `[CLS]` is just the
designated place to read that global summary out from.

---

### 11. (Technical deep-dive) Why use a Seq2Seq model for the Proof Reading module instead of a classifier?

**A.** Proof reading isn't a "pick one of N labels" problem — the output is itself a variable-length
corrected sentence, not a fixed label. That's a sequence-to-sequence task: an encoder reads the input
sentence into a contextual representation, and a decoder generates the corrected sentence
token-by-token, attending back to the encoder's representation at each generation step. In practice
this is usually implemented as fine-tuning a pretrained encoder-decoder model (T5/BART-style) on
pairs of (original, corrected) sentences — the same transfer-learning logic as fine-tuning BERT for
classification, just with a generative decoder instead of a classification head on top.

---

### 12. (Technical deep-dive) How does the Content Comparator avoid false positives from paraphrasing (e.g., "reduces" vs. "lowers")?

**A.** This is exactly the gap between surface-level and semantic similarity. Edit distance and
TF-IDF cosine similarity are both keyword/character-level — they can miss that "reduces the risk" and
"lowers the risk" mean nearly the same thing, since the individual words don't match. To catch that, I
layer in sentence-embedding similarity (Sentence-BERT-style) on top of the cheaper keyword filters:
TF-IDF cosine first, as a fast shortlist filter across a large approved-claims library, then
sentence-embedding cosine similarity on the shortlist to catch genuine paraphrases the keyword filter
missed. Edit distance stays useful for a narrower purpose — checking near-verbatim-quote claims that
are supposed to match approved label language almost exactly, where a large edit distance is itself
the signal. See chapter 03 for the full layered approach.

---

### 13. (System design) Design a claims-review pipeline where human reviewers must approve flagged content before publishing (human-in-the-loop).

**A.** Key design principles: the model layer only ever *proposes*, a human always has final sign-off
on anything below a high-confidence "clearly fine" threshold, and every human decision feeds back as
training signal.

```
Document ingested
      │
      ▼
Claim extraction + classification, ISI check, content comparator
      │
      ▼
Confidence/risk scoring per finding
      │
      ├── high-confidence "compliant, no issues" ──► auto-pass (still logged, sampled for periodic audit)
      │
      └── flagged (low confidence / unsupported claim / missing ISI / near-duplicate risk)
                  │
                  ▼
          Review queue, prioritized by risk severity
                  │
                  ▼
          Human reviewer sees: the flagged span, the model's reasoning
          signal (matched/unmatched source, similarity score, confidence),
          and the candidate approved-source match if any
                  │
          ┌───────┴────────┐
          ▼                ▼
     Approve             Reject / request revision
          │                │
          ▼                ▼
   Publish            Back to content author,
                       decision logged as new
                       labeled training data
```

Two things I'd insist on: (1) even auto-passed content is logged and periodically sampled for audit —
"high confidence" isn't the same as "verified," and you want a way to catch systematic model
overconfidence; (2) every human approve/reject decision becomes a labeled training example fed back
into the retraining pipeline (chapter 04), so reviewer judgment continuously improves the model rather
than being discarded after each review — this is what keeps the auto-pass threshold trustworthy over
time instead of drifting stale.

---

### 14. (System design) How would you monitor this system in production to know it's working correctly?

**A.** A few layers: (1) infrastructure-level — CloudWatch on endpoint latency, invocation counts,
error rates, auto-scaling activity; (2) scheduled functional health checks via EventBridge + Lambda,
invoking each endpoint with a known test input and asserting the expected output shape/values, not
just "is the instance up"; (3) model-quality-level — track the human-reviewer override rate over time
as the most meaningful proxy for whether the model's judgment still matches expert judgment, since
that's a direct ground-truth-adjacent signal rather than a proxy metric; (4) data-level — monitor
input distribution for drift (new vocabulary, new content types) as an early warning even before
override rate visibly moves. Alarms on (2) and a sustained rise in (3) are what I'd wire up to
actually trigger action — an unhealthy scheduled health check should page someone or trigger rollback
to the previous model version, and a sustained override-rate rise should trigger an earlier-than-scheduled
retrain.

---

### 15. (System design) A new therapeutic area launches with content types and claim language the model has never seen. What do you do?

**A.** This is the classic "new deployment context" version of data drift, and the right response
isn't "just retrain immediately" because you likely don't have labeled data for the new area yet.
I'd sequence it: (1) route new-therapeutic-area content through the pipeline but treat all outputs as
low-confidence-by-default, effectively forcing everything to the human review queue until the model's
had enough exposure — don't let a model with zero relevant training exposure auto-pass anything new;
(2) have reviewers' decisions on this new content become the first labeled examples specific to the
new area; (3) once enough labeled volume accumulates, fine-tune (or continue fine-tuning the existing
checkpoint) specifically incorporating the new area's data, and evaluate against a test set that
includes it before promoting; (4) update the approved-source library the Content Comparator checks
against, since it needs the new area's approved claims to have anything correct to compare new
content to. The theme: don't trust the existing model's confidence calibration on a distribution it's
never seen, and use the human-in-the-loop queue as the bootstrap mechanism for the new label set.

---

### 16. (Retrospective) What would you change if you rebuilt this today?

**A.** The biggest change is that I'd seriously evaluate LLM-based zero-shot or few-shot claim
extraction and classification as an alternative (or complement) to fine-tuned BERT for at least the
lower-volume modules. When this was originally built, fine-tuning a BERT-family encoder was the
practical way to get strong task-specific performance with a moderate labeled dataset. Modern
instruction-tuned LLMs can do zero-shot or few-shot claim extraction/classification with a well-designed
prompt (and structured output constraints), which would reduce the labeled-data bootstrap problem for
new therapeutic areas (question 15) and cut the fine-tuning/retraining maintenance burden
substantially for those modules. I'd keep fine-tuned models where they still win, though: high-volume,
latency-sensitive, well-defined classification tasks (ISI presence checking is a good candidate — it's
a comparatively narrow, high-volume, well-labeled task) are still often cheaper and more predictable
to serve as a small fine-tuned classifier than as an LLM call. I'd also invest earlier in the
override-rate monitoring and human-in-the-loop feedback loop (question 14) — that's the kind of
infrastructure that pays for itself over the system's lifetime and is easy to under-invest in early on
in favor of just shipping the first model.

---

### 17. (Technical deep-dive) Why might you choose Word2Vec-style embeddings over contextual embeddings for some part of this system, if BERT is strictly more powerful?

**A.** "Strictly more powerful" isn't quite right once you account for cost and latency, not just
accuracy. Word2Vec-style (or other static/non-contextual) embeddings are much cheaper to compute — a
lookup, not a transformer forward pass — and don't need a GPU at inference time. For a component
where throughput matters more than squeezing out the last few points of accuracy, or as a fast
first-pass filter before a more expensive contextual-embedding comparison (the TF-IDF-then-sentence-embedding
layering in the Content Comparator, chapter 03, is the same idea), a cheaper representation is a
reasonable engineering trade-off, not a mistake. The general principle: pick the cheapest
representation that clears the accuracy bar for that specific stage of the pipeline, and reserve the
expensive contextual model for the stage(s) where the extra accuracy actually changes a decision.

---

### 18. (Technical deep-dive) How would you evaluate the Content Comparator's similarity thresholds — what metric would you optimize?

**A.** I'd frame threshold selection as a precision/recall trade-off against a labeled set of "should
have been flagged as unsupported/near-duplicate" vs. "should not have been flagged" example pairs,
and plot precision/recall (or an ROC/PR curve) across candidate thresholds rather than picking a
threshold by intuition. Given the compliance context, I'd bias toward higher recall — a missed
unsupported claim is a bigger risk than an extra false-positive flag that costs a reviewer a couple of
minutes to dismiss — so I'd pick a threshold on the higher-recall end of the curve rather than the
point that simply maximizes F1. I'd also track this metric over time as the approved-source library
grows, since thresholds tuned against a smaller library may need retuning as more (and more varied)
approved content gets added.

---

### 19. (System design) This platform reviewed real regulated content for two pharma clients, Eli Lilly and AstraZeneca — how did you ensure a claim flagged for one client's content could never be seen by the other client's reviewers?

**A.** Isolation had to be enforced at every layer the data touches, not just at the UI. Concretely:
(1) **storage isolation** — each client's content, approved-source library, and model artifacts live
under separate S3 prefixes/buckets (`s3://claims-eli-lilly/...` vs. `s3://claims-astrazeneca/...`),
never a shared prefix filtered by a query parameter; (2) **compute isolation** — separate (or
explicitly namespaced) Sagemaker multi-model endpoints per client, so a routing bug can't accidentally
invoke the wrong client's model or, worse, return the wrong client's cached inference result; (3)
**identity/IAM isolation** — separate IAM roles scoped to each client's resources, so even a bug in
application code hits an AWS permission boundary before it could cross client lines; (4) **request-level
tagging** — the ECS Fargate API layer stamps every request with its `client_id` at the very first hop
(from the authenticated reviewer's session), and every downstream read — the review queue, the
approved-source lookup, the model endpoint call — filters on that `client_id`, so isolation isn't one
central check but the same scoping rule enforced independently at each stage. The design principle I'd
defend in an interview: isolation should be enforced redundantly at the infrastructure layer (S3/IAM/
endpoint boundaries), not solely as an application-level `WHERE client_id = ?` filter, because
infrastructure boundaries fail closed if application logic has a bug, and a compliance platform serving
two competing pharma companies can't treat that as a soft requirement.

---

### 20. (Technical deep-dive) What's the real-world cost of a false negative here (a non-compliant claim your model misses) versus a false positive, and how did that shape your evaluation/threshold choices in production?

**A.** The two error types are not symmetric, and treating them as if they were is the single biggest
threshold-tuning mistake you can make on a compliance platform. A **false positive** — flagging a claim
that was actually fine — costs a reviewer a few extra minutes to dismiss it. A **false negative** —
missing a claim that's actually unsupported or overstated — means non-compliant content can reach
publication for a real pharma client, which is a genuine legal and patient-safety risk, not just an
inconvenience: an unsubstantiated efficacy claim reaching a physician or patient is the exact failure
mode med-legal review exists to prevent. That asymmetry is why I biased every threshold decision toward
recall over precision on the "flag for review" side: I'd rather send the model's uncertain middle band
to a human reviewer than let the model's confidence auto-pass anything it isn't very sure about. In
practice this meant picking operating points from the higher-recall end of the precision/recall curve
(question 18) rather than the point that maximizes F1, and treating the human-in-the-loop review queue —
not the model — as the actual compliance safety net. The cost of that choice is real too — more review
volume, more reviewer hours — but for two clients whose regulated content is on the line daily, that
cost is the correct trade against the alternative.

---

### 21. (System design) How would you handle a Sagemaker endpoint outage during business hours when a client is mid-way through a content-review deadline?

**A.** First, detection has to be fast and specific: the EventBridge-scheduled health checks (chapter 05)
should catch a degraded/unhealthy endpoint within minutes via a failed synthetic invocation, not wait
for a spike in real user errors, and that failure should page on-call immediately with which client's
endpoint is affected (since Eli Lilly's and AstraZeneca's endpoints are namespaced separately, the alert
is already client-scoped). Second, mitigate before root-causing: if the current model version is the
likely cause (a bad deploy), the fastest safe action is rolling the endpoint back to the previous
registered model version (chapter 04's "keep the registry separate from the serving layer" is exactly
what makes this a fast repoint instead of a re-run of training) rather than debugging forward under
pressure. If it's an infrastructure issue rather than a model issue, auto-scaling's `MinCapacity` and
health-check-driven instance replacement should self-heal a single unhealthy instance; if the whole
endpoint is down, the Lambda routing layer is the natural place to fail over — e.g., serve a degraded
mode (route flagged-as-uncertain by default, or fall back to a lighter-weight/cached model) rather than
hard-failing every request, so the client's review workflow can keep moving on lower-risk content while
the endpoint is restored. Third, communicate: for a client mid-deadline, an incident like this needs a
status update to the client-facing side of the workflow, not just a silent internal fix — "content review
is degraded, ETA to full restoration is X" is a very different experience than requests silently timing
out. The unifying idea across all of this: because Eli Lilly's and AstraZeneca's endpoints are isolated
from each other, an outage on one client's endpoint is contained to that client — it should never be
possible for a routing failure on AstraZeneca's endpoint to also take down Eli Lilly's review workflow.

---

*Practice tip: pick three of these — one behavioral, one deep-dive, one system-design — and say them
out loud, unscripted, in under two minutes each. That's the real test of whether you know this well
enough to defend it live, not whether you can recognize a good answer on the page.*

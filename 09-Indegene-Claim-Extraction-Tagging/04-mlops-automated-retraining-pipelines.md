# 04 — MLOps: Automated Retraining Pipelines

## Why this chapter matters

The resume bullet says "built automated pipelines for NLP model retraining and maintenance of
associated model artifacts." This is the MLOps half of the project, and it's often the more
differentiating thing to talk about in an interview — plenty of candidates can say "I fine-tuned
BERT," fewer can talk fluently about *how you keep it working in production over time* without
someone manually babysitting it. This chapter covers why models need retraining at all, what triggers
should kick off a retrain, what stages a retraining pipeline runs through, and how you version and
trace model artifacts once you have more than one.

## Why models go stale

A model trained once and deployed forever degrades for a few distinct reasons, and knowing which one
you're looking at changes what you do about it:

- **Data drift.** The distribution of incoming content changes over time — new drug launches, new
  therapeutic areas, new marketing formats (social snippets vs. long-form leave-behinds) introduce
  vocabulary and sentence structures the model wasn't trained on. The input distribution shifts even
  if the *task* hasn't changed.
- **Concept drift.** The relationship between input and correct label changes — e.g., regulatory
  guidance updates what counts as an adequately-supported efficacy claim, so the same sentence that
  was "fine" under old guidance should now be flagged.
- **Label/taxonomy evolution.** New claim-type tags get added as the business's content categories
  evolve (a new therapeutic area might need new claim subtypes), and the existing model was never
  trained to predict them at all.
- **Performance degradation from feedback.** Human reviewers overriding the model's flags/tags is
  itself a label source — a rising override/disagreement rate is a direct signal the model's current
  behavior doesn't match current reviewer judgment.

## Retraining triggers

There are three common patterns for *when* to kick off a retrain, and a mature pipeline usually
combines more than one:

**1. Scheduled retraining.** Retrain on a fixed cadence (weekly, monthly) regardless of measured
drift, simply because new labeled data (human review corrections, newly approved source documents)
has accumulated. Simple to implement and reason about; the downside is you're either retraining more
often than needed (wasted compute) or not reacting fast enough to a sudden drift event.

**2. Drift-triggered retraining.** Continuously (or periodically) monitor the statistical properties
of incoming data and the model's own confidence/prediction distribution, and trigger a retrain when a
drift metric crosses a threshold — e.g., population stability index (PSI) on the input feature/embedding
distribution, or a rising rate of low-confidence predictions. This reacts faster to real shifts, but
needs more monitoring infrastructure to detect the drift in the first place.

**3. Performance-degradation-triggered retraining.** Track a live proxy for model quality — most
directly, the rate at which human reviewers *override* the model's classification or ISI decision —
and trigger a retrain when that override rate crosses a threshold. This is arguably the most directly
meaningful trigger for a human-in-the-loop system like this one, since reviewer disagreement is a
ground-truth signal, not a proxy.

For claim/ISI classification specifically, a realistic setup layers scheduled retraining (say,
monthly, to absorb accumulated review corrections as new labeled data) with a performance-degradation
trigger as a safety net that can force an earlier retrain if reviewer overrides spike between
scheduled runs — this maps directly onto the "EventBridge Scheduler" component in the chapter 00 MLOps
diagram, which is well suited to the scheduled part, while a CloudWatch alarm on a custom
"override rate" metric is well suited to the degradation-triggered part.

## Pipeline stages

An automated retraining pipeline isn't just "run the training script again" — a defensible one has
distinct, ordered stages, each acting as a gate the run has to pass before moving to the next:

```
 1. Data validation
    - schema check (expected columns/fields present)
    - label distribution sanity check (no sudden collapse to one class)
    - dedup / leakage check (no train/test overlap)
         │
         ▼
 2. Training
    - fine-tuning job (Sagemaker — chapter 05)
    - fixed random seed, logged hyperparameters
         │
         ▼
 3. Evaluation
    - held-out test set: accuracy, precision/recall/F1 per class
    - compare against the currently-deployed model's metrics on the SAME test set
         │
         ▼
 4. Model registry
    - log the new model artifact, its metrics, its training data version,
      and its lineage (what code + data produced it) — versioned, not overwritten
         │
         ▼
 5. Deployment gate
    - automatic promotion only if new model beats the deployed model by a
      meaningful margin on the evaluation metrics
    - otherwise: hold in registry, alert a human, do NOT auto-deploy a regression
```

The **evaluation gate is the most important stage to be able to defend in an interview.** A pipeline
that retrains and redeploys automatically without comparing against the current production model's
performance on a fixed benchmark set is a pipeline that can silently ship a regression — for a
compliance-adjacent system, an undetected accuracy drop in ISI classification isn't just an
inconvenience, it's a compliance risk. The gate should compare like-for-like: same test set, same
metrics, and ideally hold out that test set from ever being used in training so the comparison stays
honest over successive retraining cycles.

## Artifact versioning and lineage

Once you have more than one model version in play (which happens the moment you retrain), you need
to be able to answer, for any deployed model: *what data trained it, what code trained it, and what
its evaluation metrics were.* That's artifact lineage. In practice this means:

- **Never overwrite a model artifact in place.** Every training run writes to a new versioned path
  (`s3://.../claim-classifier/v17/model.tar.gz`, chapter 05 covers S3's role here concretely), so you
  can always roll back to a previous version if a new one misbehaves in production.
- **Tag each artifact with its provenance**: training data snapshot/version, code commit hash,
  hyperparameters, evaluation metrics. This is what lets you answer "why did the model's behavior
  change between last month and this month" — you can diff the training data and code between two
  artifact versions.
- **Keep the registry separate from the serving layer.** The model registry is the source of truth
  for "what versions exist and how good are they"; the deployment step is a separate, explicit action
  that points the serving endpoint at a specific registered version. This separation is what makes
  rollback a fast, low-risk operation (repoint the endpoint at the previous version) rather than a
  re-run of training.
- **Keep lineage client-scoped, not just version-scoped.** In production this pipeline runs
  independently for Eli Lilly and AstraZeneca — separate S3 prefixes, separate registry entries, separate
  retraining schedules — so "what data trained this artifact" always resolves to one client's data only;
  chapter 05 covers the endpoint- and IAM-level isolation this depends on.

## Tying it back

"Built automated pipelines for NLP model retraining and maintenance of associated model artifacts" is
describing exactly this: a pipeline that (a) decides when to retrain (scheduled and/or
drift/degradation-triggered), (b) runs training through validation → train → evaluate → register →
deploy-gate stages rather than blindly redeploying, and (c) keeps every model artifact versioned in
S3 with enough metadata to trace lineage and roll back. Chapter 05 covers the concrete AWS
implementation of the training and serving stages (Sagemaker, S3, EventBridge, Lambda/API Gateway);
this chapter is the platform-agnostic reasoning behind why the pipeline is shaped the way it is —
the part of the answer that stays true regardless of which cloud you're on.

# 03 — Explainable AI: SHAP, LIME, and Shapash

## Why this chapter matters for this project

"Explainable AI, Shapash, Lime" is on the candidate's core resume skill list and is explicitly mapped
to this project in the curriculum's skills matrix. In a model-risk-monitoring context, explainability
is not a nice-to-have research topic — it's often a **regulatory requirement**. SR 11-7-style model
risk frameworks expect that when a model makes or influences a decision, someone can explain *why*,
not just *what*. This chapter covers the three explainability tools most associated with the
candidate's resume — SHAP, LIME, and Shapash — what each is actually good for, and how they support
the "accuracy" and "completeness" auditing this project's evaluation harness needed to produce.

## Why explainability shows up inside an LLM evaluation project

At first glance, SHAP/LIME (built for tabular and classical ML models) and Shapash (a dashboard layer
on top of them) seem like an odd fit for evaluating a generative text assistant. The connection is
this: a production evaluation harness like the one in this project is not *only* scoring the LLM's
text output — it likely also includes **classical ML components** that classify or score responses:

- A toxicity/harmfulness classifier deciding whether a response should be flagged (Chapter 04).
- A bad-actor / prompt-injection detector deciding whether an incoming query looks adversarial.
- A relevance or intent classifier routing queries or deciding which knowledge base to retrieve from.
- Possibly a lightweight "risk score" model combining several metric signals into one number for the
  dashboard.

Every one of these is a natural fit for SHAP/LIME/Shapash, because they're typically simpler
classifiers (logistic regression, gradient boosting, or small text classifiers) sitting *around* the
LLM — and a bank's model risk reviewers will ask "why did the moderation layer flag this response, or
fail to flag that one?" just as readily as they'll ask about the LLM itself. Explainability on these
surrounding classifiers is what turns "the moderation model flagged it" into "the moderation model
flagged it because of these three words/features, with these weights" — which is what a compliance
reviewer or auditor actually needs to sign off.

## SHAP (SHapley Additive exPlanations)

SHAP is grounded in **Shapley values**, a concept from cooperative game theory: given a "game" (the
model's prediction) played by several "players" (the input features), how do you fairly distribute
credit for the outcome among the players? SHAP treats each feature as a player and computes, for a
single prediction, how much each feature pushed the prediction above or below the model's average/
baseline output.

Key properties that make SHAP attractive for a regulated environment:

- **Theoretically grounded consistency.** Shapley values have provable fairness properties (if a
  feature's contribution increases in every possible scenario, its attributed importance can only
  increase or stay the same) — this matters when an auditor asks "is this explanation method itself
  reliable," because you can point to the underlying math rather than "it's a popular library."
- **Both local and global explanations.** SHAP values for a single prediction explain that one
  decision (local); averaging `|SHAP value|` across many predictions gives global feature importance
  (which features matter most across the whole model) — useful for both "why was this response
  flagged" (local, for a specific audit case) and "what does our harmfulness classifier generally key
  off of" (global, for model documentation).
- **Additivity.** SHAP values for a prediction sum exactly to (prediction - baseline), so the
  explanation is a literal accounting of the prediction, not just a rough approximation.

Cost: SHAP is computationally expensive for exact calculation on complex models (exact Shapley values
require evaluating all feature subsets); in practice you use approximations like `TreeSHAP` (fast,
exact for tree-based models like XGBoost/LightGBM) or `KernelSHAP` (model-agnostic but slower, sampling-
based).

```python
import shap
explainer = shap.TreeExplainer(harmfulness_classifier)
shap_values = explainer.shap_values(response_features)
shap.summary_plot(shap_values, response_features)  # global feature importance
```

## LIME (Local Interpretable Model-agnostic Explanations)

LIME takes a different, simpler approach: to explain one prediction, it perturbs the input slightly
(for text: dropping/masking words; for tabular data: jittering feature values), observes how the
model's prediction changes across those perturbations, and fits a simple, interpretable **local
surrogate model** (typically weighted linear regression) around that one prediction's neighborhood.
The surrogate's coefficients become the explanation — "removing the word 'urgent' would have dropped
the harmfulness score by 0.3, so 'urgent' contributed +0.3 to the flagged score."

Key properties:

- **Model-agnostic and lightweight.** LIME treats the underlying model as a black box it can only
  query — this makes it trivially applicable to almost anything, including LLM-based classifiers
  where you can't easily get internal gradients or tree structure.
- **Purely local.** LIME does not claim to explain the model globally, only the neighborhood around
  one specific input — which is honest but also a limitation: you can't average LIME explanations
  across many predictions the way you can with SHAP and get a reliable global picture.
- **Sensitive to perturbation strategy.** For text, how you perturb (word deletion vs. synonym swap vs.
  masking) materially changes the explanation, and LIME explanations can be somewhat unstable
  (re-running LIME on the same input can give slightly different coefficients due to the random
  sampling involved) — worth mentioning as a known limitation if asked.

```python
from lime.lime_text import LimeTextExplainer
explainer = LimeTextExplainer(class_names=["safe", "harmful"])
exp = explainer.explain_instance(response_text, classifier.predict_proba, num_features=8)
exp.as_list()  # [('urgent', 0.31), ('click here', 0.22), ...]
```

## Shapash — packaging SHAP/LIME for non-technical stakeholders

Shapash (open-sourced by the French insurer MAIF) is not a competing algorithm to SHAP/LIME — it's a
**dashboard and reporting layer** built on top of them, specifically designed to make explanations
consumable by people who are not data scientists: risk officers, compliance reviewers, auditors,
business stakeholders. This is the layer that matters most for the "regulatory model-risk
documentation" half of this chapter's brief.

What Shapash adds on top of raw SHAP/LIME output:

- **Human-readable feature labels and value formatting**, instead of raw column names and encoded
  values — turning `feat_23 = 1` into "Contains urgency language: Yes."
- **An interactive web dashboard** (`SmartExplainer` + `SmartApp`) that lets a non-technical reviewer
  browse individual predictions, filter by outcome, and see local + global explanations without
  writing any code — exactly the artifact a model risk committee would want attached to a model's
  documentation package.
- **Contribution plots and stability reports** out of the box, and **automated summary report
  generation**, which maps directly onto the kind of "evidence the model was reviewed and explained"
  documentation SR 11-7-style frameworks expect.

```python
from shapash import SmartExplainer
xpl = SmartExplainer(model=harmfulness_classifier)
xpl.compile(x=response_features)
xpl.plot.features_importance()   # global view for model documentation
app = xpl.run_app()              # interactive dashboard for reviewers
```

## Choosing between them — and how this ties back to "completeness" and "accuracy" auditing

A defensible framing for an interview:

- **SHAP** when you need theoretically consistent, auditable explanations — especially for tree-based
  or otherwise structured classifiers in the pipeline (e.g., a gradient-boosted harmfulness/bad-actor
  score) — and you're willing to pay the computational cost.
- **LIME** when you need a fast, model-agnostic, "good enough" local explanation, particularly useful
  for quick debugging of *why one specific flagged response was flagged* during development, or when
  the underlying model doesn't have a SHAP-optimized explainer available.
- **Shapash** when the audience is not the engineering team — when the deliverable is a reviewable
  artifact for risk/compliance stakeholders, not a debugging tool for developers.

The link back to "accuracy" and "completeness" auditing: explainability doesn't itself measure
accuracy or completeness the way Ragas metrics do (Chapter 02) — it answers the *next* question a
reviewer asks after seeing a metric score: "the harmfulness classifier scored this response 0.9 —
why?" Without SHAP/LIME/Shapash, "why" is a black box, which is precisely the state a model risk
framework is designed to prevent. Explainability is what makes the metric scores from Chapters 01-02
*defensible*, not just *produced*.

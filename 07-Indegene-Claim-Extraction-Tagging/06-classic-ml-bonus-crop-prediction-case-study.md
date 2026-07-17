# 06 — Bonus: Classic ML Case Study — Crop Production Prediction

## Why this chapter is here

Everything else in this course is NLP and deep learning. This chapter is deliberately different: it's
your personal project, **"ML based Effective Crop Production"** — predicting a suitable crop for
cultivation from sensor readings (rainfall, temperature, soil moisture, humidity, pH) — and it's
included because interviewers routinely test whether you understand *classic* tabular ML fundamentals
alongside the deep learning material, since a huge share of real-world ML work is still tabular
data and tree-based models, not transformers. It's also a good STAR-answer for "tell me about a
project outside of work," if asked.

## Framing the problem

Given a row of sensor readings — rainfall (mm), temperature (°C), soil moisture (%), humidity (%),
soil pH — predict which crop is best suited for cultivation under those conditions. This is a
**multi-class classification** problem (not multi-label — a given plot of land gets one recommended
crop, not several simultaneously) with a small number of continuous numeric features and a
categorical target with a moderate number of classes (rice, wheat, maize, cotton, etc.).

This is architecturally about as far from BERT-based claim classification as you can get within the
same résumé, and that contrast is useful to articulate explicitly in an interview: text classification
needs a way to turn unstructured language into features (chapter 01); tabular sensor classification
*already has* well-defined, meaningful numeric features — the modeling problem is different in kind,
not just in scale.

## Step 1 — feature engineering from sensor data

With tabular sensor data, "feature engineering" is a lighter lift than with text, but it's not
nothing:

- **Raw features**: rainfall, temperature, soil moisture, humidity, pH — each already a meaningful,
  independently interpretable number. No embedding or vectorization step needed.
- **Scaling**: tree-based models (see below) are scale-invariant, so scaling doesn't matter for them,
  but if you ever compare against a distance-based or linear model (k-NN, logistic regression, SVM)
  you'd standardize features first, since rainfall (0–300mm) and pH (0–14) live on wildly different
  scales and would otherwise dominate a distance calculation purely due to units.
- **Domain-informed derived features**: e.g., a rainfall-to-temperature ratio, or bucketing pH into
  acidic/neutral/alkaline bands, can help a model pick up agronomic thresholds more easily than raw
  continuous values alone — though with a good tree-based model this is often unnecessary, since trees
  can already learn nonlinear thresholds and interactions directly from raw features (that's a point
  worth making explicitly if asked "would you engineer more features here").
- **Missing/noisy sensor readings**: real sensor data has dropouts and noise; a production version of
  this system would need imputation (e.g., median fill or a sensor-specific interpolation) and basic
  range validation (a soil moisture reading of 150% is a sensor fault, not a real value) before it
  ever reaches the model.

## Step 2 — model selection: why tree-based models for tabular data

**Random Forest** (an ensemble of decision trees, each trained on a bootstrap sample of the data and
a random subset of features, with predictions aggregated by majority vote) is the natural default for
this kind of problem, and it's worth being able to explain *why* tree-based models tend to win on
tabular data specifically, versus neural networks:

- **No feature scaling needed** — trees split on thresholds, so the raw units of rainfall vs. pH
  don't need to be normalized against each other.
- **Handles nonlinear relationships and interactions natively** — e.g., "high rainfall AND low
  temperature favors rice" is exactly the kind of conjunctive, nonlinear rule a decision tree
  represents naturally as a sequence of splits, without needing you to hand-engineer an interaction
  term.
- **Robust with modest data sizes** — deep learning wants a lot of data to shine; a random forest with
  a few thousand labeled rows across five numeric features tends to generalize well without
  elaborate regularization.
- **Interpretable via feature importance** — you can directly ask the trained model which sensor
  readings mattered most for its predictions (below), which matters if you ever need to explain a
  recommendation to a non-technical stakeholder (e.g., an agronomist).

A single **Decision Tree** is more interpretable still (you can literally draw the tree) but higher
variance — small changes in training data can produce quite different trees. Random Forest trades a
bit of that interpretability for materially better generalization by averaging over many
decorrelated trees, which is almost always the right trade for a deployed prediction system over a
single tree.

```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import classification_report

X = df[["rainfall", "temperature", "soil_moisture", "humidity", "ph"]]
y = df["crop_label"]

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y
)

model = RandomForestClassifier(n_estimators=200, random_state=42)
model.fit(X_train, y_train)

y_pred = model.predict(X_test)
print(classification_report(y_test, y_pred))
```

`stratify=y` matters here specifically because with a moderate number of crop classes, a plain random
split can leave some classes under-represented in the test set purely by chance — stratifying keeps
the train/test class proportions matched to the full dataset.

## Step 3 — evaluating a multi-class recommendation-style output

Since this is multi-class (one crop recommended per row, several possible crops), evaluation should
go beyond plain accuracy, for the same reason it should for claim classification: accuracy can hide
poor performance on individual classes, especially if crop classes are imbalanced in the training
data (some crops may be far more common in the historical dataset than others).

- **Per-class precision/recall/F1** via `classification_report` — shows whether the model is
  systematically weaker on any particular crop.
- **Confusion matrix** — for a recommendation-style task, *which* crops get confused with which
  others is informative in its own right: two crops that thrive under similar sensor conditions (e.g.,
  two water-intensive crops) being confused with each other is a much more benign error than confusing
  a water-intensive crop with a drought-tolerant one, even though both show up as "one misclassification"
  in an aggregate accuracy number.
- **Feature importance** (`model.feature_importances_`) — beyond validating the model, this is a
  genuinely useful agronomic sanity check: if the model says rainfall and soil moisture dominate its
  decisions and temperature/pH barely matter, that's either a real domain insight or a sign the
  training data doesn't have enough variation in temperature/pH to let the model learn their effect —
  worth distinguishing between those two explanations before trusting the importances at face value.

```python
import pandas as pd

importances = pd.Series(model.feature_importances_, index=X.columns).sort_values(ascending=False)
print(importances)
```

## Tying it back

This project is a useful interview pivot point precisely because it's *not* NLP: if an interviewer's
line of questioning drifts toward "have you worked with structured/tabular data, not just text,"
this is the answer, and it demonstrates the same underlying judgment as the pharma NLP work — picking
a model class appropriate to the data structure (trees for tabular sensor data vs. transformers for
text), engineering features appropriate to the modality, and evaluating with per-class metrics rather
than a single aggregate number when class-level errors matter. Notebook
`05_crop_prediction_classic_ml.ipynb` builds a small synthetic version of this dataset and walks
through exactly this random forest workflow, including the feature-importance plot.

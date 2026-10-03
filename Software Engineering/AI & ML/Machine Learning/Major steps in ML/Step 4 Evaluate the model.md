---
tags: [ai-ml, ai-ml/machine-learning, course]
aliases: [Evaluate the Model, ML Step 4, ML Metrics, Precision and Recall, F1 Score]
updated: 2026-09-27
---
# Step 4 Evaluate the model

> [!tldr] TL;DR
> After training, measure how well the model does **on data it has never seen** (the test set), using a metric that matches the problem. Classification uses accuracy, **precision**, **recall**, **F1** and **log loss**; regression uses **MAE**, **RMSE** and **R²**. Accuracy alone is misleading on imbalanced data. Compare training vs test performance to spot **overfitting**.

*After you've collected data and trained a model, you evaluate how well it performs. The metrics are usually very specific to the problem you defined; as you grow in ML you'll meet many more.*

## Accuracy and the confusion matrix
**Model accuracy**: how often the model predicts the correct class (e.g. the correct flower species). Most other classification metrics come from the **confusion matrix**:

| | Predicted positive | Predicted negative |
|---|---|---|
| **Actually positive** | TP (true positive) | FN (false negative) |
| **Actually negative** | FP (false positive) | TN (true negative) |

```text
accuracy  = (TP + TN) / all
precision = TP / (TP + FP)     "of everything I flagged, how much was right?"
recall    = TP / (TP + FN)     "of everything real, how much did I catch?"
F1        = 2 · precision · recall / (precision + recall)
```

> [!warning] Why accuracy can lie
> If 99% of camera frames contain no spill, a model that always says "no spill" is 99% accurate and completely useless. On imbalanced data, look at precision, recall, F1, or PR/ROC curves. See [[Example 3 Detecting Spills]].

## Metrics tailored to the use case

### 1. Precision
How trustworthy the classifier is **when it predicts positive**: true positives divided by all positive predictions.
*Example*: the model says 100 people are diabetic and 80 really are. Precision = 80/100 = **0.8**.

### 2. Recall
The classifier's **sensitivity**: true positives divided by all actual positives in the dataset.
*Example*: of those 100 predictions, 80 were right, but there are 120 diabetic people in total. Recall = 80/120 ≈ **0.67**.

Precision and recall trade off: lowering the decision threshold catches more positives (recall up) but raises false alarms (precision down). Pick based on which error costs more: missing cancer (favour recall) vs flagging legit emails as spam (favour precision).

### 3. Log loss (cross-entropy)
Measures how **confident and correct** the predicted probabilities are, not just the final class. It punishes confident wrong answers heavily.
*Example*: will a customer buy a t-shirt? They did buy one. A model that said 80% gets loss −ln(0.8) ≈ **0.22**; one that said 55% gets −ln(0.55) ≈ **0.60**. Both "predicted t-shirt", but the first was more certain and is rewarded for it. A model that said 5% would get −ln(0.05) ≈ **3.0**.

### 4. Mean Absolute Error (MAE)
The **average absolute difference** between predicted and actual values in regression, in the same units as the target.
*Example*: 10 houses, actual prices $200k, 250k, ..., 650k (step $50k); predicted $180k, 240k, 290k, 360k, 410k, 480k, 510k, 560k, 610k, 640k. The absolute errors are 20, 10, 10, 10, 10, 30, 10, 10, 10, 10 (thousand), sum 130k, so MAE = **$13,000**. *(The original note said $15,000; the arithmetic gives $13,000.)*

Related: **RMSE** (root mean squared error) squares errors before averaging, so big misses count more. For the same houses RMSE = √(2.2×10⁹ / 10) ≈ **$14,800**. This is the "RMS" used in [[Example 1 Predicting Home Prices]].

### 5. Hinge loss
Used for maximum-margin binary classifiers such as SVMs, with labels y ∈ {−1, +1} and a raw score f(x):

```text
hinge = max(0, 1 − y · f(x))
```

It is 0 when the prediction is on the correct side **with a margin of at least 1**, and grows linearly otherwise.
*Example*: a fraudulent transaction (y = +1) scored 2 → max(0, 1 − 2) = 0. A legit one (y = −1) scored −1 → max(0, 1 − 1) = 0. Total loss **0**: both are correct with enough margin. If the legit one had scored +0.5 instead: max(0, 1 + 0.5) = 1.5. *(The original example computed 1; the correct total is 0.)*

### 6. R² (coefficient of determination)
The **goodness of fit** of a regression: the proportion of variance in the target that the model explains. 1 is perfect, 0 is no better than always predicting the mean, negative is worse than that.

```text
R² = 1 − Σ(y − ŷ)² / Σ(y − ȳ)²
```

*Example*: for the 10 houses above, the squared errors sum to 2.2×10⁹ and the total variance around the mean ($425k) is 2.06×10¹¹, so R² ≈ 1 − 0.011 ≈ **0.99**.

### 7. Quantile (pinball) loss
Evaluates **quantile regression**, where you predict a percentile rather than the mean (e.g. "the 90th percentile delivery time"). For quantile q:

```text
loss = q · (y − ŷ)         if y ≥ ŷ   (under-prediction)
       (1 − q) · (ŷ − y)   if y < ŷ   (over-prediction)
```

For the median (q = 0.5) it's half the absolute error, so it behaves like MAE. For q = 0.9, under-predicting costs 9 times more than over-predicting, which pushes predictions up to the 90th percentile.

### 8. F1 score
The **harmonic mean of precision and recall**; high only if both are high.
*Example*: precision 0.8 and recall 0.67 (from above) → F1 = 2·0.8·0.67 / (0.8 + 0.67) ≈ **0.73**.

### 9. KL divergence
Measures how one probability distribution P differs from a reference distribution Q: the information lost when Q is used to approximate P.

```text
KL(P ‖ Q) = Σ P(x) · log(P(x) / Q(x))
```

It is ≥ 0, equals 0 only when the distributions match, and is **not symmetric**. It shows up in variational autoencoders, knowledge distillation, and in LLM fine-tuning with RLHF, where a KL penalty keeps the tuned model close to the original.

## Summary table
| Task | Metric | Use when |
|---|---|---|
| Classification | Accuracy | classes are balanced |
| | Precision | false positives are costly |
| | Recall | false negatives are costly |
| | F1 | you need a balance, classes imbalanced |
| | Log loss | you need good probabilities |
| | ROC-AUC / PR-AUC | comparing models across thresholds |
| Regression | MAE | errors in real units, robust to outliers |
| | RMSE | large errors are especially bad |
| | R² | explaining variance, comparing fits |
| | Quantile loss | predicting ranges/percentiles |
| Clustering | Silhouette | no labels ([[Example 2 Micro-genre Book Genre]]) |

## Overfitting vs underfitting
- **Underfitting** (high bias): poor on both train and test. The model is too simple.
- **Overfitting** (high variance): great on train, poor on test. The model memorised noise. Fix with more data, regularisation, simpler models, early stopping. See [[Neural Networks & Deep Learning Basics]].

## Senior perspective
- Pick the **business metric first** (cost of a missed fraud vs a blocked customer), then the ML metric that tracks it.
- Report metrics per **segment** (region, device, new vs returning users); a good average can hide a failing group, which is also how fairness issues show up.
- Use **cross-validation** for small data, and a **time-based split** for anything time-dependent.
- Offline metrics are a proxy. Confirm with an online test (A/B or shadow mode) before full rollout.
- LLM systems need different evaluation (rubrics, LLM-as-judge, golden sets): see [[LLM Evaluation & Observability]].

## Interview questions

> [!question]- Precision vs recall: which would you optimise for a cancer screening model?
> Recall: missing a real case (false negative) is far worse than a false alarm that leads to a follow-up test. You'd still watch precision so the follow-up load stays manageable.

> [!question]- Why is accuracy a bad metric for fraud detection?
> Fraud is rare, so predicting "not fraud" every time gives very high accuracy while catching nothing. Use precision, recall, F1 or PR-AUC instead.

> [!question]- MAE vs RMSE?
> Both in target units. RMSE squares errors, so it penalises large errors more and is more sensitive to outliers; MAE treats all errors linearly. RMSE ≥ MAE always.

> [!question]- How do you detect overfitting?
> Training performance keeps improving while validation/test performance stalls or gets worse. A large train-test gap is the signal.

## Related
- [[Major steps in ML]] · [[Machine Learning]] (course index) · [[AI & ML Index]]
- Previous: [[Step 3 Train the model]] · Next: [[Step 5 Use the model]]
- [[LLM Evaluation & Observability]]

## Sources
- AWS DeepRacer Student: [Introduction to machine learning](https://student.deepracer.com/learn/85a3a216-703b-40c8-ad70-374767eba381)
- scikit-learn: [Metrics and scoring](https://scikit-learn.org/stable/modules/model_evaluation.html) · [hinge_loss](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.hinge_loss.html) · [mean_pinball_loss](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.mean_pinball_loss.html)
- Google ML Crash Course: [Classification: accuracy, precision, recall](https://developers.google.com/machine-learning/crash-course/classification/accuracy-precision-recall)

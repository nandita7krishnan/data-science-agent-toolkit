---
name: model-evaluator
description: Score a trained model honestly — pick the right metric, compare to a naive baseline, report bootstrap CIs, plot residuals/calibration, and break out by slices.
when_to_invoke:
  - "Evaluate this model"
  - "How well does the model do?"
  - "Score model on test set"
  - "Is this model production-ready?"
---

# Model Evaluator

A model's score on the test set is necessary but not sufficient. **A 90% accurate model that's 1pt better than predicting the majority class is not interesting.** This skill produces an honest evaluation: metric matching the question, baseline comparison, CI, segment breakouts, and calibration.

## Inputs

- The trained model.
- The held-out test set (must not have been used in training or hyperparameter tuning).
- The target's business meaning (drives metric choice).
- Optional: business-relevant slices to break out by (region, surface, lifecycle stage).

## Outputs

- Primary metric with a 95% bootstrap CI.
- Naive-baseline score on the same test set with CI.
- Diagnostic plots: residuals (regression) or confusion + calibration + ROC/PR (classification).
- Segment-level performance table.
- One-paragraph honest summary: is this production-ready, or not?

## Procedure

1. **Pick the metric to match the question:**
   - **Regression** (predicting a continuous quantity, e.g., LTV): RMSE if errors are roughly symmetric and you want to penalize large errors; MAE if you care about typical-case error and outliers exist.
   - **Classification, balanced classes** (e.g., A/B classification): accuracy is fine; AUC for ranking.
   - **Classification, imbalanced** (e.g., churn at 5% base rate): **never accuracy alone.** Use precision, recall, F1, and PR-AUC at the operating threshold the business will use.
   - **Probabilistic predictions**: log-loss + calibration curve. A sharp model that's miscalibrated is dangerous.
2. **Compute on the held-out test set.** Not train, not validation. If you cross-validated, the test set is still untouched.
3. **Compute the naive baseline** on the same test set:
   - **Regression**: predict the train-set mean (or median for MAE). Or predict-the-last-value for forecasting.
   - **Classification**: predict the train-set majority class (for accuracy); or predict the train-set base rate (for log-loss / Brier). Or "stratified random" for AUC.
   - **A model that doesn't beat its naive baseline is not a model.** Stop and explain.
4. **Bootstrap confidence intervals.** Resample the test set with replacement (typically 1000 iterations, `random_state=SEED`), recompute the metric on each. Report the 2.5th–97.5th percentile.
5. **Plot diagnostics:**
   - **Regression:** residuals vs prediction (look for funnel = heteroscedasticity), residuals vs each numeric feature (look for missed nonlinearity).
   - **Classification (binary):** confusion matrix at the chosen threshold, ROC curve, PR curve, **calibration curve** (predicted probability vs observed frequency, in 10 bins). Hand to `plotting-graphs`.
6. **Segment breakouts.** Compute the primary metric for each business-relevant slice. Flag any slice whose metric is > 1 stddev worse than the overall — that's a fairness or robustness issue.
7. **Compare to the previous model**, if any. Same test set, same metric. Improvement isn't real if it's within the bootstrap CI.
8. **Honest summary.** State the metric, the CI, the baseline gap, the worst slice, and a go/no-go recommendation. **Never** report the metric without the baseline.

## Big-data / safety guards

- **Do not retune on the test set.** If you peek at test performance and rerun training, you've contaminated the test set; build a new holdout.
- **Set `random_state=SEED`** for the bootstrap and any stochastic step.
- **Don't average across slices to mask a worst case.** Report the worst slice explicitly; lying-by-aggregation is worse than admitting the gap.

## Worked example

> User: "Evaluate the second-order-within-30d classifier on the held-out test set."

1. Question is binary classification, base rate ~22% (mildly imbalanced). Pick **PR-AUC + precision/recall at threshold 0.5** as primary; report ROC-AUC for ranking.
2. Test set: 50K customers from `first_order_date` between 2024-10-01 and 2024-10-31 (window after train cutoff).
3. **Baseline 1**: predict the train majority class (no repeat). Test accuracy 78% (= 1 − base rate). PR-AUC ≈ 0.22 (= base rate). Sets the floor.
4. **Baseline 2**: predict the train base rate (0.22 for everyone). PR-AUC same ≈ 0.22; ROC-AUC = 0.5; log-loss = 0.51.
5. **Model:**
   - PR-AUC = 0.41 (95% bootstrap CI [0.39, 0.43]). Beats baseline by 0.19 — meaningful.
   - ROC-AUC = 0.74 (CI [0.72, 0.75]).
   - At threshold 0.5: precision = 0.51, recall = 0.34, F1 = 0.41.
   - Log-loss = 0.43.
6. **Calibration:** predicted-vs-observed in 10 bins is close to the 45° line; max bin deviation < 0.04. Acceptable.
7. **Segment breakouts** (PR-AUC):
   - Web: 0.42 — overall.
   - iOS: 0.45 — best.
   - Android: 0.34 — **worse than overall by 0.07**, flag.
   - First-touch organic: 0.46.
   - First-touch paid_social: 0.31 — **worse than overall by 0.10**, flag.
8. **Summary:**
   > Model beats both naive baselines (PR-AUC 0.41 vs 0.22, +0.19; CI does not overlap baseline). Calibration is acceptable (<4 pt max bin deviation). However, performance is meaningfully worse on Android (0.34) and paid_social cohort (0.31). Recommend: investigate why before shipping; do not deploy a single threshold across segments.

## What to add to docs/ if missing

- A new metric the team standardizes on (e.g., "we always report PR-AUC for churn classifiers") → add to `docs/metric-definitions.md` under a new "ML metrics" section.
- A confirmed segment-level performance gap → log in `models/<name>/MODEL_CARD.md` under "Known limitations."

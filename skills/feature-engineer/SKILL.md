---
name: feature-engineer
description: Build a feature pipeline for tabular ML — encode, scale, impute, date-feature, assemble with ColumnTransformer, and run a leakage check before fitting anything.
when_to_invoke:
  - "Build features for the X model"
  - "Engineer features for prediction"
  - "Prepare data for training"
  - "Create the feature pipeline"
---

# Feature Engineer

Tabular feature pipelines fail in two ways: **leakage** (model uses information unavailable at prediction time → optimistic offline metrics, broken in prod) and **train/test skew** (different transforms applied → inconsistent predictions). Both are preventable with discipline.

## Inputs

- **Target** column and its definition.
- **Feature candidates**: numeric, categorical, datetime, text.
- **Prediction time**: when the model fires in production. Anything not knowable *at that time* is leakage.
- **Train/test split** definition: random for IID, time-based for forecasting (preferred for retention/churn/conversion).

## Outputs

- A `ColumnTransformer` (or equivalent Spark pipeline) fit on train, applied to train and test with the **same** parameters.
- A leakage-check report (target leakage, future leakage, group leakage).
- A short data-card snippet for `models/<model_name>/MODEL_CARD.md`.

## Procedure

1. **Lock the prediction time.** Write down: "Model predicts at time T using features available *strictly before* T." Every feature gets evaluated against this rule.
2. **Categorical encoding:**
   - **Low cardinality (≤ 20):** one-hot.
   - **High cardinality (> 20):** target encoding **with cross-validation folds** (no fold leakage) or frequency encoding. Never raw-target-encoding on the full train set — that's leakage.
   - **Ordinal** (`xs/s/m/l/xl`, `bronze/silver/gold/vip`): integer-encoded with the explicit order, not lexical.
3. **Numeric scaling:**
   - Trees (`xgboost`, `lightgbm`, RF): scaling not required.
   - Linear / NN / distance-based: standardize (`StandardScaler`) or robust-scale if heavy-tailed.
4. **Missing values:**
   - **By column type:** numeric → median (with a `<col>_was_null` indicator); categorical → "missing" as its own category; datetime → forward-fill if a sequence, else median.
   - **Don't impute the target.** If the target is missing for some rows, drop those rows (or the model decision is post-hoc).
5. **Date features:**
   - Extract: `dayofweek`, `month`, `hour`, `is_weekend`, `is_holiday` (from a holiday calendar table).
   - **Cyclical encode** for cyclic features: `sin(2π * x / period)`, `cos(2π * x / period)`. Don't use raw integer day-of-week (Sunday=0 ≠ Monday=6 in distance terms).
   - **Time since reference**: `days_since_signup`, `days_since_last_order`. These are leakage candidates — verify they use only data ≤ prediction time.
6. **Aggregations / window features:**
   - "Average order value last 90 days" — must use orders strictly *before* the prediction time. Easy to get wrong with `BETWEEN start AND end` if `end` includes the target observation.
   - For Spark: window functions partitioned by entity, ordered by time, with a *rangeBetween* explicitly bounded to *before* the prediction event.
7. **Assemble with `ColumnTransformer`** (or Spark `Pipeline`) so train and test transformations are identical and parameters are fit on train only.
8. **Leakage checks** (the most important step):
   - **Target leakage:** any feature that is a deterministic or near-deterministic function of the target. Check: train-set correlation of each feature with target. > 0.95 → suspect.
   - **Future leakage:** for any time-derived feature, log the latest source row's timestamp and confirm < prediction_time on a sample.
   - **Group leakage:** if records are grouped (e.g., per customer), verify train and test don't share groups (use `GroupKFold` or time-based split).
   - **Train/test skew:** apply the *same* fit pipeline to test; verify column count and order match.
9. **Save the fitted pipeline** as a serialized object (`joblib`) under `models/<name>/feature_pipeline.joblib`. Pin `scikit-learn` and `pandas` versions in the model card.

## Big-data / safety guards

- For Spark feature pipelines, use `pyspark.ml.Pipeline` — don't `.toPandas()` a 100M-row training set unless you're sampling.
- **Set `random_state=SEED`** (per `docs/environment.md`) on every transformer with stochasticity (e.g., `TargetEncoder` with CV).
- **PII**: customer features should be derived (count of orders, days since signup) — not raw `email` or `customer_id`. The model should not be able to memorize individuals. See `docs/data-classification.md`.

## Worked example

> User: "Build features for a model that predicts whether a customer will place a second order within 30 days of their first."

1. **Prediction time:** the day of the customer's first completed order. Features must use only data ≤ that date.
2. **Target:** `did_repeat_within_30d` (1 if a second completed order in the next 30 days).
3. **Feature candidates:**
   - `first_touch_channel` (categorical, low cardinality) → one-hot.
   - `region` (categorical, low cardinality) → one-hot.
   - `signup_surface` (categorical, low cardinality) → one-hot.
   - `days_signup_to_first_order` = `first_order_date - signup_ts::date` → numeric. Verify only uses data ≤ first_order_date.
   - `first_order_subtotal` (numeric) → standard scale.
   - `first_order_item_count` (numeric, from `fct_order_items`) → standard scale.
   - `first_order_dayofweek` → cyclical encode.
   - `discount_codes_used` (boolean: at least one code on first order) → 0/1.
4. **Excluded** as leakage: `lifetime_value`, `lifetime_orders`, `lifecycle_stage = 'repeat_buyer'`. All include post-first-order information.
5. Fit `ColumnTransformer` on train, apply to test. Confirm column count matches.
6. Leakage check: max train-set Pearson correlation of any feature with target = 0.18 (`days_signup_to_first_order`). Plausible — repeat-likelihood does correlate with engaged signups. No leakage red flag.
7. Save pipeline. Add a paragraph to the model card: features, exclusions, leakage check.

## What to add to docs/ if missing

- A new feature pattern that's reused across models (e.g., "RFM features over `fct_orders`") → consider a `docs/feature-conventions.md` (not in default repo) or expand this skill's procedure.
- A holiday calendar table → flag for Data Eng in `docs/data-owners.md`; until it exists, use a Python list and mark it as a TODO.

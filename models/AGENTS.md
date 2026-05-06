# models/AGENTS.md

This folder is for **trained ML models** that this team owns. One folder per model. Each model has a model card, training code, evaluation, and the artifact (or a reference to where the artifact lives, since artifacts are typically too large to commit).

## Folder structure

```
models/
├── AGENTS.md
└── <model-slug>/
    ├── MODEL_CARD.md         (purpose, data, performance, limitations, owner)
    ├── train.py              (training script — reproducible from scratch)
    ├── evaluate.py           (evaluation script)
    ├── feature_pipeline.py   (or notebook — the ColumnTransformer / Spark Pipeline)
    ├── predict.py            (inference script — what production calls)
    ├── tests/                (unit tests on the feature pipeline at minimum)
    ├── artifacts/            (not committed — models live in object storage / model registry)
    └── README.md             (one-page overview, links to MODEL_CARD)
```

## Conventions

1. **Model card is required.** Use the lightweight template in this file (see "Model card" section below). A model without a card is not shippable.
2. **Reproducibility.** `train.py` should reproduce the model end-to-end from a frozen training-data snapshot. Pin all dependencies per `docs/environment.md`. Set the seed per `docs/environment.md`.
3. **Held-out test set is sacred.** It is not used in training, hyperparameter selection, or threshold setting. If you peeked at it, build a new one.
4. **Feature pipeline is the model.** A trained model + its feature pipeline travel together. Serialized separately, loaded together.
5. **No PII in training.** Customer features are derived (counts, recencies, days_since_X), not raw identifiers (`email`, `customer_id`). See `docs/data-classification.md`.
6. **Baseline first.** Every model card reports the naive baseline alongside the model. "X% accuracy" without a baseline is meaningless.

## Model card (lightweight)

Every `MODEL_CARD.md` answers, in this order:

1. **Purpose.** What does the model predict? Who uses the prediction, for what decision?
2. **Data.** Training-set window, test-set window, features (with classes from `docs/data-classification.md`), target definition.
3. **Performance.** Primary metric + CI. Naive baseline. Segment breakouts. Calibration curve summary.
4. **Limitations.** Known performance gaps (worst slice). Distribution drift assumptions. When the model would need retraining.
5. **Operating point.** If a threshold is used, what is it and why. Tradeoffs at the operating point (precision/recall).
6. **Owner.** Person + Slack channel. Retraining schedule. Monitoring approach.
7. **Change log.** Version bumps with what changed.

## Training

1. Run `analysis-planner` first if the use case is fuzzy. **Modeling without a clear decision use case is a science project, not a product.**
2. Run `feature-engineer` skill to build the pipeline. **Leakage check is non-negotiable.**
3. Train. Set the seed.
4. Run `model-evaluator` skill on the held-out test set with bootstrap CIs and segment breakouts.
5. Decide: ship / don't ship / iterate. Document the decision in the model card.

## Inference

- `predict.py` exposes a stable function signature. Don't change it without versioning.
- Inference logs every prediction's input features and prediction (sampled if volume is high) for monitoring.
- Predictions on customer-level data are governed by `docs/data-classification.md` — predictions can be PII if joined with identity.

## Monitoring

- **Distribution drift**: weekly check on input feature distributions vs training distribution. Alert on > N% PSI.
- **Performance drift**: when ground truth lands (often delayed by the prediction window), recompute the primary metric on the new window. Compare to the model card's claim. Alert on > 1× CI degradation.
- **Calibration drift**: same idea, on the calibration curve.

## Deprecation

- A model is deprecated when (a) it's been replaced, (b) the use case ended, or (c) performance has drifted below acceptable.
- Mark `**[DEPRECATED YYYY-MM-DD]**` in the model card. Don't delete — keep history.
- Stop the inference job (per `pipelines/AGENTS.md` if applicable).
- Move artifacts out of active storage; keep one copy in cold storage with the model card.

## Hand-offs

- **To Production**: serving infra is out of scope for this repo. Hand off to the platform team with the model card + serialized model + feature pipeline.
- **To DS peer for review**: `notebook-reviewer` skill + the model card. Reviewer's checklist: leakage, baseline comparison, calibration, segment performance.

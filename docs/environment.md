# Environment

The reproducibility contract. **Run anything stochastic without checking this and you'll be wrong, just not yet.**

## Versions

- **Python:** 3.12 (pinned; do not bump without team review).
- **Package manager:** `uv` ([astral-sh/uv](https://github.com/astral-sh/uv)). Not pip, not conda.
  - Install: `curl -LsSf https://astral.sh/uv/install.sh | sh`
  - Sync: `uv sync`
  - Add a dep: `uv add <pkg>`
- **Spark:** Spark 3.5.x via `databricks-connect` (configured per-developer; see your warehouse setup).
- **Jupyter:** `ipykernel` registered as `data-science-agent-toolkit-py312` (auto-registered by `uv sync`).

## Random seeds

Every stochastic step needs a fixed seed. Set it in the **setup cell** of every notebook and at the top of every script.

```python
import os
import random
import numpy as np

SEED = int(os.environ.get("RANDOM_SEED", 42))

random.seed(SEED)
np.random.seed(SEED)

# scikit-learn:
# pass random_state=SEED to every estimator and split function

# PyTorch:
# import torch
# torch.manual_seed(SEED)
# torch.cuda.manual_seed_all(SEED)
# torch.backends.cudnn.deterministic = True
# torch.backends.cudnn.benchmark = False

# Spark sampling: pass seed=SEED
# df.sample(fraction=0.01, seed=SEED)
```

**Where this matters:**

- Train/test splits — never accept the default `random_state=None`.
- Sampling for EDA — `df.sample(fraction=0.01, seed=SEED)`.
- Bootstrap CIs — `np.random.default_rng(SEED)`.
- ML model init — `RandomForestClassifier(random_state=SEED)`, etc.
- Cross-validation splits — `KFold(n_splits=5, shuffle=True, random_state=SEED)`.
- Anything else that calls a PRNG.

## Time zone

- **All event timestamps in tables are UTC.** This is non-negotiable.
- **All scheduled jobs run in UTC.**
- **Reporting timestamps in deliverables default to UTC unless the audience asked otherwise.** When in doubt, ask.

```python
from datetime import datetime, timezone
now = datetime.now(timezone.utc)  # always tz-aware
```

Never use `datetime.now()` without `timezone.utc` — naïve datetimes are a bug factory.

## Determinism gotchas (Spark)

- **`COLLECT_LIST`, `FIRST`, `LAST`** without `ORDER BY` are non-deterministic. If the result matters, sort first.
- **Window functions** with ties in `ORDER BY` are non-deterministic for the tied rows. Add a tie-breaker (`ORDER BY x, primary_key`).
- **`UNION` (without `ALL`)** sorts; `UNION ALL` does not. Use `UNION ALL` and an explicit `ORDER BY` if order matters.
- **Floating-point sums** are order-dependent in distributed compute. For finance-critical sums, cast to `DECIMAL` early.

## Fixtures

Small reference data lives in `data/fixtures/` (committed). Loading pattern:

```python
import pandas as pd
df_ref = pd.read_csv("data/fixtures/<your-fixture>.csv")
```

Fixtures are okay to commit if:

- Small (< 100 KB)
- Non-sensitive (no PII, no business secrets)
- Static (regenerating on every run would be wasteful)

Otherwise, pull from a table or external source at runtime.

## Environment variables

See `.env.example` for the full list. Load via `python-dotenv`:

```python
from dotenv import dotenv_values
config = dotenv_values(".env")  # does NOT pollute os.environ
```

We use `dotenv_values` rather than `load_dotenv` so the `.env` doesn't leak into subprocesses or get printed in tracebacks.

## Reproducibility checklist (before claiming "done")

- [ ] Seed set, applied to every PRNG path.
- [ ] All timestamps tz-aware UTC.
- [ ] No floats summed without a deterministic ordering when precision matters.
- [ ] Notebook executes top-to-bottom in a fresh kernel (`Restart & Run All`).
- [ ] Output files (figures, csvs) regenerate identically when re-run.
- [ ] Versions of any non-vendored deps captured in `uv.lock` (committed).

If a re-run produces different numbers, one of these failed. Don't ship until you find which.

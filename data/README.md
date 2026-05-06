# data/

Local data storage. **Most things here are not committed.** This folder exists so paths in code resolve consistently across developers, not as a place to commit data.

## What's allowed in here, committed

- `fixtures/` — small (< 100KB), non-sensitive, static reference data (e.g., a country-code lookup, a holiday calendar). Loading pattern in `docs/environment.md`.
- `sample_*.csv` — tiny, scrubbed sample files used by tests. Must not contain PII or Confidential data per `docs/data-classification.md`. Pre-cleared with `pii-scrubber` skill before commit.

## What's NOT allowed in here, committed

- **Raw warehouse extracts** of any kind. Even "I just dumped 100 rows for testing." If you need warehouse data, query it at runtime.
- **Any file > 100 KB** — likely too much for this folder's purpose; reconsider.
- **`.parquet`, `.feather`, `.arrow`, `.csv` exports of analytics tables** — keep these out of version control. If one slips through, `git rm` it before merging.
- **Any column from the PII or Confidential class** in `docs/data-classification.md`.
- **Personal scratch data** (your own usage, your own customer history). Even if technically you're allowed to query yourself, that data isn't a fixture.

## Folder structure (when fully populated locally)

```
data/
├── README.md          (this file — committed)
├── fixtures/          (committed: small static reference data)
├── raw/               (local only: untransformed pulls from warehouse)
├── interim/           (local only: half-processed intermediates)
├── processed/         (local only: final shapes used by analyses/models)
└── external/          (local only: third-party data — alt-data, vendor exports)
```

The non-fixture folders may not exist on disk until a developer runs an analysis that produces them. Keep them out of version control so they stay local.

## Pulling data locally

1. **Query the warehouse.** That's the source of truth.
2. **Cache locally** to `data/raw/<topic>/<date>.parquet` if you'll be iterating on the analysis without changing the source slice — saves re-querying and warehouse credits.
3. **Document the cache** at the top of the notebook ("This notebook reads from `data/raw/d7-retention/2025-01-15.parquet`, refreshed by re-running cell 3.").
4. **Refresh discipline.** Don't trust a cache that's been on disk for more than the source's freshness SLA. Default: re-query if the cache is older than 24h.
5. **Delete on completion.** When the analysis ships, the cache should be deleted (or at least eligible for deletion). Don't accumulate.

## Handling a leak

If a file with PII or Confidential data lands in `data/` and gets staged for commit:

1. **Don't commit.** `git restore --staged data/<file>`.
2. If already committed but not pushed: rewrite history (`git commit --amend` if it's the latest, or `git rebase` for older). Verify with `git log --stat`.
3. If already pushed to a remote: notify `@privacy` per `docs/data-classification.md` and follow their incident process.

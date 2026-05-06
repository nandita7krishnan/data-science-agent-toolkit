# Data Classification

Every column belongs to one of four classes. **Before pasting any query output into Slack, email, a notebook published outside the team, or a screenshot, run `skills/pii-scrubber/SKILL.md`.**

> Forking? Replace specifics with your org's classification scheme. Most schemes have 3-4 tiers; map them onto the structure below.

## The four classes

| Class | What it is | Where it can go |
|---|---|---|
| **PII** | Personally identifiable information about a customer or employee | Production storage only. Never in screenshots, decks, Slack, email, or any notebook shared outside the DS team without scrubbing. |
| **Confidential** | Business secrets — unreleased pricing, exec compensation, M&A discussions, undisclosed financials | Internal stakeholders with a need-to-know. Never external. |
| **Internal** | Default. Operational data about the business that isn't a secret but isn't public | Anyone with an ABC Business account. Don't post externally. |
| **Public** | Already disclosed externally — public-facing prices, marketing copy, published metrics in earnings | Anywhere |

## Classification table

The columns most likely to bite you. Not exhaustive — when in doubt, **classify up** (treat as the stricter class).

### PII — must be scrubbed before any external surface

| Column | Where it appears | Why |
|---|---|---|
| `email` | `dim_customers`, `stg_marketing__campaigns` | Direct identifier |
| `phone_number` | `dim_customers`, `dim_customer_addresses` | Direct identifier |
| `first_name`, `last_name` | `dim_customers` | Direct identifier (in combination) |
| `street_address`, `street_address_2` | `dim_customer_addresses` | Direct identifier (precise location) |
| `postal_code` | `dim_customer_addresses` | Quasi-identifier; PII when combined with date-of-birth or rare attribute |
| `date_of_birth` | `dim_customers` | Quasi-identifier; combine with postal_code or last name → PII |
| `ip_address` | `fct_page_events`, server logs | Quasi-identifier per GDPR |
| `device_id` | `fct_page_events.device_id` | Pseudonymous identifier; PII per CCPA |
| `customer_id` (ABC Business internal) | Everywhere | Pseudonymous; PII when joined with a direct identifier |
| `payment_card_*` (any) | None — should not be in any analytics table | PCI scope; if you see it, treat as an incident |

### Confidential — internal-only, need-to-know

| Column / area |
|---|
| `dim_customers.lifetime_value` (when joined to identifiable customer) |
| `fct_orders.cost_of_goods_sold`, `gross_margin` |
| Any column related to unreleased products (look for `is_unannounced = TRUE` flags) |
| Pre-public earnings figures — anything within the financial blackout window |
| Personnel data (`dim_employees`, comp tables) — not in scope for this repo, but if you encounter it: stop |

### Internal — default

Everything in `analytics_prod.core` not flagged above. Order counts, page-view counts, conversion rates, etc.

### Public

- abcbusiness.example product prices visible on the storefront.
- Metrics that have already appeared in published earnings or press releases. (Verify the "already published" part — pre-earnings is Confidential.)

## Per-class transformations

When sharing data, transform PII to the lowest class that satisfies the use case.

| Original | Aggregate (best) | Hash | Mask | Drop |
|---|---|---|---|---|
| `email = "alice@example.com"` | `count(distinct email) = 1247` | `sha256("alice@example.com") = "a3f8..."` | `a***@e***.com` | column dropped |
| `customer_id = "cus_abc123"` | `count(distinct customer_id)` | `sha256("cus_abc123" + project_salt)` | `cus_******23` | dropped |
| `ip_address = "192.0.2.42"` | `count(distinct ip_address)` | `sha256(ip + salt)` | `192.0.2.0/24` (subnet) | dropped |
| `postal_code = "94103"` | `count by postal_code` | n/a | `941XX` (truncate) | dropped |
| `date_of_birth = "1985-04-12"` | bucket to `age_band = '36-40'` | n/a | `1985-XX-XX` | dropped |

**Aggregation is preferred over hashing.** Hashes are reversible with a known salt or rainbow-table attack on small spaces. Aggregation is irreversible.

**Minimum bucket size for aggregation:** 25 rows. Reporting "1 customer in postal code 12345 spent $50K" defeats the point. If a bucket has < 25 rows, suppress it (`<25`) or merge into a larger bucket.

## Decision flow

```
Output contains a column from the PII table above?
├── YES → run pii-scrubber skill
│         ├── Can you aggregate? → aggregate, drop the raw column.
│         ├── Need row-level? → hash with salt, document in deliverable footer.
│         └── Need to share externally? → don't. Talk to @privacy first.
└── NO  → check Confidential table; same flow.
        └── If neither → Internal. Safe to share with @abc-business accounts.
```

## When you spot a violation

- **In committed code:** Don't fix-then-commit silently. Open a separate PR with the fix and notify `@privacy`.
- **In a chat message you sent:** Delete it (most platforms allow this), notify `@privacy` describing what was sent and to whom. Speed beats embarrassment.
- **In a shared dashboard:** Restrict access immediately, then notify `@privacy`.

## Audit trail

If a deliverable contains hashed or masked PII, **document it in the deliverable's footer**:

> Footer example: "Customer identifiers SHA-256 hashed with a project-scoped salt (not included here). PII columns (email, phone, address) excluded prior to aggregation."

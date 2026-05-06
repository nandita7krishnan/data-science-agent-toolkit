# Glossary

Product codes, acronyms, and jargon used in this repo. **First place to look when a term is unclear.**

> Forking? Replace ABC Business-specific terms with your own. Keep the structure.

## Company / product

| Term | Definition |
|---|---|
| **ABC Business** | The fictional online retailer this repo is built around. Replace with your company. |
| **Storefront** | The ABC Business website (web + mobile web). Distinct from the iOS/Android apps. |
| **App** | The ABC Business native mobile apps (iOS, Android). Distinct from Storefront. |
| **Merchant catalog** | The set of products ABC Business has listed for sale. |
| **SKU** | Stock-keeping unit — a uniquely identifiable product variant (size, color, etc.). One product can have many SKUs. |

## Customer lifecycle

| Term | Definition |
|---|---|
| **Visitor** | An unidentified user (no `customer_id`) browsing the site. Tracked by `anonymous_id`. |
| **Customer** | An identified user with an ABC Business account (`customer_id`). |
| **First-time buyer (FTB)** | A customer whose first order completed within the analysis window. |
| **Repeat buyer** | A customer with ≥ 2 completed orders, lifetime. |
| **Churned customer** | No completed order in the trailing 180 days. See `metric-definitions.md` for the exact formula. |
| **Reactivated customer** | A previously-churned customer who placed an order in the analysis window. |

## Engagement / events

| Term | Definition |
|---|---|
| **DAU / WAU / MAU** | Daily / Weekly / Monthly Active Users. Definition: distinct customers with ≥ 1 qualifying event in the window. See `metric-definitions.md` for the exact event filter. |
| **Session** | A continuous block of activity from one device. Sessions end after 30 min of inactivity. |
| **Page event** | Any tracked event in `fct_page_events` (page_view, product_view, add_to_cart, begin_checkout, purchase). |
| **Funnel** | An ordered sequence of events the team measures conversion through (typical: `product_view → add_to_cart → begin_checkout → purchase`). |

## Commerce / orders

| Term | Definition |
|---|---|
| **GMV** | Gross Merchandise Value. Sum of `order_subtotal` for orders in `status = 'completed'`. **Excludes** taxes and shipping per definition (see `metric-definitions.md`). |
| **AOV** | Average Order Value. `GMV / count(completed orders)`. |
| **Net revenue** | GMV minus refunds, returns, and chargebacks. Reconciles with finance ledger. |
| **Cart abandonment** | Sessions where `add_to_cart` fired but no `purchase` followed within the same session. |
| **Refund window** | 30 days from order date. Refunds beyond this require manual approval. |

## Data engineering

| Term | Definition |
|---|---|
| **`stg_*`** | Staging table. One row per source row, light cleaning only. Owned by the team that produces the source data. |
| **`dim_*`** | Dimension table. Slowly-changing entity (customer, product, store). Often SCD Type 2. |
| **`fct_*`** | Fact table. One row per event or transaction, immutable once written. |
| **`int_*`** | Intermediate table. Logical reshaping that's reused across multiple downstream models. Not surfaced to consumers. |
| **SCD Type 2** | Slowly Changing Dimension Type 2 — keeps history with `valid_from` / `valid_to` / `is_current` columns. |
| **Grain** | What one row of a table represents. *Always* state the grain when documenting or querying a table. |
| **Partition column** | The column the table is physically partitioned by (usually a date). Filtering on the partition column is the difference between a $0.01 query and a $1,000 query. |
| **Late-arriving data** | Rows that land in the table after the partition they belong to has already been "closed." See `pipelines.md` for the watermark policy. |

## Roles / teams

| Term | Definition |
|---|---|
| **DS** | Data Scientist. Primary consumer of `dim_*` / `fct_*`. |
| **DE / AE** | Data Engineer / Analytics Engineer. Owns the pipelines that produce the tables in `docs/schema/`. |
| **PM** | Product Manager. Common stakeholder for analyses and experiments. |
| **Marketing analytics** | The team that owns `stg_marketing__*` tables (campaigns, attribution). See `docs/external-tables/`. |
| **Finance** | Source of truth for revenue. Anything quoted as "revenue" in an exec deck must reconcile with finance. |

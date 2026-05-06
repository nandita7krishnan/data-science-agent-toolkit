# Business Info

Business context an AI agent needs to write correct queries and pipelines for **ABC Business**. Update when the business changes.

> Forking? Replace the ABC Business-specific content; keep the section structure.

## What ABC Business is

ABC Business is an online retailer selling physical goods (apparel, home, electronics) direct-to-consumer in **US, Canada, UK, and Australia**. Web (Storefront) and native mobile apps (iOS, Android). No B2B, no marketplace sellers — every SKU is shipped from ABC Business inventory.

## Customer segments

The team segments customers along three axes. Use the canonical names below in any analysis.

| Axis | Values | Where to find it |
|---|---|---|
| **Lifecycle stage** | `prospect`, `first_time_buyer`, `repeat_buyer`, `vip`, `at_risk`, `churned`, `reactivated` | `dim_customers.lifecycle_stage` (recomputed daily) |
| **Acquisition channel** | `paid_search`, `paid_social`, `organic`, `email`, `affiliate`, `referral`, `direct` | `dim_customers.first_touch_channel` (immutable) |
| **Region** | `us`, `ca`, `uk`, `au` | `dim_customers.region` (set at first order, immutable) |

**VIP** = top 5% of customers by trailing-365-day net revenue, recomputed weekly. Definition lives in `pipelines.md` (job: `dim_customers_lifecycle_refresh`).

## Order statuses

Every order in `fct_orders` has one of these statuses. Filter on `status` deliberately — most metrics want only `completed`.

| Status | Meaning | Counts toward GMV? |
|---|---|---|
| `pending` | Payment not yet captured | No |
| `completed` | Payment captured, shipped or pre-shipping | Yes |
| `cancelled` | Customer or system cancelled before fulfillment | No |
| `refunded` | Fully refunded after `completed` | Yes (then subtracted in net revenue) |
| `partially_refunded` | Partial refund issued | Yes (with partial subtraction) |
| `chargeback` | Card chargeback — fraud/dispute | Yes (then subtracted in net revenue) |

## Calendar — periods that distort the data

Aggressively important for trend analyses. **Always check whether the analysis window overlaps these.**

| Period | Approx. dates | Effect |
|---|---|---|
| **Black Friday / Cyber Monday (BFCM)** | Last Friday of Nov → following Monday | Order volume 4–6× normal; AOV drops 10–20% (deal-driven). Returns spike late December. |
| **Holiday peak** | Nov 20 – Dec 24 | Sustained 2–3× normal volume. Shipping cutoffs change conversion behavior in last 5 days. |
| **Post-holiday returns wave** | Dec 26 – Jan 15 | Refunds 3–4× baseline. Net-revenue trend looks worse than reality if you use order-date attribution. |
| **Memorial Day / July 4 / Labor Day promos** | Standard US retail dates | 1.5–2× volume, modest AOV impact. |
| **Prime Day adjacency** | Mid-July | ABC Business runs counter-promos; conversion-rate dips 5–10% during Amazon's event window. |

## Mobile vs web

| Surface | DAU share | Conversion rate (rough) | Notes |
|---|---|---|---|
| Storefront (web) | ~55% | ~2.5% | Higher AOV; baseline for cohort analyses. |
| iOS app | ~30% | ~4.5% | Highest engagement; push notifications drive day-of activity. |
| Android app | ~15% | ~3.0% | Geography-biased toward UK/AU. |

**Don't aggregate across surfaces** for engagement metrics without thinking about the mix. Surface-level breakouts are usually expected.

## Marketing context

- **Attribution model:** Last-touch on `fct_page_events.utm_source` (default). For multi-touch, use `analytics_prod.core.fct_attribution` — owned by Marketing Analytics; see `docs/data-owners.md` for contacts. (Schema doc not in this kit's worked examples; document with `schema-documenter` skill before first use.)
- **Email is owned by Lifecycle Marketing**, not the central marketing team. Their campaigns appear under `utm_source = 'email'` with `utm_medium IN ('newsletter','triggered','transactional')`.
- **Transactional emails** (order confirmations, shipping) should be excluded from any "marketing influence" analysis: `utm_medium != 'transactional'`.

## Org-relevant context

- **DS reports to the Product org**, not Engineering. Stakeholder asks usually come from PMs.
- **The CFO reads the weekly business review.** Any number that goes there must reconcile with finance — use `net revenue`, not GMV.
- **Experimentation Council** reviews any A/B test before launch. Write-ups go in `experiments/` and follow the template in `experiments/AGENTS.md`.

## What you should always ask before running a "company-wide" number

1. Which surfaces? (web only, app only, or all)
2. Which regions? (us only, or worldwide)
3. Which order statuses? (completed only, or completed + partially refunded)
4. Internal traffic excluded? (default yes; confirm)
5. Time zone? (default UTC; surface-level analyses sometimes want local)
6. Is the window spanning a calendar period above? If yes, call it out in the deliverable.

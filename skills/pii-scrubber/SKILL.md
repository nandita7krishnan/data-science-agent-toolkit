---
name: pii-scrubber
description: Detect and mask PII in query output, logs, or notebook cells before sharing — applies the right transformation per data class from docs/data-classification.md.
when_to_invoke:
  - "Scrub PII from this output"
  - "Is it safe to share this?"
  - Always before pasting any query output to Slack, email, screenshot, dashboard, or external doc that *may* contain PII.
  - The user is about to share customer-level data outside the DS team.
---

# PII Scrubber

This is a **safety skill.** Run it whenever output crosses a trust boundary — DS team → other teams, internal → external, code → screenshot.

## Inputs

- The output to scrub (DataFrame, query result, log, text snippet).
- **Destination** of the output (Slack channel, email recipients, dashboard, deck, external blog) — drives the strictness level.

## Outputs

- A scrubbed copy with PII transformed (aggregated / hashed / masked / dropped) per `docs/data-classification.md`.
- An audit footer noting what was transformed and how.

## Procedure

1. **Read `docs/data-classification.md`** for the canonical PII column list and per-class transformation rules.
2. **Identify all candidate columns.** Both literal PII columns and quasi-PII (`customer_id`, `anonymous_id`, `device_id`, `ip_address`, `postal_code`, `date_of_birth`).
3. **Choose transformation by use case** (aggregation > hashing > masking > dropping). Strongest available wins:
   - **Can the question be answered by aggregates only?** Replace customer-level columns with counts / rates. Best.
   - **Need row-level structure but not identity?** Hash with a project salt. Good if the salt is documented.
   - **Need to share format but not value?** Mask (`a***@e***.com`, `192.0.2.0/24`).
   - **Don't need it at all?** Drop the column.
4. **Apply minimum bucket size** (default 25) when aggregating: any group with < 25 rows is suppressed (`<25`) or merged into a larger group. **A "1 customer in postal code 12345 with $50K LTV" defeats aggregation.**
5. **Check Confidential columns** alongside PII. `cost_of_goods_sold`, `gross_margin`, `lifetime_value` are **not PII** but are restricted (per `docs/data-classification.md`). For external destinations, treat the same way.
6. **Re-scan the output** after transformation. Did any free-text column (`comment`, `feedback`, `properties.message`) sneak through? Free text is high-risk — drop it unless the use case requires it.
7. **Add the audit footer** to the output:

```
> PII transformed before sharing. Customer identifiers SHA-256 hashed
> with a project-scoped salt (not included here). Email/phone/address columns dropped.
> Aggregations suppressed for groups with n < 25.
```

8. **Confirm destination strictness.** External deck → must be Public class only. Internal Slack with a finance partner → Internal class okay. Private DM with a peer → DS-team-Internal okay. **When in doubt, level up.**

## Big-data / safety guards

- **Treat hashes as quasi-identifiers.** A hash + 3 other quasi-attributes can still reidentify. Aggregation > hashing whenever feasible.
- **Free text is the silent killer.** Free-text fields can contain emails, phone numbers, even SSNs that customers typed in. Default to dropping free text.
- **Salts must be project-scoped, not global.** Reusing a salt across projects creates linkability.
- **Never** include the salt in the output footer. Reference its name only.

## Worked example

> User: "I want to post these top 20 customers by LTV to the marketing channel — they're our VIPs."

1. Read `docs/data-classification.md`. Columns at issue: `customer_id` (PII), `email` (PII), `lifetime_value` (Confidential), `lifetime_orders` (Internal).
2. Destination = `#marketing` (cross-team Slack). Strictness = at least Internal; PII not allowed without consent + privacy review.
3. **Reframe**: the marketing team likely doesn't need 20 individuals. They need **the segment**. Replace 20 rows with: count of VIPs, mean LTV, median LTV, top decile cutoff. Aggregation handles it.
4. If **truly** row-level (e.g., the marketing team needs the customers' segment IDs to load into a campaign tool), that's a *campaign tool flow*, not a Slack post. Tell the user.

Output for the Slack post:

```
*Top-decile VIPs by trailing-365-day LTV (n=12,344, region=US)*

• Mean LTV: $XXX
• Median LTV: $XXX
• 90th percentile cutoff: $XXX
• Lifecycle distribution: vip 100%, repeat_buyer 0%

> Customer-level identifiers withheld. For activation, sync the VIP cohort
> via the Campaign Tool's audience builder; do not paste customer IDs in
> Slack channels.
```

## What to add to docs/ if missing

- A new column observed to be PII or quasi-PII → add to `docs/data-classification.md` under the appropriate class.
- A new sharing-destination policy (e.g., "no customer-level data in `#marketing` ever") → add to `docs/data-classification.md`.

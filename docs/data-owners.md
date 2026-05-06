# Data Owners

Who owns what, and who to escalate to. **Check this before pinging "data" — there's no such team.**

> Forking? Replace the names below with your real owners. Keep the structure (Domain → Table/area → Owner → Slack → Escalation).

## Quick lookup

| Domain | Tables | Owning team | Slack |
|---|---|---|---|
| **Commerce / orders** | `fct_orders`, `fct_refunds`, `dim_orders_*` | Data Eng — Commerce | `#data-eng-commerce` |
| **Customer / identity** | `dim_customers`, `dim_customer_addresses` | Data Eng — Customer | `#data-eng-customer` |
| **Clickstream / events** | `fct_page_events`, `fct_app_events` | Data Eng — Clickstream | `#data-eng-clickstream` |
| **Catalog / product** | `dim_products`, `dim_skus` | Data Eng — Catalog | `#data-eng-catalog` |
| **Marketing** | `stg_marketing__campaigns`, `fct_attribution` | Marketing Analytics (external) | `#marketing-analytics` |
| **Finance** | `fct_revenue_recognized`, `dim_chart_of_accounts` | Finance Data | `#finance-data` |
| **Experimentation** | `fct_experiment_assignments` | Experimentation Platform | `#experimentation` |

## Role-level contacts (placeholders — replace)

| Role | Person | Slack | Email |
|---|---|---|---|
| **Head of Data** | _< name >_ | `@head-of-data` | data-leadership@abcbusiness.example |
| **DS Manager** | _< name >_ | `@ds-manager` | ds-manager@abcbusiness.example |
| **Data Eng Manager** | _< name >_ | `@de-manager` | de-manager@abcbusiness.example |
| **Analytics Eng Lead** | _< name >_ | `@ae-lead` | ae-lead@abcbusiness.example |
| **Data Platform on-call** | rotates weekly | `@dp-oncall` | dp-oncall@abcbusiness.example |
| **Privacy / Legal** | _< name >_ | `@privacy` | privacy@abcbusiness.example |

## Escalation paths

### Data quality issue (wrong number, missing rows, schema drift)

1. Post in the owning team's channel (table above) with: table name, partition, and what's wrong (with a query).
2. If no response in **2 business hours** during work hours, page the team's on-call (each team has a PagerDuty schedule; check the channel topic).
3. If the issue is impacting an **executive deliverable** (board deck, weekly review), escalate to `@de-manager` immediately — don't wait.

### Pipeline failure / SLA miss

1. The job's owning team gets paged automatically by the orchestrator.
2. As a *consumer*, you'll see a delay banner. If the delay is >2× the SLA, post in the owning channel.
3. Don't try to "fix it yourself" by rerunning — talk to the owner first.

### Schema change request

1. **Don't add columns yourself** to upstream tables. Open a ticket with the owning team.
2. For derived/internal tables (`int_*` in this repo), the rules in `pipelines/AGENTS.md` apply — ping the analytics-engineering author directly.

### PII / privacy concern

1. Anything you suspect violates `docs/data-classification.md` → `@privacy` immediately. Don't wait, don't post the offending content.
2. If you've already shared something problematic externally (Slack, email, doc), say so explicitly — it changes the response.

## When to use which channel

- `#data-eng-<domain>` — questions about specific tables (schema, freshness, history)
- `#data-questions` (cross-team) — "where do I find...", "is there a table for..."
- `#data-incidents` — broadcast for active incidents only. Don't use for slow questions.
- DMs — last resort. Things in DMs aren't searchable for the next person who has the same question.

## RACI for common deliverables

| Deliverable | Responsible | Accountable | Consulted | Informed |
|---|---|---|---|---|
| Ad-hoc analysis | DS | DS Manager | PM (stakeholder) | Product team |
| A/B test write-up | DS | DS Manager | PM, Eng | Experimentation Council |
| Production model | DS + DE | DS Manager | Platform Eng | Privacy |
| New `dim_*`/`fct_*` table | Data Eng | DE Manager | DS (consumer), AE | Data Catalog |
| Weekly business review numbers | DS | Head of Data | Finance | Exec team |

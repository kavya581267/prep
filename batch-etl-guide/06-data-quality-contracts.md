# Data Quality & Contracts — In Depth

## Why quality is not “unit tests only”

Batch pipelines **touch money**: **impressions billed**, **revenue recognized**, **commissions**. A silent **drift** in **join keys** can understate **delivery** for **weeks** before a finance **reconciliation** catches it.

**Quality** is therefore a **layered contract** between **producers** and **consumers**, encoded in **expectations** + **monitoring**, not a **one-time** manual review.

## Levels of checks (what each catches)

### Schema & type contracts

**Non-null** primary keys, `revenue_usd` **numeric**, `country_code` **in** allowed set.

**Tooling** examples: **Great Expectations**, **Deequ**, **dbt tests**, **warehouse** **constraints** (where supported).

**Failure:** **hard stop** publish for **red** tests on **core** facts.

### Row-level rules

**Cross-field logic:** `if event_type='click' then click_ts is not null`.

**Referential:** `campaign_id` exists in `dim_campaign` **or** lands in **quarantine** table with reason code.

### Aggregate / distributional checks

**Row count** for `dt` within **±10%** of **7-day median** unless **known campaign** explains spike.

**Revenue** within **±0.5%** of **ad server** **summary** **extract** (not perfect truth, **directional**).

**Why distributions:** catches **empty partition** bugs (join **exploded wrong** way) where row rules still **pass**.

## Quarantine pattern (deep)

Instead of failing entire **1B** row load for **0.001%** bad rows:

1. Write **good** rows to **`fact_clean`**.  
2. Write **bad** rows to **`fact_quarantine`** with **`reason_code`**, **raw payload ref**.  
3. Alert **data** **SLA** channel with **sample** bad rows.

**Staff:** Quarantine shifts failures from **binary** “all or nothing” to **operational** triage—**finance** may still require **hard fail** on **revenue-affecting** anomalies.

## Contract between teams (beyond tests)

Document in **one page** per critical table:

- **Grain** (what one row **means**).  
- **Timezone & cutoff**.  
- **Keys** uniqueness definition.  
- **Breaking change policy** (notify **N days**, dual-write period).

**dbt** **contracts** (YAML) or **Proto/Avro** for **events** are **machine-checkable** complements.

## Worked example — silent cardinality explosion

**Bug:** join condition **off-by-one** on **`session_id`** → **many-to-many** explosion.

**Row tests** on **single** column **non-null** still pass.  
**Aggregate test:** `sum(impressions)` **3×** vs yesterday → **alarm**.

## Soundbite

> “Quality is **layered**: **schema**, **row rules**, **distribution** checks, and **human** **quarantine** workflows—backed by **cross-team** **contracts** so **schema drift** isn’t discovered by **the CFO**.”

Next: [07-orchestration-observability.md](./07-orchestration-observability.md)

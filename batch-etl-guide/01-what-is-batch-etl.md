# What Batch ETL Is — In Depth

## The plain definition

**Batch ETL** is processing **bounded chunks** of data on a **schedule** (or manual trigger): **Extract** from sources, **Transform** into **clean, conformed, aggregated** shapes, **Load** into systems that serve **analytics, finance, and products**.

**Bounded chunk** usually means “**everything for `dt=2025-03-25` Eastern time**” or “**all changes since watermark `T`**”—you can **draw a box** around what the job owns.

**Streaming** blurs lines with **micro-batch**, but **finance** still often wants a **closed** daily **ledger** with an explicit **cutoff**—that’s cultural and contractual, not purely technical.

## ETL vs ELT (and why both exist)

**Classic ETL** transforms **before** load because:

- The **target** is dumb storage, **or**  
- **Egress** from the source is expensive—you only pull **columns you need**, **or**  
- You must **mask** PII **before** landing in less-controlled zones.

**Modern ELT** loads **raw** or lightly typed data into a **powerful warehouse**, then uses **SQL** (often **dbt**) to transform **inside** the warehouse compute layer.

**Staff decision:** “Where is compute **cheapest** and **governance** strongest for **this** workload?” If your **lake** + **Spark** cluster is already **petabyte-scale**, maybe ETL in Spark then load **aggregates** only. If **Snowflake** credits are **cheaper** than maintaining Spark for a team, ELT wins.

## The layered lakehouse mental model (walkthrough)

Imagine an ads pipeline:

1. **Landing / raw zone** — append-only **JSON** or **Parquet** **events** as produced by ingestion. **Never** “update in place”; you **append** new files with **`ingestion_batch_id`**.

2. **Conformed / cleansed** — **typed** columns, **dedupe** within agreed rules, **foreign keys** repaired or quarantined.

3. **Core** — **star schema** facts and dims: `fact_impressions_daily`, `dim_campaign_scd2`.

4. **Marts** — **team-specific** aggregates tuned for **BI** **grain**.

**Why layers:** you can **replay** transformations from **raw** when logic bugs surface without re-fetching from the **API** edge (sometimes).

## Idempotency is the soul of batch

If Airflow **retries** a task, **or** you **re-run** for `dt=D`, the output for **`dt=D`** must not **double-count** revenue.

**Patterns:** **partition overwrite** for that **`dt`**, **MERGE** **upsert** keyed by **`event_id`**, **insert-only** with **dedupe view** on top.

**Beginner mistake:** “DELETE old rows then INSERT” **without** **transaction** boundaries—**readers** see **empty** **window** mid-job.

## Batch vs streaming in the same company

Common **mature** pattern: **stream** for **fast** **ops** **visibility**; **batch** for **finance** **truth** **with** **reconciliation**.

**Interview:** “I’d never call stream totals **official** until **batch** **reconciles** unless executives explicitly accept **approximation** **SLA**.”

## Worked example — cutoff semantics

**Requirement:** “**Daily revenue** uses **orders** **placed** **before 23:59:59 America/New_York**.”

**Failure mode:** cluster runs in **UTC**; **`dt`** partition is **UTC date**; **some** late Eastern orders land in **wrong** **partition**.

**Fix:** store **`event_ts_utc`** and **`event_date_et`** as **generated** **business** **partition** key used consistently in **batch** and **reporting**.

## Soundbite

> “Batch ETL is how we **freeze** **truth** for a **time window**: layered **raw→core→mart**, **idempotent** **publishes**, and **explicit** **cutoffs** so finance and **streaming** **dashboards** don’t argue past each other.”

Next: [02-extract-patterns.md](./02-extract-patterns.md)

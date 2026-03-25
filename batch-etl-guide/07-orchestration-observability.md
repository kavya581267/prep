# Orchestration & Observability — In Depth

## What orchestrators actually solve

**Airflow / Prefect / Dagster / cloud schedulers** encode **dependencies**: “**Run** **staging load** after **raw** **lands**, **then** **core fact**, **then** **marts**.”

They also manage **retries**, **SLA timers**, **backfills**, and **permissions** boundaries.

**Beginner mistake:** thinking Airflow “**does ETL**”—it **coordinates** **compute** (Spark, SQL, scripts); **data movement** still obeys **physics**.

## Sensors vs time-based schedules

**Cron:** “**6am** daily”—simple but may run **before** **source** export completes.

**Sensor:** waits for **file marker** `EXPORT_READY` or **partition** **freshness** in **catalog**—**ties** pipeline to **upstream** reality.

**Cost:** sensors **poll**; use **reasonable** **poke intervals**.

## Retries and partial writes (critical section)

If task **fails** after writing **half** of `dt=D`, **blind** retry may **duplicate** if **append-only**.

**Idempotent task design:**

1. Write to **staging** location tagged with **`run_id`**.  
2. **Publish** step **atomically** **promotes** staging → **canonical**.

**Orchestrator** retries **only** failed step; **publish** remains safe.

## Observability beyond “task green”

Measure:

- **Freshness:** `now() - max_available_ts` in output table.  
- **Volume:** rows written vs **7d baseline**.  
- **Duration** trend vs **data growth**.  
- **Cost** **\$** per run if attributable.

**Lineage:** OpenLineage / DataHub—**ties** **job failure** to **dashboard** **impact**.

## Runbooks

For each **Sev-impacting** pipeline, **one** page:

- **Owner** + **escalation**.  
- **Safe** **kill** vs **let finish** guidance.  
- **How** to **rerun** `dt=D`.  
- **Known** **failure** modes (**S3 503**, **warehouse** **lock**).

## Soundbite

> “Orchestration encodes **DAG** **dependencies** and **retry** policy; **observability** proves **freshness** and **volume**, not just **task** **state**. **Idempotent** publish makes retries **safe**.”

Next: [08-cost-sla-backfill.md](./08-cost-sla-backfill.md)

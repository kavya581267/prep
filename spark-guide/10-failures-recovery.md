# Failures & Recovery — In Depth

## Task failures and retries

A **task** is the smallest unit scheduled on an executor. Transient failures (**network blip reading S3**, **JVM hiccup**) trigger **retry** up to **`spark.task.maxFailures`** (often **4** in configs).

If all attempts fail, the **stage** fails → **job** fails (unless **stage** is optional—rare).

**Your job:** read **stack traces** in Spark UI or **executor** logs; categorize **OOM** vs **data** issue vs **infra**.

## Executor loss

If an **executor** dies, Spark **will** **reschedule** missing tasks on **healthy** executors.

For **map** stages reading **immutable** Parquet, **recompute** is straightforward.

For **shuffle** stages, the shuffle **map output files** must still be **available** via Spark’s **external shuffle service** or **recomputed** if lost—can be **expensive**.

## Straggler speculation

If `spark.speculation=true`, Spark launches **duplicate** copy of slow task **speculatively**; first to **finish** wins, other **killed**.

**Risk:** If task has **side effect** writes (`INSERT` without idempotency), **duplicate** attempts can **double-write**.

**Staff rule:** Speculation on **read-only** heavy jobs OK; on **writes** require **idempotency** keys or **disable** speculation for that app.

## Stage failures from data skew / OOM

Skew drives **one** task **OOM** or **spill endlessly**. Retry won’t help—**same** partition still **too fat**.

**Fix path:** skew mitigations, **raise** memory temporarily while planning better fix, **reduce** state with **pre-agg**.

## Driver failures

**Batch job:** driver dies → application **dies** → orchestrator (Airflow) **reruns** **job**—assuming **idempotent** **output**.

**Streaming:** use **checkpoint recovery** to restart **query** from **last** **committed** **offsets** (with compatible code).

## Common production root causes (tell a war story)

- **S3** **throttling** / **503** → **exponential backoff** client settings, **prefix** **partitioning** to widen throughput (S3 **sharding** guidance).  
- **Metastore** / **glue** catalog timeouts.  
- **Bad** **input** file **corrupt** Parquet **footer** → **isolate** path and quarantine.  
- **Auth** token **expiry** on long jobs.

## Soundbite

> “I classify failures: **transient infra** (retry), **data skew** (fix plan), **code** (bug). I’m cautious with **speculation** on **non-idempotent** writes.”

Next: [11-interview-questions-by-level.md](./11-interview-questions-by-level.md)

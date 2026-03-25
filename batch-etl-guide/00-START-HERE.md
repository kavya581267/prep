# Batch Processing & ETL — Deep Track (Beginner → Advanced)

These files are written as **explanations**, not bullet decks. Batch ETL is where **business truth** is manufactured: if you get **idempotency**, **time zones**, and **contracts** wrong, dashboards and money **quietly disagree**.

Spark may be **one** engine you use; these patterns apply to **BigQuery SQL**, **Snowflake tasks**, **dbt**, **Hive**, etc.

## Order

| Step | File |
|------|------|
| 1 | [What batch ETL is](./01-what-is-batch-etl.md) |
| 2 | [Extract patterns](./02-extract-patterns.md) |
| 3 | [Transform: idempotency & SCD](./03-transform-idempotency-scd.md) |
| 4 | [Load: warehouse & lakehouse](./04-load-warehouse-lake-merge.md) |
| 5 | [Partitions & files](./05-partitioning-files.md) |
| 6 | [Data quality & contracts](./06-data-quality-contracts.md) |
| 7 | [Orchestration](./07-orchestration-observability.md) |
| 8 | [Cost, SLA, backfill](./08-cost-sla-backfill.md) |
| 9 | [Interview Q&A deep](./09-interview-questions-by-level.md) |

## Companion

- [`../spark-guide/00-START-HERE.md`](../spark-guide/00-START-HERE.md)

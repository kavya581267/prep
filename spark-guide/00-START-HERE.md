# Apache Spark — Beginner → Advanced (Deep Track)

This folder is meant to be **read like a mini-course**, not a checklist. Each file explains **why** things work, **what** breaks in production, and **how** to reason in interviews—with **examples** and **numbers** where helpful.

**If a section feels short,** the intention is still to **combine** files in order; later files assume earlier ones.

## Reading order

| Step | File |
|------|------|
| 1 | [What Spark is & when to use it](./01-what-is-spark-when-to-use.md) |
| 2 | [RDD, DataFrame, Dataset, Spark SQL](./02-rdd-dataframe-dataset-sql.md) |
| 3 | [Lazy evaluation, DAG, lineage](./03-lazy-evaluation-dag-lineage.md) |
| 4 | [Partitions, narrow vs wide, shuffle](./04-partitions-narrow-wide-shuffle.md) |
| 5 | [Joins, broadcast, skew](./05-joins-broadcast-skew.md) |
| 6 | [Catalyst & Tungsten](./06-catalyst-tungsten.md) |
| 7 | [Memory, spill, serialization](./07-memory-spill-serialization.md) |
| 8 | [Tuning executors & AQE](./08-tuning-executors-aqe.md) |
| 9 | [Structured streaming](./09-structured-streaming-overview.md) |
| 10 | [Failures & recovery](./10-failures-recovery.md) |
| 11 | [Interview questions + model answers](./11-interview-questions-by-level.md) |

## Companion

- **Batch ETL** (extract/load/orchestration/quality): [`../batch-etl-guide/00-START-HERE.md`](../batch-etl-guide/00-START-HERE.md)  
- **Kafka + lake** context: [`../staff-deep-dive-guide/04-data-streaming-kafka-spark-lakehouse.md`](../staff-deep-dive-guide/04-data-streaming-kafka-spark-lakehouse.md)

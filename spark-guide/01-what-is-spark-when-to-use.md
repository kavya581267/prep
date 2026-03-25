# What Spark Is & When to Use It — In Depth

## The problem Spark was built to solve

Imagine you have **500 GB** of advertising **impression logs** for one day, stored as **Parquet** files in S3. On your laptop, **pandas** can hold only a **fraction** of that in RAM. Even if you streamed in chunks, **single-threaded** Python would take **many hours** to compute something as simple as **revenue grouped by country and hour**.

**Apache Spark** is a **distributed computing framework** that lets you treat that dataset as **one logical table**, while the work is **split** across **many machines** (executors). Each machine reads **its** share of files, applies **transformations**, and participates in **shuffle-based** operations (like **GROUP BY** or **JOIN**) that require **exchanging data** over the network.

So in one sentence: **Spark is a cluster-scale data-parallel engine with a SQL/DataFrame programming model and a lazy optimizer.**

## The cluster roles (you must internalize this)

When you run a Spark application:

1. **Driver process** — This is where **your** program runs (`spark-submit`, a notebook kernel, or a long-running job). The driver **builds the query plan**, **schedules** stages and tasks, and **collects** metrics. If the driver dies, the **application** is gone (unless you use **cluster mode** patterns where the driver is restarted from a **checkpoint** in streaming—but that’s a special case).

2. **Executors** — JVM processes on worker nodes. Each executor has **CPU cores** and **memory**. Executors run **tasks** (units of work on **one partition** of data). Executors **read** from storage (S3/HDFS), **shuffle** data to each other, and **write** results.

3. **Cluster manager** — YARN, Kubernetes, Mesos, or a **managed** service (EMR, Glue, Databricks) decides **where** executors **physically** land. You usually don’t debug the cluster manager in interviews unless something about **allocation** is broken—but you **do** care that you **requested** enough **cores** and **memory**.

**Staff-level framing:** Spark performance is almost never “Spark is slow”; it’s **I/O**, **shuffle**, **skew**, **GC**, **too many small tasks**, or **bad join plans**—all of which show up in the **Spark UI**.

## Jobs, stages, tasks (the execution vocabulary)

Suppose you write:

```python
df = spark.read.parquet("s3://lake/impressions/dt=2025-03-01/")
by_country = df.groupBy("country").agg({"revenue_usd": "sum"})
by_country.write.mode("overwrite").parquet("s3://curated/revenue_by_country/dt=2025-03-01/")
```

Nothing runs on **line 1–2** until **`write`** (an **action**). When `write` runs, Spark builds a **DAG** (directed acyclic graph) of **stages**:

- A **stage** is a set of tasks that can run **without** waiting on a **shuffle barrier** (roughly: pipelined work on partitions).
- A **shuffle** typically **ends** one stage and **starts** another—because tasks now need **data from many upstream partitions**.

**Tasks** are the actual threads of parallel execution: e.g. “read **file chunk 7**, parse Parquet row groups, **map** to `(country, revenue)`, **send** each key to the right reducer partition.”

**Interview tip:** Say “**shuffle** introduces a **stage boundary**” and you’ll sound like you’ve opened the Spark UI before.

## Why Spark vs Hadoop MapReduce (historical, still asked)

**MapReduce** (classic) materialized **intermediate** results to **disk** after **map** and often after **reduce**, and the API was **low-level**. Iterative algorithms (like **many steps of ML**) paid that **disk round-trip** every loop.

Spark’s early pitch was: **keep working sets in memory** between **iterations** when they fit, **spill** gracefully when they don’t, and express workflows as a **DAG of operators** rather than hand-rolled map/reduce classes.

Modern Spark is **still** **I/O** and **shuffle** bound for many **ETL** jobs—but the **DataFrame/SQL** layer plus **Catalyst** optimizations mean **you** get **predicate pushdown** and **column pruning** “for free” when you use **columnar** sources properly.

## When Spark is a strong choice

- **Large-scale ETL**: cleansing billions of rows, joins between **fact** and **dimension**, sessionization, aggregates for **reporting** marts.
- **Feature engineering** at scale: rolling windows, aggregations per **user** (watch skew!).
- **Batch** backfills: reprocess **N** days of **raw** after a bug fix—**idempotent** writers assumed (pairs with batch-etl guide).

## When Spark is a weak choice

- **Small** data (megabytes to low gigabytes fit in **RAM**): pandas / **DuckDB** / single-node SQL may be **faster end-to-end** once you count **cluster startup** and **JVM** overhead.
- **Millisecond** latency **serving** queries: use an **OLTP** database, **cache**, or **stream** processor.
- **Team** has **no** operational depth and **vendor warehouse** (Snowflake, BigQuery) can run the same SQL **cheaper** in **total cost of ownership**—**Staff** picks **economics**, not fashion.

## Worked example — order of magnitude

**Input:** 2 TB Parquet for `dt=2025-03-01`, **snappy** compressed, **50** files, **schema** with **30** columns but you only **`select 4`** columns.

**Without** **projection pruning** (hypothetical bad API): read **2 TB** from S3.  
**With** **columnar** **read** + pruning: read maybe **300 GB** (depends on column sizes—**illustrative**).

Spark’s optimizer plus **Parquet** means **the columns you never referenced might never be read from disk**—mention this in interviews; it’s a differentiator from “I treated Spark like reading CSV.”

## One “Staff answer” you can memorize

> “Spark is our **batch parallel** engine over **partitioned** **columnar** files. I care about **shuffle** cost, **partition** count, **join** strategy, and **skew**. For small or interactive slices, I’d push work to the **warehouse** or **single-node** engine unless the **pipeline** is already **standardized** on Spark.”

Next: [02-rdd-dataframe-dataset-sql.md](./02-rdd-dataframe-dataset-sql.md)

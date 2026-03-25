# RDD, DataFrame, Dataset, Spark SQL — In Depth

## Why there are multiple APIs (historical layers)

**RDD** (Resilient Distributed Dataset) was the **original** abstraction: a **partitioned collection** of **opaque records** (often tuples or custom classes) with **functional** operations (`map`, `filter`, `reduceByKey`). RDDs expose **maximum flexibility**: you control **exactly** what runs per record.

**DataFrame** layers a **schema** (column names and types) on top of distributed data—similar to a **table**. Under the hood, Spark still partitions rows, but the **Catalyst optimizer** can **see** your query structure and **rewrite** it.

**Dataset** (strongest in **Scala/Java**) adds **compile-time typing** to DataFrames—you get `Dataset[Customer]` instead of `Row`. PySpark **does not** have the same **typed Dataset** story in practice; **DataFrame** is the workhorse.

**Spark SQL** is the **SQL** surface over the **same** engine: `spark.sql("SELECT ...")` plans through **Catalyst** the same way as DataFrame operations.

**Interview answer:** “I write **DataFrames** / **SQL** unless I’m maintaining **legacy** RDD code or need a **primitive** escape hatch.”

## RDD — what still matters conceptually

Even if you never **author** RDDs in a new job, interviewers reference them because they explain **fault tolerance**.

Each RDD remembers **lineage**: “I am the result of **`filter`** applied to **parent RDD** P.” If a **partition** is lost on an executor, Spark can **recompute** that partition from **lineage** by **re-running** the **narrow** transformations from a **checkpoint** or **parent** **persist**.

**Important limitation:** long chains of RDD transformations without **`checkpoint`** can make **recovery** expensive (recompute from **source**) or hit **stack depth** limits in extreme cases.

## DataFrame mental model

A DataFrame is a **distributed table** of **Rows** attached to a **StructType** schema. Operations fall into:

- **Transformations** (lazy): `select`, `filter`, `withColumn`, `join`, `groupBy`, etc.
- **Actions** (eager): `count`, `collect`, `write`, `take`, `head`.

Until an action runs, Spark only grows a **logical plan**—it does **not** read all your data immediately.

## Predicate pushdown — explained with a story

You have **two years** of data laid out as:

```text
s3://lake/events/dt=2023-.../
s3://lake/events/dt=2024-.../
...
s3://lake/events/dt=2025-03-01/
```

You run:

```python
df = spark.read.parquet("s3://lake/events/").filter("dt == '2025-03-01'")
```

If `dt` is a **partition column** stored in the **directory structure** (Hive-style partitioning), Spark’s **scan** can **prune** directories and **avoid listing** or **reading** irrelevant **objects**.

Even inside **one** day’s Parquet files, if you `filter("event_type = 'impression'")`, **Parquet** **column statistics** and **Catalyst** may **push** filters into the **FileScan** operator—**skipping row groups** that can’t possibly match (depending on file **footer** stats and Spark version).

**What to say in interviews:** “I structure **lake** data so **partition columns** match my **filter** patterns; I **`select`** only needed **columns** so **columnar** storage can save **I/O**.”

## Projection pruning (column pruning)

Parquet stores **columns** separately. If the Parquet file has **100** columns but your query uses **5**, Spark only decodes those **5** from **disk**—this can be a **10–50×** I/O reduction depending on column widths.

**Anti-pattern:** `df = spark.read.parquet(path)` then later in a huge pipeline you forget you carried **fat** columns into a **shuffle**—you pay **network** and **CPU** serializing **unused** fields.

## Why naive Python UDFs are slow

A **Python UDF** forces **per-row** **serialization** between the **JVM** (where Spark executes most of the plan) and **Python**. For **billions** of rows, that boundary dominates.

**Better options:**

1. **Built-in Column expressions** (`when`, `concat`, `regexp_extract`, etc.)—stay on **JVM**.
2. **Pandas UDF / Arrow vectorized UDF** (API names evolved across versions)—**batch** rows in Apache Arrow buffers to amortize overhead.
3. Move **hot** logic into **SQL** or **Scala/Java** library.

**Staff nuance:** “I’d **prototype** in UDF, **profile**, replace **inner loop** with **vectorized** or **native** expressions.”

## Cache / persist vs laziness

`df.cache()` or `df.persist(StorageLevel.MEMORY_AND_DISK)` materializes **after** the next **action** that needs that **DataFrame**—then **reuses** partitions in memory (or spills to disk) for **subsequent** actions.

**Do not** cache by default. **Memory** is finite; **cached** RDDs **evict** others. Use cache when you **know** you’ll **re-read** the same **DAG branch** multiple times **in one session**—for example iterative **ML** or **repeated** queries in a notebook.

**Always** `unpersist()` when done in long-lived sessions to avoid **mysterious OOM**.

## One compact interview story

> “I treat Spark as a **columnar scan + shuffle engine**. I organize storage for **partition prune**, I **select** early, I **filter** early so Catalyst can **push down**, and I avoid **Python row UDFs** on huge facts unless **vectorized**.”

Next: [03-lazy-evaluation-dag-lineage.md](./03-lazy-evaluation-dag-lineage.md)

# Partitions, Narrow vs Wide, Shuffle — In Depth

## What a partition really is

At execution time, Spark splits a DataFrame into **partitions**: disjoint subsets of rows that **move together** through a **stage**. Each **task** processes **one** partition in that stage.

Partitions are **not** “a fixed number of bytes”—they depend on **how** data was read (one **HDFS** block vs many small files) and on **`repartition`/`coalesce`**.

## Why partition count matters

**Too few partitions** → **underutilization**: you might have **200** cores cluster-wide but only **20** tasks; **80%** of CPUs idle.

**Too many partitions** → **overhead**: each task has **startup** cost, **scheduling** chatter, and **tiny** I/O reads; **shuffle** **writes** become **millions** of tiny files if unmanaged.

**Heuristic many teams use** as a **starting guess**: aim for **shuffle** output partitions such that **each** task processes **~100–200 MB** of **shuffle** data—**highly** workload dependent; **Spark UI** is truth.

## Narrow vs wide transformations (deep)

**Narrow:** `map`, `filter`, `select`, `drop`, `withColumn` composed of **row-local** expressions that don’t need **peer rows**.

**Interpretation:** output partition **i** depends **only** on input partition **i**. Spark can **pipeline** multiple narrow ops without **reshuffling**.

**Wide:** `groupByKey`, `groupBy().agg`, `join` (usually), `distinct`, `repartition`, `orderBy` **globally**.

**Interpretation:** to compute correct answer, nodes must **exchange** records across the cluster → **shuffle** **(aka exchange)**.

## Shuffle mechanics (so you can debug “why is my job slow?”)

Conceptually a shuffle has **two sides**:

1. **Map side** (same stage as preceding transforms): each task writes **shuffle data** to **local disk** organized into **files** by **reduce partition id**.

2. **Reduce side**: each reducer **fetches** its partition’s shuffle blocks from many map tasks, **merges** them (often with **sort**), runs **aggregate** or **join merge**.

**Resource pressure:** **disk** spill if memory tight, **network** saturation, **slow** disks on a **noisy neighbor** node.

## `spark.sql.shuffle.partitions`

Many wide operations default to **200** shuffle partitions (`spark.sql.shuffle.partitions`). That default dates from **small cluster** era; **today** you commonly **bump** to **800**, **2000**, etc.—**after** validating.

**Staff observation:** Blindly setting **10,000** partitions can **explode** small file counts on **output** and **hurt** **metadata** operations—balance **parallelism** vs **scheduling** overhead.

## `coalesce` vs `repartition`

**`coalesce(N)`** when **reducing** partition count **often avoids** **full shuffle** by **collapsing** existing partitions (implementation details vary; Spark tries to avoid **full** redistribution).

**Use case:** after a **heavy filter** dropped **90%** of rows—**600** tiny partitions remain; **`coalesce(80)`** before **expensive** **wide** op reduces **task count**.

**`repartition(N)`** **always** **shuffles** data (full exchange) to **spread** rows **evenly**—good when you **know** you have **skew** from upstream ingest or need **even** **distribution** before **join**.

**Interview trap:** “I always `repartition` before every stage” — unnecessary shuffle **tax**.

## Worked mini-example (numbers)

Assume **1.2 billion** rows after filter, **~120 GB** on disk in **Parquet**. If you **groupBy** with **200** shuffle partitions:

- **Average** **~600M rows / 200** isn’t how it works—**skew** dominates—but **idealized average** is **6M** rows per reducer.  
If **one** key appears **300 million** times, **one** reducer gets **300M** rows → job “stuck” at **199/200** tasks.

**That** is why **salting** / **AQE** / **two-stage** aggregation appears in the join doc.

## Soundbite

> “I set **shuffle partitions** from **data size** and **cluster width**, validate in **Spark UI** for **skew**, and use **`coalesce`** to **reduce** **empty** partitions without paying **extra** **shuffle**.”

Next: [05-joins-broadcast-skew.md](./05-joins-broadcast-skew.md)

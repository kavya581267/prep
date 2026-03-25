# Spark — Interview Questions with Model Answers

Use these as **speaking drills**—answers are **paragraph** depth, not one-liners.

---

## Beginner

### Q1. What is Spark?

**Model answer:** Spark is a **distributed data processing engine**. You write **SQL** or **DataFrame** code that logically looks like working on one table, but behind the scenes it compiles into a **DAG** of **stages** and **tasks** that run on **executors** across a cluster. It’s optimized for **parallel scan** of large **columnar** files and **shuffle-based** joins and aggregations. The **driver** coordinates everything; **executors** do the parallel work.

### Q2. Transformation vs action?

**Model answer:** **Transformations** build the **lazy** query plan—they don’t execute immediately (`select`, `filter`, `join` without sinking). An **action** triggers execution because the driver needs a **materialized** result: `count`, `write`, `collect`, `take`. Lazy evaluation lets **Catalyst** optimize the whole chain before touching terabytes of data.

### Q3. What is a partition?

**Model answer:** A partition is a **chunk** of a distributed dataset processed by **one task** in a **stage**. Partition count controls **parallelism**. Too few partitions leave CPUs idle; too many create **scheduling** overhead and **small files** on output. Shuffle operations **redistribute** rows into new partitions keyed by **hash/range** rules.

### Q4. Why Parquet?

**Model answer:** Parquet is **columnar**: Spark reads only requested **columns** efficiently, applies **compression** per column, and carries **statistics** (min/max per row group) to **skip** data. That interacts well with **predicate pushdown** and **projection pruning** in **Catalyst**.

### Q5. What does `collect()` do?

**Model answer:** It **brings all partitions** to the **driver** as an in-memory collection. It should only be used for **tiny** results—debug samples or small aggregates. On large datasets it **OOMs** the driver or becomes impossibly slow.

---

## Intermediate

### Q6. What is a shuffle?

**Model answer:** A shuffle is a **distributed exchange**: records have to move between nodes so that **all rows sharing a key** end up on the **same reducer partition** for a **groupBy** or a **join**. Implementations write **map-side shuffle files** to disk, **fetch** them on reduce side, and merge/sort. Shuffle is often the **dominant cost** in big jobs because it’s **network + disk + CPU** heavy.

### Q7. Narrow vs wide transformation examples?

**Model answer:** **`filter`** and **`select`** are **narrow**: each output partition depends on **one** input partition, so Spark can **pipeline** them in one **stage**. **`groupBy`** and most **`join`** are **wide**: an output partition depends on **many** input partitions, requiring a **shuffle stage boundary**.

### Q8. When broadcast join?

**Model answer:** If one join side is **small enough** to fit in memory across the cluster after being **broadcast** to executors, Spark can avoid shuffling the **large** fact table for that join. It’s chosen based on **size stats** and **`autoBroadcastJoinThreshold`**. Mis-sized broadcast can **OOM** the driver or hurt **network**—I validate with **`explain`** and historical **size**.

### Q9. Data skew—how handle?

**Model answer:** Skew means a few keys hold most rows, so one **reducer** does disproportionate work. With Spark 3 I’d rely on **AQE skew join** first when enabled. Manually I might **salt** keys for **two-phase aggregation**, **pre-aggregate** the big side, **isolate** NULL buckets, or **filter** skewed keys into a separate pipeline.

### Q10. What is `spark.sql.shuffle.partitions`?

**Model answer:** It’s the default **reducer partition count** for shuffle operations in SQL/DataFrame. It shapes how many parallel reduce tasks exist after shuffle. Too low = fat tasks, risk OOM; too high = overhead and tiny outputs. I tune based on **shuffle size** seen in Spark UI and often complement with **AQE coalescing**.

---

## Advanced

### Q11. Explain Catalyst briefly.

**Model answer:** Catalyst converts DataFrame/SQL into a **logical plan**, applies rule-based and cost-based optimizations—**predicate pushdown**, **projection pruning**, **constant folding**, **join reordering** when stats exist—then generates a **physical plan** choosing join algorithms and shuffle layout, optionally with **whole-stage codegen** via Tungsten for narrow pipelines.

### Q12. AQE—value and requirements?

**Model answer:** Adaptive Query Execution adjusts plans at runtime based on **shuffle statistics**: reduce too many small shuffle partitions, split skewed partitions, switch join strategies. It needs accurate enough runtime stats; if input has **no** stats and extremely weird patterns, I still might intervene manually.

### Q13. Speculation risks?

**Model answer:** Speculative duplicate tasks can finish **writes** twice if outputs aren’t **idempotent**. I use speculation cautiously for **ETL** **writes** or ensure **idempotent** sinks/partition overwrites.

### Q14. Driver OOM causes?

**Model answer:** Collecting huge results, oversized **broadcast**, accidentally materializing big lists in driver code, some metadata operations gone wrong. I fix by pushing aggregation to cluster, shrinking broadcast inputs, increasing driver memory **only** after eliminating sins.

### Q15. Why avoid Python UDF on huge data?

**Model answer:** Row-wise Python UDFs cross the **JVM↔Python** boundary per row, which dominates runtime. I prefer SQL built-ins or **Arrow vectorized** UDFs that process **batches**.

---

## Staff scenario

### S1. Job stable at 45 min suddenly 3 hours—what do you check?

**Model answer:** I’d compare **Spark UI** stage graph **before/after**. I’d look for a **new** sort-merge join vs broadcast, increased **input** **bytes** read, **spill**, **GC**, **shuffle skew**, **S3 throttling**, and **partitions** blowing up due to config drift. I’d verify **data** changes (skewed key emerged) vs **code** change vs **infra** (instance type). Then reproduce on a **single day** partition to bisect.

### S2. “Exactly-once” Spark streaming to Delta—true?

**Model answer:** I’d clarify **end-to-end** exactly-once is a **system property**: Spark’s **checkpoint** gives **exactly-once processing semantics** relative to offsets, but the **sink** must commit atomically—often **idempotent MERGE** keyed by event id. I’d say **effectively-once** with **dedupe** is the honest production pattern unless the full stack proves **txn** end-to-end.

---

## Practice drill

Pick **three** questions above; **record** yourself for **90 seconds** each without reading verbatim.

Back to: [00-START-HERE.md](./00-START-HERE.md)

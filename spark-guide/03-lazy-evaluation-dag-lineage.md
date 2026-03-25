# Lazy Evaluation, DAG, Lineage — In Depth

## Lazy evaluation in plain language

When you write:

```python
a = spark.read.parquet(path)
b = a.filter("country = 'US'")
c = b.select("user_id", "revenue_usd")
```

**No cluster work** has happened yet beyond **maybe** **inferring** schema or **listing** paths depending on API. Spark has recorded **three** operations in a **logical plan**: **scan**, **filter**, **project**.

Only when you run an **action**—`c.count()`, `c.write...`, `c.collect()`—does Spark:

1. **Optimize** the logical plan (Catalyst).  
2. **Choose physical operators** (broadcast hash join vs sort-merge join, etc.).  
3. **Submit** **jobs** consisting of **stages** and **tasks**.

**Why lazy?** Because Spark can **reorder** safe operations: for example, **`filter`** before **`project`** might reduce rows earlier in some plans, and **combine** multiple maps into **one** **codegen** stage.

**Beginner mistake:** Assuming `count()` on `a` and `count()` on `b` “reuses” work unless **`b`** was **materialized** (cache/checkpoint/write). Spark **may** **re-scan** Parquet twice unless you **persist** or the optimizer can **reuse** **exchange** in limited cases—**don’t rely** on magic; **measure**.

## DAG intuition (what you’d draw on a whiteboard)

Think of each **operation** as a **node** and **data dependencies** as **edges**:

`Scan Parquet → Filter → Project → Exchange (shuffle) → Aggregate`

A **shuffle** is a **fork+join**: every **map-side** task emits records grouped by **shuffle partition**, **_sorted or hashed_ depending on join/agg**, then **reduce-side** tasks pull their partition.

The **DAG is acyclic**—no infinite loops in the **static plan** (streaming is different internally but still structured).

## Stages and shuffle boundaries

**Rule of thumb:** **wide dependencies** (need data from **many** parents to produce an output partition) introduce **shuffle** and therefore **new stage**.

**Narrow dependencies:** each output partition depends on **at most one** input partition → can **pipeline** in one stage (map/filter on HDFS block, for example).

**Interview question:** “What breaks pipelining?” **Answer:** shuffle; **sorted** operations that require **total order** across partitions; **repartition** to increase/decrease partition count with full shuffle.

## Lineage and fault tolerance

If an **executor** dies mid-stage, Spark **reruns** **lost tasks** on other executors. For **map** stages, it’s **cheap** if **sources** are **immutable** files. For **shuffle** stages, Spark may need to **re-fetch** **shuffle** **files** if **external shuffle service** still has them—or **recompute** the **map side** of shuffle if necessary.

**Checkpoint** (especially in **Streaming**) **cuts** lineage by writing **reliable** **storage** and **starting fresh** logical graph—prevents **unbounded** recovery cost.

## `explain()` — how to actually use it

`df.explain()` prints the **physical** plan. `df.explain(mode="extended")` also shows **logical** and **optimized** logical plans.

**When you’re stuck**, look for:

- **Filter** **before** **join**?  
- **BroadcastExchange** (small table replicated)?  
- **SortMergeJoin** vs `BroadcastHashJoin`?  
- **Exchange** with **large** partition counts?

**Staff behavior:** “I’d attach **`explain`** from prod-like run to the design doc before declaring victory.”

## Anti-pattern: driver-side loops firing jobs

```python
for country in countries:
    df.filter(df.country == country).write.parquet(f".../{country}")
```

Each loop iteration triggers a **full** submit—**massive** **scheduling** overhead and potentially **repeated** **scans**.

**Better:** **single** **job** with **partitionBy** on write (if output layout permits) or **one group** operation with care—still watch **skew** if one **country** dominates.

## Soundbite

> “Lazy execution is an **optimization contract**: I **inspect** **`explain`**, I understand **stage** boundaries at **shuffles**, and I avoid **driver** loops that **multiply** **jobs**.”

Next: [04-partitions-narrow-wide-shuffle.md](./04-partitions-narrow-wide-shuffle.md)

# Load: Warehouse & Lakehouse — In Depth

## “Load” is where physics meets accounting

By **load**, we **materialize** curated tables **consumers** trust: BI tools, ML feature jobs, **finance** exports.

**Physics:** writing **petabytes** generates **small files**, **metadata** operations, **compaction**, **locking** behaviors.

**Accounting:** totals after load must match **definitions** agreed with **finance** (timezone, **gross** vs **net**, invalid traffic rules).

## Raw vs curated vs serving (repeat on purpose)

**Raw:** evidence, **immutable** append. **Corrections** come as **new** files or **compensating** rows—**not** silent overwrites (unless **legal** retention mandates delete—then **GDPR** workflows).

**Curated:** **deduped**, **typed**, **conformed** to **surrogate keys**.

**Serving:** **wide** marts optimized for known query patterns—**may** **denormalize** heavily.

## Hive-style partitions vs lakehouse tables

**Hive-style:** `s3://curated/fact/dt=2025-03-25/hour=14/....parquet`.

Engines discover partitions via **directory listing** or **catalog** metadata.

**Lakehouse (Iceberg/Delta):** **table** is a **logical object** with **manifests**; commits are **atomic** **snapshots**. **Time travel** = read snapshot id `N`.

**Staff advantage:** **MERGE** + **schema evolution** + **concurrent writers** coordination **better** than naked Parquet directories for **teams** at scale.

## MERGE pattern — walk through

**Staging** for `dt=D` has **clean** rows with **`event_id`** unique.

**Fact** already has prior days.

```sql
MERGE INTO fact f
USING staging s
ON f.event_id = s.event_id AND f.dt = s.dt   -- composite depends on model
WHEN MATCHED THEN UPDATE SET ...             -- only if late-arriving columns change
WHEN NOT MATCHED THEN INSERT ...
```

**Interview nuance:** **`WHEN MATCHED`** updates for **facts** can be **dangerous**—do you **want** to **rewrite** history? Sometimes **append-only** fact + **correction** **rows** with **`delta_amount`**.

## Small files on write (prevent early)

If Spark writes **2000** files for one **`dt`** because **shuffle partitions** = **2000**, **downstream** queries **list** slowly.

**Levers:** **`coalesce`** before write (careful with **skew**), **`repartition`** to target **~128–256MB** files, **post-write compaction** job (Delta **`OPTIMIZE`**, Iceberg **`rewrite_data_files`**).

## Two-phase publish (no torn reads)

1. Write to **`.../dt=D/_staging_runid=xyz/`**.  
2. **Validate** row counts + **constraints**.  
3. **Atomic** **catalog** swap: promote `xyz` to **canonical** `dt=D` (engine-specific).

**Without this**, a reader running `SELECT sum(revenue) WHERE dt=D` mid-write might **undercount**.

## Soundbite

> “Load is **not** ‘write Parquet’—it’s **atomic publish**, **right file sizes**, **MERGE** semantics aligned to whether **history** **mutates**, and **verification** before **consumers** see **`dt=D`**.”

Next: [05-partitioning-files.md](./05-partitioning-files.md)

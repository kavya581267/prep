# Joins, Broadcast, Skew — In Depth

## Join is not one operation internally

When you write `df_fact.join(df_dim, on="campaign_id")`, Spark may physically execute:

- **Broadcast Hash Join** (small dimension fits in memory **broadcast** to all executors), or  
- **Shuffle Hash Join**, or  
- **Sort-Merge Join** (both sides large—**shuffle** on join key, **sort** within partition, **merge** walk).

The choice depends on **statistics** (sizes), **hints**, and **AQE** adaptations.

## Broadcast join — the happy path

**Story:** `dim_campaign` has **50,000** rows, **few MB**. `fact_impressions` has **2 billion** rows on **S3**.

Spark can **collect** the **small** table to the **driver** (or **direct broadcast** in newer paths), serialize it once, and ship it to **every executor**. Each executor **joins locally** while scanning **its** share of **fact**—**no shuffle on the fact side for this join**.

**Configs to know:** `spark.sql.autoBroadcastJoinThreshold` (size limit). If you **raise** it recklessly, you’ll **OOM** driver or executors.

**Symptoms of a bad broadcast:** driver **GC** storms, **`BroadcastExchange`** stage **hung**, **“java.lang.OutOfMemoryError: Java heap space”** on driver.

**Staff mitigation:** **Approximate** dimension size, **filter** dimension **early**, or **bucket** the fact if you **must** sort-merge.

## Sort-merge join — default for big–big

Both sides get **shuffled** by `campaign_id`. Each reducer gets **all** rows for **some** subset of keys (hash partitioning of key to **N** reducers).

**Cost:** two **big** shuffles (sometimes optimized into one depending on plan).

**When it’s fine:** keys are **reasonably** **uniform**; **partition** counts are tuned; **network** healthy.

## Skew — what it looks like on the UI

**Spark UI** → **Stages** → one task takes **10×** longer than siblings; **input records** **massive** for that task.

**Real-world skew causes in ads data:**

- `NULL` **user** or **device** bucket absorbing **untracked** traffic  
- A **single mega-campaign**  
- **Bots** concentrated on one **placement**

## Mitigation playbook (learn in order)

### 1) AQE skew join handling (Spark 3)

With **`spark.sql.adaptive.enabled=true`**, Spark can **detect** skew at runtime based on **shuffle statistics** and **split** the skewed partition into **subpartitions**, **replicating** the small side accordingly.

**Interview:** “I’d enable AQE in prod; still validate because not all skew patterns are caught.”

### 2) Salting (manual technique)

Add random suffix to **hot** key during pre-join:

```text
key_salted = concat(campaign_id, '-', random_int(0, S-1))
```

**Explode** both sides with same salt range? Usually used in **two-phase aggregation**:

- Phase 1: partial aggregate per `(key, salt)`  
- Phase 2: **global** aggregate per `key`

For joins, patterns get **trickier**—often **isolate** hot keys to **separate** **pipeline**.

### 3) Isolate + special-case

`WHERE campaign_id IS NOT NULL` for **join** path; **separate** job for **NULL** bucket with different rules.

### 4) Pre-aggregate large side

If you only need **SUM(revenue) by campaign**, **aggregate fact first** to **millions** of rows instead of **billions** before joining **wider** dims.

## Worked skew toy (talk through aloud)

- **2B** fact rows across **200** shuffle partitions → **avg 10M** rows/task (oversimplified).  
- **One** `campaign_id = X` accounts for **400M** rows**.**  
- If hash puts **X** on **partition 47** → partition 47 processes **400M** join rows, others far less.

**Goal:** break partition 47 into **K** subparts (AQE) or **salt** aggregation of revenue for X before joining metadata.

### Broadcast misuse story

Someone broadcast a “small” `geo_lookup` **1 GB** **zip** table; **executors** **OOM**. Fix: **pre-filter** to needed countries, **partition** join instead, or **increase** memory **with** **eyes open**.

## Soundbite

> “I **inspect** join **physical** plan for **broadcast** vs **sort-merge**, I **size** broadcast with **stats**, and I treat skew as **expected** in ads facts—**AQE first**, **manual salt** when needed.”

Next: [06-catalyst-tungsten.md](./06-catalyst-tungsten.md)

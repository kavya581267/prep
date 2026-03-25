# Partitioning & File Sizing — In Depth

## Why we partition tables in the file/lake world

**Partitioning** **prunes** I/O: queries with `WHERE dt BETWEEN ...` **touch** fewer **directories** / **metadata** **entries** than scanning **one** giant folder.

**Good partition keys** have:

- **High** **filter** rate in real queries (`dt`, `region` sometimes).  
- **Moderate cardinality**—not **billions** of distinct values for **fact** tables on storage layout granularity.

## The “user_id partition” anti-pattern

If you partition a **high-cardinality** fact by **`user_id`** (billions), you get **billions** of **directories** → **metadata** explosion, **listing** slow, **Spark** driver **pressured** to enumerate files.

**Alternative:** partition by **`dt`** only; **cluster** / **bucket** by **`user_id`** within day if needed for joins.

## Target file size (rule-of-thumb culture)

Many teams aim for **128MB–512MB** **compressed** **Parquet** per file for **analytic** scans: **large** enough to amortize **S3 GET**, **small** enough for **parallelism** within day.

**Too small:** **millions** of **KB** files → **list** APIs throttled, Spark **task** overhead.

**Too large:** **few** files → **under** utilizing cluster **parallelism** for that partition.

## Bucketing / clustering (when it helps)

**Bucket** (or **cluster**) on **`campaign_id`** if cardinality is **moderate** can **co-locate** rows that will be **joined** frequently, reducing **shuffle** on repeated **pipelines**.

**Tradeoff:** **Inflexible** if **query patterns** **shift**—you **recluster** periodically.

## Z-order / data layout engines

**Delta ZORDER** / similar: reorganize data for **multi-column** filter skipping. **Costs** **rewrite** **I/O**—run on **cadence** balanced by **query savings**.

## Worked example — small file death spiral

**Symptom:** `dt=2025-03-01` has **40,000** Parquet files of **2MB** each.

**Cause:** Kafka → raw land **micro-batches** every **30s** without **compaction**.

**Fix:** **hourly compaction** job **merging** to **target** size; **long-term** fix **buffer** in ingest or **dedicated** **compaction** **service**.

## Soundbite

> “Partition for **prune**, size files for **scan efficiency**, bucket only when **join** physics justifies it—I measure **list** latency and **files per partition** as first-class **metrics**.”

Next: [06-data-quality-contracts.md](./06-data-quality-contracts.md)

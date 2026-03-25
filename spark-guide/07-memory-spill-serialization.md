# Memory, Spill, Serialization — In Depth

## Executor memory is not “one big heap for my DataFrame”

JVM heap on an executor is **partitioned** **conceptually** between:

- **Execution memory** (joins, **aggregations**, **shuffle** buffers)  
- **Storage memory** (cached/persisted blocks)  
- **Reserved** overhead and **user** memory for **internal structures**

When execution memory is stressed, Spark **spills** shuffle or aggregation data to **disk**. **Spill** prevents immediate OOM but **kills** performance if chronic.

**What you see:** “**Spill (disk)**” columns high in Spark UI **aggregators** / shuffle stages.

## Out-of-memory failure modes (recognize each)

### Driver OOM

Common causes:

- **`collect()`** on large DataFrame  
- **`toLocalIterator()`** misuse  
- **Broadcast** too large  
- **deeply nested** logical plan issues (rarer)

**Fix:** **Never** `collect()` big data—**write** to storage or **aggregate** first.

### Executor OOM

Common causes:

- **Skew** creating **giant** partition in memory for **hash aggregation**  
- **Broadcast** unexpectedly large per executor copy (still bounded by broadcast mechanics but heavy)  
- **Too many concurrent tasks** on a **small** heap  
- **Cache** **everything** indefinitely in a **notebook**

## Spill: good emergency brake, bad steady state

Spill writes serialized bytes to **executor local disk** (often **YARN** `nodemanager` local dirs). If disk is **slow** or **filled**, tasks **time out**.

**Staff approach:** If spill > **X%** of shuffle **bytes** consistently, fix **partition count**, **memory per executor**, or **skew**—not “add **shuffle** partition count forever.”

## Serialization formats (conceptual)

Shuffle data and **broadcast** objects are **serialized** for network/disk. **Kryo** (often recommended for custom types in **Scala/Java**) can be **faster** than Java serialization for **non-Row** objects.

**PySpark Row** shuffle still crosses **Python** boundaries when Python is in the **driver** path, but **executors** are JVM—many Spark SQL operations avoid **Python** entirely on executors.

## Garbage collection pressure

Huge heaps + many **short-lived** objects per task → long **stop-the-world** pauses → **heartbeats** missed → **executor** considered dead.

**Mitigation levers:**

- Reduce **object churn** (prefer **primitives** in hot code—often **you** can’t for DataFrame, but **UDFs** matter).  
- **Right-size** heap **vs** **parallelism** tradeoff (sometimes **smaller** heap **more** executors).  
- Tune **GC** (**G1** common) with cluster vendor guidance.

## `persist` / `cache` discipline

**When it helps:** iterative **ML** / **graph**-like revisits of same dataset within **one** application.

**When it hurts:** **one-pass** ETL that **never** revisits cached DF—**you** consumed RAM for nothing.

**Pattern:** `df.unpersist(blocking=False)` when done; for blocking release in tight memory scenarios, **`blocking=True`** (API varies slightly by language version—verify).

## Worked example — broadcast too big

**Symptom:** Driver heap spikes during **`BroadcastExchange`**.

**Investigation:** `explain` shows **BroadcastHashJoin**; **`dim`** looked small in dev (**filtered**), but **prod** dimension has **hidden** historical rows.

**Fix:** **dynamic partition pruning** on dimension ingest, **increase** threshold only after measuring **serialized size**.

## Soundbite

> “I distinguish **driver** vs **executor** OOM; I treat **spill** as a **signal** not a **solution**; I avoid **`collect`** on facts and I **invalidate** unnecessary **cache**.”

Next: [08-tuning-executors-aqe.md](./08-tuning-executors-aqe.md)

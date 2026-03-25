# Batch ETL — Interview Questions with Model Answers

---

## Beginner

### Q. Define ETL vs ELT.

**Answer:** ETL **transforms before** loading into a destination—often to reduce **egress** or enforce **cleansing** early. ELT loads **raw or lightly typed** data into a **powerful warehouse** and applies transforms **in-database** with SQL. Choice depends on **governance**, **cost**, and **team** skills.

### Q. Why still use batch if we have streaming?

**Answer:** Streaming gives **fast** **signals**; batch often defines **official** **numbers** after **late data**, **human corrections**, and **cross-system reconciliation** settle. Many finance processes **require** **closed periods**. I use both with an explicit **contract** for which is **official**.

---

## Intermediate

### Q. How make incremental extract safe?

**Answer:** Use a **watermark** with **overlap window** to catch stragglers, then **dedupe** by **primary key** downstream. Periodically validate with **full** snapshot or **hash aggregate** for critical tables. Document **late arriving** **updates** policy.

### Q. Explain SCD2 vs SCD1.

**Answer:** SCD1 **overwrites** attributes—no history. SCD2 **versions** rows with **valid_from/to** so we can **join** facts to **dimension as-of** event time—needed when **historical** **truth** matters for reporting or compliance.

### Q. Prevent duplicates on Airflow retry?

**Answer:** Don’t **append** directly to **canonical** fact in the **same path** twice without keys. Use **staging + atomic swap**, **`MERGE`**, or **partition overwrite**. Design tasks so **retry** **replaces** **partial** work.

### Q. Small files problem?

**Answer:** Many **tiny** objects increase **list** and **open** overhead and hurt **vectorized** scan throughput. Fix via **compaction**, adjusting **upstream** **batch** sizes, **`coalesce`** before **write** (careful with skew), or **engine** **`OPTIMIZE`**.

---

## Advanced / Staff

### Q. Design idempotent daily fact load from raw events.

**Answer:** Land **raw** append-only with **`ingestion_batch_id`**. Build **staging** **dedupe** by **`event_id`**. Validate with **row + aggregate** checks. **MERGE** into **`fact`** keyed by **`event_id`** **partitioned** by **`dt`**, or **overwrite** **`dt` partition** after **two-phase** write. Log **published** **manifest** with **run metadata**.

### Q. Partner says “batch totals disagree with dashboard.”

**Answer:** Align **definitions**: **timezone cutoff**, **dedupe** rules, **invalid traffic** removal, **currency**. Compare **same** **grain** (hour vs day). Often stream **approximate** vs **batch authority**—I’d establish **which** is **billing truth** and **document** **latency**.

### Q. How cost-manage a growing Spark ETL?

**Answer:** **Partition pruning**, **column pruning**, reduce **shuffle** via **broadcast** or **pre-agg**, right-size **clusters**, **compaction** to cut **metadata**, **chargeback** **\$** to owners, consider **warehouse** offload for **simple** **SQL** **layers**.

---

## Drill

Record yourself for **2 minutes** on **idempotent publish** + **SCD2** join predicate—common **Staff** combo.

Back: [00-START-HERE.md](./00-START-HERE.md)

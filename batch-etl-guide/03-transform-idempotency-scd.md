# Transform: Idempotency, Determinism, SCD — In Depth

## Why batch transforms must be idempotent

Orchestrators **retry** failed tasks. Engineers **re-run** a day after fixing a bug. **Backfills** reprocess months.

If the second run **adds** duplicate **facts** or **double-multiplies** revenue, you lose **trust**.

**Idempotency** means: **multiple executions** of the **same** **logical** job for **`dt=D`** converge to the **same** final dataset state **as one correct run**.

## Patterns that achieve idempotency

### Partition overwrite for `dt=D`

Write output to `.../dt=2025-03-25/` with **dynamic overwrite** of **that** partition **only** (engine-specific syntax). **Second** run replaces partition wholesale.

**Risk:** **Readers** querying mid-write see **partial** partition unless **atomic** **partition** swap or **two-phase** write (write temp → **promote** metadata).

### MERGE / upsert keyed by natural id

**Delta/Iceberg/BQ** `MERGE INTO fact USING staging ON fact.event_id = staging.event_id WHEN MATCHED THEN UPDATE ... WHEN NOT MATCHED THEN INSERT`.

**Great** when **intraday** **multiple** loads land **same** **business keys**.

### Deduplicated append + view

**Append-only** fact with duplicates; nightly **view** `SELECT * FROM (SELECT *, row_number() OVER (PARTITION BY event_id ORDER BY ingest_ts DESC) rn FROM fact_raw) WHERE rn=1`.

**Tradeoff:** storage + query cost; **simplicity** for **immutable** **audit** trail.

## Determinism (subtle but interview-worthy)

If two runs produce **different row order** but same **set**, that’s usually fine. If they produce **different aggregates** because of **`now()`** or **non-stable sort**, you’ve created **flaky tests** and **confusing diffs**.

**Rules:**

- Parameterize **`run_ts`**, **`batch_id`** from orchestrator.  
- Always **`ORDER BY` deterministic** columns before **dedupe** tie-break.

## SCD — slowly changing dimensions (tell a story)

**Scenario:** `dim_campaign` has **`campaign_id`**, **`name`**, **`advertiser_id`**.

**SCD Type 1 (overwrite):** name change **updates row in place**—**history lost**. Fine if name typos irrelevant to **finance**.

**SCD Type 2 (track history):** each change inserts **new row** with **`valid_from`**, **`valid_to`**, **`is_current`**. **Fact** tables join using **`dimension_key` surrogate** tied to **version** active at **`fact.event_ts`**.

**Bridge to revenue:** **Guaranteed** deals often require **historical** truth—“**what was the name on the contract date**?”—that’s **SCD2** territory.

## Worked example — join facts to SCD2

**Fact** has `campaign_natural_id` + `event_ts`.

**Join predicate conceptually:**

```sql
JOIN dim_campaign c
  ON f.campaign_natural_id = c.campaign_natural_id
 AND f.event_ts >= c.valid_from
 AND f.event_ts <  c.valid_to  -- or <= depending on convention
```

**Bug:** half-open vs closed intervals wrong by **one** second → **double match** or **none**.

## Soundbite

> “Transform encodes **business time**: **idempotent** publish strategies for **`dt`**, **deterministic** SQL, and **SCD2** when **history** is **legally** or **financially** **material**.”

Next: [04-load-warehouse-lake-merge.md](./04-load-warehouse-lake-merge.md)

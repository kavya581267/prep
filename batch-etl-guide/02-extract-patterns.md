# Extract Patterns — In Depth (Full, Incremental, CDC)

## What “extract” really means

**Extract** is the **first** time you **copy** operational data into your **analytics** **control plane** (lake, warehouse staging). Everything downstream—**quality**, **joins**, **reporting**—assumes **extract** is **correct**, **complete**, and **auditable**.

## Full extract (snapshot)

**How it works:** Pull **entire table** (or **entire partition**) each run.

**When it shines:**

- Table is **small** (config dimensions).  
- Source has **no reliable change** columns.  
- You need **periodic** “**reset** truth” to catch silent **mutations** missed by incremental.

**Costs:**

- **Heavy** on **source** DB if done naively (**lock** risk, **I/O**).  
- **Network** egress charges.  
- **Late** runs mean **big** copies.

**Engineering mitigations:** read **replicas**, **export** APIs, **parallel** slice by **key** ranges, **compress** in flight.

## Incremental extract (watermark / cursor)

**How it works:** Remember **`last_successful_max(updated_at)`** or **`last_id`**. Next run pulls **only rows where** **`updated_at > watermark`** (plus **careful** **overlap** buffer).

**Why overlap buffer:** A row might **commit** at **T** but be **visible** to your reader **slightly** later than watermark boundary—**short** **replay** window (e.g., 5–15 minutes overlap) catches **near-miss**. Then **dedupe** downstream.

**Failure modes:**

- Application **doesn’t update** `updated_at` on **all** **code paths** → **silent** **miss**.  
- **Backdated corrections** **fall** **below** watermark unless **full scan** periodic.  
- **Clock skew** across shards (rare) causes oddities.

**Staff practice:** Monthly **full** **reconciliation** **hash** on **critical** tables even if daily incremental.

## CDC — change data capture

**How it works:** Database **transaction log** (binlog, WAL) → **Debezium**/vendor tool → **Kafka/Kinesis** → consumer **merges** changes.

**Strengths:**

- **Low-impact** on **OLTP** compared to **polling**.  
- Captures **deletes** that incremental `updated_at` might miss if soft-delete not enforced.  
- Near-real-time **feed** into **micro-batch** **processors**.

**Complexities:**

- **Schema** **evolution** must be **handled** (ALTER adds column mid-stream).  
- **Ordering** per primary key must be **guaranteed** or **merge** must be **commutative** with **last-write-wins** rules explicitly chosen.  
- Initial **snapshot** + **stream** **handoff** is a **project** (backfill + **offset** alignment).

## Worked example — incremental with overlap

**Watermark** **`2025-03-24 23:55:00`**.

**Query:** `WHERE updated_at > watermark - interval '10 minutes'` → captures stragglers.  
Downstream **dedupe** by **`id`** using **`row_number()`** over **`updated_at desc`**.

## Worked example — CDC delete

**CDC event:** `DELETE pk=9001`.  
**Warehouse row** must **vanish** or **`is_deleted=1`** depending on **model**.

If you only ever **append** incremental **without** **delete** handling, **ghost** rows remain—**bad** for **inventory** or **campaign** **status**.

## Soundbite

> “Extract chooses how **truth** enters our world: **full** for **reset**, **incremental** for **scale** with **overlap** + **dedupe**, **CDC** when we must see **deletes** and **minimize** **OLTP** **load**—each with **explicit** **failure** **modes**.”

Next: [03-transform-idempotency-scd.md](./03-transform-idempotency-scd.md)

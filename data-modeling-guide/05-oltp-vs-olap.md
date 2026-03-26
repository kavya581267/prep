# 5 — OLTP vs OLAP: two kinds of “truth”

**Goal:** Stop forcing one schema to do **transactions** and **analytics** at the same time without a plan.

---

## 5.1 OLTP (online transaction processing)

**Workload:** Many short **reads/writes**; **row-level** operations; **strong consistency** per transaction; users wait on **milliseconds**.

**Examples:** Checkout, account update, campaign status change, entitlement check.

**Shape:** **Normalized** (often to 3NF), **many small** tables, **indexes** tuned for **point lookups** and **short ranges**.

---

## 5.2 OLAP (online analytical processing)

**Workload:** Fewer huge **scans** and **aggregations**; “sum/dedupe by dimension across months”; latency **seconds to minutes** acceptable in batch; **columnar** storage wins.

**Examples:** Finance close, executive dashboards, audience reach reports.

**Shape:** Often **star/snowflake** (file `06`), **wide** fact tables, **precomputed** rollups.

---

## 5.3 Why “one database to rule them all” breaks

| Concern | OLTP pressure | OLAP pressure |
|---------|----------------|---------------|
| Index design | Speed up single-row paths | Speed up billion-row scans |
| Locking | Frequent small txns | Bulk loads |
| Schema churn | Feature velocity | Stable contracts for BI |

**Pattern:** **Operational DB** + **CDC/replication** → **warehouse/lake** → **marts** for departments.

---

## 5.4 Operational vs analytical “source of truth”

- **OLTP** is usually **source of truth** for **current state** (what privileges does this user have **now**?).  
- **OLAP** is **source of truth** for **audited history** of **reporting definitions**—if ETL is wrong, reports lie even when OLTP is correct.

**Staff:** Define **data contracts** (schemas, SLAs) between layers.

---

## 5.5 CQRS (name the pattern)

**Command Query Responsibility Segregation:** **Write model** optimized for **invariants**; **read model** optimized for **screens**—often **eventually consistent**.

Not always two databases; sometimes **projections** + **cache**. Modeling impact: **you accept** duplicated shapes **on purpose**.

---

## 5.6 What you should be able to say

- “I’d keep **OLTP normalized** and **project** to analytics—never run **heavy scans** on checkout primary without isolation.”  
- “**CQRS** means **multiple models**; I need **replay** or **CDC** to rebuild reads if they drift.”

**Next:** [Dimensional modeling](./06-dimensional-modeling-star-schema.md).

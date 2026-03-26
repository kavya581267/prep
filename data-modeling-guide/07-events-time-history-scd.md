# 7 — Events, time, and history (SCD)

**Goal:** Model **what happened when**, **how state changed**, and **what we knew at decision time**—without corrupting reporting.

---

## 7.1 Events vs state tables

| Style | Best for |
|-------|-----------|
| **Event log** (append-only) | Audit, replay, streaming pipelines, “every change is a fact” |
| **Current state** (mutable row) | Fast reads of “where is my order now?” |

**Common pattern:** **OLTP** holds **current state**; **outbox/events** emit **changes**; **warehouse** stores **history** (SCD or event store projections).

---

## 7.2 Slowly changing dimensions (SCD) — types

| Type | Behavior | Example |
|------|----------|---------|
| **Type 1** | **Overwrite**; no history | Fix typo in campaign name |
| **Type 2** | **New row** with **valid_from** / **valid_to** (or `is_current`) | Campaign renamed for compliance; reports must use **as-of** join |
| **Type 3** | Add **limited** “previous value” columns | Rare; “old region / new region” only |

**Interview favorite:** **Type 2** for **dimensions** that must be **time-traveled** in BI (“revenue by **old** vs **new** segment”).

---

## 7.3 As-of joins

Reporting question: “What did we **call** this segment **on the day** of the impression?”

**Need:** Dimension row where `event_date` ∈ [`valid_from`, `valid_to`]. Gets **expensive** if done wrong—**date keys** and **partitioning** help.

---

## 7.4 Bi-temporal (advanced name drop)

**Valid time** — when the fact was true in the **real world**.  
**Transaction time** — when the **system** recorded it.

**Use:** Corrections arrive late; regulators ask “what did you **believe** on March 1?” Two timelines—heavyweight; use when **domain requires**.

---

## 7.5 Snapshots for “balance as of date”

Event streams are great; some finance questions want **end-of-day balance**. Materialize **daily snapshot** tables: simpler for many SQL users.

---

## 7.6 Idempotency and event design

Events should carry **`event_id`** (unique) or **natural dedupe key** so **at-least-once** delivery doesn’t double-count.

**Example keys:**

- `impression_id` from ad server  
- `(log_batch_id, offset)` from pipeline  

---

## 7.7 What you should be able to say

- “**Mutable OLTP** + **immutable events** + **Type 2 dims** is a standard analytics story.”  
- “I’d never lose **price at order time**—that’s a **fact** or **snapshot**, not live product join.”

**Next:** [NoSQL & alternative shapes](./08-nosql-and-alternative-shapes.md).

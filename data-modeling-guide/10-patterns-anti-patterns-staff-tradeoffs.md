# 10 — Patterns, anti-patterns, Staff tradeoffs

**Goal:** A **catalog** plus **one full worked design review** so you can narrate tradeoffs in interviews and real reviews.

---

## 10.1 High-value patterns (with quick examples)

| Pattern | What it solves | Example |
|---------|----------------|---------|
| **Surrogate PK + unique business key** | Stable joins; integrations still find rows | `users.id` UUID + `UNIQUE(email)` |
| **Junction + edge columns** | M:N with **pair-specific** facts | `campaign_placement(bid_floor, priority)` |
| **Explicit fact grain** | No double-count in BI | “One row = day × campaign × country” |
| **Type 2 SCD** | Historical reporting after resegmentation | New `campaign_key` when `segment` changes |
| **Event + projection** | Fast reads + audit | Kafka `OrderPaid` → `order_summary` table |
| **Outbox** | Same txn for **DB + message** | Insert order + outbox row; publisher tails outbox |
| **Idempotency key** | Retries don’t double-charge | `UNIQUE(tenant_id, idempotency_key)` on payments |

---

## 10.2 Anti-patterns (symptoms and fixes)

| Smell | What goes wrong | Direction |
|-------|------------------|-----------|
| **God table** | 80 nullable columns, nobody owns it | Split by **bounded context** or **vertical** slice |
| **JSON as only schema** | BI can’t query; validation only in app | **Contract** + **flatten** hot paths to columns |
| **Polymorphic FK everywhere** | No FK constraint; join explosion | Separate tables per type or **typed** union |
| **Cache = source of truth** | Drift after crash | **DB authoritative**; cache **expirable** |
| **Global counter row** | `UPDATE hits SET c=c+1` hot row | **Shard** counters, use **approx**, or **stream** |
| **Reporting on OLTP primary** | Latency spikes | **Replica** with guardrails or **warehouse** |

---

## 10.3 Worked example — design review narrative

**Prompt:** “We need **campaigns** that target **placements**, track **daily spend**, and let finance report **by historical segment** after **reclassification**.”

**Entities:**

- `advertisers`, `campaigns` (OLTP)  
- `campaign_placement` junction with `bid_floor_usd`, `priority`  
- warehouse: `fct_spend_daily` at grain **day × campaign_key × geo_key**  
- `dim_campaign_scd` **Type 2** for `segment`

**Keys:**

- OLTP `campaigns.id` UUID **surrogate**; optional `external_trafficker_id` **unique** for integrations.

**Flow:**

1. **Operational** updates change **current** campaign row (name, status).  
2. **Segment** change → **close** old SCD row (`valid_to`), **open** new with new `campaign_key`.  
3. **ETL** maps spend facts to **correct** `campaign_key` by **date** using **as-of** join.

**Failure modes called out:**

- Forgetting **junction** edge fields → people put `bid_floor` on **campaign** (wrong if per-placement).  
- Type 1 overwrite on **segment** → **March** revenue suddenly **moves** in refreshed reports.  
- Missing **`tenant_id`** on SaaS → **cross-tenant** leak.

**That** is a **Staff-level** story: requirements → schema → **time** → **failure modes**.

---

## 10.4 Worked example 2 — inventory and overselling (correctness under concurrency)

**Requirements:** `products` has `stock_qty`. Many concurrent checkouts must **not** sell below zero.

**Naive model:** `UPDATE products SET stock_qty = stock_qty - :q WHERE id = :id` — two txs can both read **1**, both subtract, **-1** stock.

**Better patterns:**

1. **`SELECT … FOR UPDATE`** on **`products`** row in **same txn** as **`order_lines`** insert—**serializes** writers (simple; **hot SKU** bottleneck).  
2. **`inventory_reservation` rows** — reserve on checkout, **confirm** on payment; **`SUM(reserved)`** vs **`stock`** with constraints.  
3. **Optimistic** versioning: `UPDATE … WHERE id AND version = :v` / **`stock_qty >= :q`**—retry on conflict.

**Model lesson:** Correctness is **schema + transaction**, not schema alone.

---

## 10.5 Data-quality gates (mart SLO sketch)

| Check | Example |
|-------|---------|
| **Reconciliation** | `SUM(fct_orders_daily.revenue)` within **0.1%** of OLTP **for same date range** |
| **Uniqueness** | Zero **duplicate** `(grain keys)` in fact |
| **Freshness** | **MAX(partition_ts)** < **2h** old for daily mart |
| **Orphans** | No **FK** in mart `campaign_key` missing in `dim_campaign` |

Run in **CI** for **staging** and as **monitors** in **prod**.

---

## 10.6 Ownership (RACI hint)

| Artifact | Accountable |
|----------|-------------|
| OLTP **schema** | Service team |
| **Event** **schema** | Producer + **registry** admin |
| **Mart grain** definitions | Analytics / **data product** owner |
| **PII** classification | Security / privacy |

Ambiguous ownership → **schema drift** and **silent** metric changes.

---

## 10.7 Staff interview prompts — crisp answers

**“How do you model X?”**  
Start with **read/write paths** and **consistency**; then **entities**, **keys**, **cardinality**, **OLTP vs mart**.

**“OLTP vs warehouse?”**  
Source of truth per layer; **CDC**; **tests** on marts; never heavy **aggregation** on checkout DB.

**“Multi-region writes?”**  
**ID** generation without collision; **avoid** distributed FK unless necessary; **conflict** strategy (last-write-wins vs CRDT vs single leader per entity).

**“GDPR erase?”**  
**PII inventory**; **anonymize** facts vs **delete**; **propagate** to warehouse and **backups** per legal—not only OLTP.

---

## 10.8 Tradeoff matrix

| Optimize for… | Often accept… |
|---------------|----------------|
| **Join flexibility** | More **join cost** on huge scans unless warehouse |
| **Write throughput** | **Async** read models |
| **Agile schema** | **Nullable** phases, **registry**, **expand–contract** |
| **Tenant isolation** | **Higher** cost per **enterprise** tier |
| **BI clarity** | **Duplication** in facts/dims |

---

## 10.9 Checklist before signing off

1. **Who** queries (service / BI / ML) and **latency** SLO?  
2. **Grain** of facts documented?  
3. **PK/FK/UNIQUE** match business **rules**?  
4. **Migration** path for next field—**expand–contract** steps?  
5. **PII** map and **retention**?  
6. **Hot partitions** / **hot keys**?  
7. **Replay** path if projection wrong?  
8. **Concurrency** on hot rows (inventory, balances, global counters)?

---

## 10.10 Where to go deeper

- Kimball — dimensional modeling  
- Kleppmann — logs, stream processing, consistency  
- Files **01–09** in this folder for the **progressive** base

---

You now have both **reference patterns** and a **full** narrated example tying **OLTP**, **junction**, **SCD**, and **facts** together.

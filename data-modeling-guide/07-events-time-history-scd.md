# 7 — Events, time, and history (SCD)

**Goal:** Model **immutable events**, **mutable current state**, and **slowly changing dimensions** so reporting answers “what happened?” and “what did we **know** then?” correctly.

---

## 7.1 Events vs state tables

### State table (mutable)

`orders.status` moves `pending` → `paid` → `shipped`. **One row**; you **overwrite** `status`.

**Good for:** “What is the state **now**?” (APIs, fulfillment screens)

### Event log (append-only)

Each change is a **new row** (or Kafka message):

| event_id | order_id | occurred_at | type | payload |
|----------|----------|-------------|------|---------|
| e1 | o1 | 2025-03-20T10:00:00Z | OrderCreated | `{"customer_id":"c1"}` |
| e2 | o1 | 2025-03-20T10:01:05Z | OrderPaid | `{"payment_id":"pay_9"}` |
| e3 | o1 | 2025-03-21T08:00:00Z | OrderShipped | `{"tracking":"1Z..."}` |

**Good for:** Audit, **replay**, “timeline for support,” rebuilding projections.

**Common architecture:** OLTP holds **current state** + **outbox** → **stream** → warehouse **stores raw events** + **curated facts**.

---

## 7.2 Slowly changing dimensions (SCD) — worked example

**Scenario:** `dim_campaign` includes **`segment`** (brand / performance / DR). Finance needs reports **as of** each month under **old** segment names after **reclassification**.

### Type 1 — overwrite (no history)

```sql
UPDATE dim_campaign SET segment = 'Performance'
WHERE campaign_key = 101;
```

All **past** reports that **refresh** now show the new segment—often **wrong** for historical close.

### Type 2 — new row per version

| campaign_key | campaign_id_natural | campaign_name | segment | valid_from | valid_to | is_current |
|--------------|---------------------|---------------|---------|------------|----------|------------|
| 101 | CMP-9001 | Spring Promo | Brand | 2025-01-01 | 2025-03-15 | 0 |
| 198 | CMP-9001 | Spring Promo | Performance | 2025-03-16 | 9999-12-31 | 1 |

Same **natural** `CMP-9001`, **two** surrogate keys. **New** facts after Mar 16 use `198`; **old** facts keep `101`—historical revenue still rolls up under **Brand**.

### Type 3 — “previous value” column (limited)

Rare; e.g. `current_region`, `previous_region` only—**two** time steps max.

---

## 7.3 As-of join (pattern in SQL)

**Question:** For each impression on **2025-03-10**, what **segment** did **`CMP-9001`** belong to?

You need the dimension row where **`event_date`** ∈ [`valid_from`, `valid_to`]:

```sql
SELECT i.impression_date, i.campaign_id_natural, d.segment
FROM fct_impressions i
JOIN dim_campaign_scd d
  ON d.campaign_id_natural = i.campaign_id_natural
 AND i.impression_date >= d.valid_from
 AND i.impression_date <  d.valid_to;
```

**Performance:** **Index** `(campaign_id_natural, valid_from, valid_to)` or **current** snapshot + **point-in-time** tables for heavy workloads.

---

## 7.4 Bi-temporal (two timelines) — example sketch

**Subscription** “valid in real life” Jan 1–Dec 31, but **entered** wrong and **corrected** Feb 5.

| valid_from | valid_to | transaction_from | transaction_to | amount_usd |
|------------|----------|------------------|----------------|------------|
| 2025-01-01 | 2025-12-31 | 2025-01-10 | 2025-02-05 | 99 |
| 2025-01-01 | 2025-12-31 | 2025-02-05 | 9999-12-31 | 129 |

- **Valid time** — contractual truth.  
- **Transaction time** — what the **database believed** when.

Used in **finance** and **insurance**; heavy for simple apps.

---

## 7.5 Snapshots — “balance as of date”

**Events** are great; accountants often want **`account_balance_snapshot`**:

| as_of_date | account_id | balance_usd |
|------------|------------|-------------|
| 2025-03-31 | a1 | 10_500.00 |

Built nightly from **ledger events** or **closing process**. Easier for **Excel-era** consumers than full event replay.

---

## 7.6 Idempotency in events (concrete)

**At-least-once** Kafka delivery means duplicates. Your **fact loader** should **dedupe**.

**Message:**

```json
{
  "event_id": "imp_9f3c2a1b-…",
  "impression_id": "imp_77001",
  "campaign_id": "CMP-9001",
  "occurred_at": "2025-03-20T14:22:01Z",
  "cost_usd": 0.012
}
```

**Loader:** `INSERT INTO raw_impressions … ON CONFLICT (event_id) DO NOTHING` (or merge in warehouse with **unique** on `event_id`).

**Natural key** if no UUID: `(log_file_id, line_offset)` from batch ingest.

---

## 7.7 Price snapshot (ties to file `01`–`03`)

Never join **`order_lines`** to **live** `products.price` for **historical revenue**. The **line** carries **`unit_price_usd` at purchase**; that is the **business fact** for the order.

---

## 7.8 Outbox pattern — DDL sketch

Same **transaction** persists **business row** + **outbox message** so you never “updated DB but **lost** the Kafka message.”

```sql
CREATE TABLE outbox (
  id               bigserial PRIMARY KEY,
  aggregate_type  text NOT NULL,
  aggregate_id    text NOT NULL,
  event_type      text NOT NULL,
  payload_json    jsonb NOT NULL,
  created_at      timestamptz NOT NULL DEFAULT now(),
  published_at    timestamptz NULL
);
CREATE INDEX outbox_unpublished ON outbox (id) WHERE published_at IS NULL;
```

**Publisher** (separate process): `SELECT … FOR UPDATE SKIP LOCKED`, publish, set **`published_at`**.

---

## 7.9 Append-only event store (narrow table)

| position | stream_id | event_type | schema_version | payload | created_at |
|----------|-----------|------------|----------------|---------|------------|
| 1 | order.o1 | OrderCreated | 1 | `{"customer_id":"c1"}` | … |
| 2 | order.o1 | OrderPaid | 1 | `{"payment_id":"pay_9"}` | … |

**Replay:** scan by **`stream_id`** order. **Snapshot** caches current state for **fast read** (event-sourcing style).

---

## 7.10 Late-arriving facts vs Type 2 dimensions

**Problem:** Impression **fact** for **2025-03-10** lands in the warehouse on **March 20**; **`dim_campaign_scd`** was **resegmented** March 15—attach **old** key 101 or **new** key 198?

**Policies:**

- **Event timestamp wins:** map using **impression_date** → **as-of** join (correct for “revenue under labels **as of** impression day”).  
- **Load / current wins** (rare): only if the business explicitly wants **latest** labels on backfill.

Document the choice on the **mart** README—this is where **engineering** and **finance** often disagree.

---

## 7.11 CDC vs domain events

| Mechanism | Carries | Consumer builds |
|-----------|---------|-----------------|
| **CDC** | **Row** before/after | Mirrors, **SCD** from diffs |
| **Domain event** | **`OrderPlaced`** intent | projections, integrations |

**Both** can exist: **CDC** for warehouse **freshness**, **events** for **service decoupling**. Do not confuse **row snapshots** with **business events** when defining **history**.

---

## 7.12 What you should be able to say

- “**Type 2 SCD** when history must **split** by attribute change; **Type 1** for typos only.”  
- “Events need **stable ids** for **idempotent** ingest; **as-of joins** need **valid_from/to**.”  
- “**Outbox** couples DB + broker; **late facts** need a written policy vs **Type 2** keys.”

**Next:** [NoSQL & alternative shapes](./08-nosql-and-alternative-shapes.md).

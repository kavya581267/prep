# 8 — NoSQL & alternative shapes

**Goal:** See **document**, **wide-column**, **key-value**, and **search** models with **JSON / CQL-shaped** examples and **when** they beat relational tables.

---

## 8.1 When relational still wins

You need **ad-hoc joins** (“customers who ordered product X **and** returned Y”), **strong FK constraints**, and **complex reporting** across many entities—**PostgreSQL** is often the right **default**.

---

## 8.2 Document store — user profile aggregate

**Access pattern:** Load **full profile** for `user_id` in **one** read for the mobile app.

```json
{
  "_id": "user_8a7f6e",
  "email": "ada@example.com",
  "created_at": "2025-01-10T08:00:00Z",
  "preferences": {
    "theme": "dark",
    "notifications": { "email": true, "push": false }
  },
  "addresses": [
    {
      "type": "home",
      "city": "San Francisco",
      "postal_code": "94107",
      "is_default": true
    }
  ],
  "loyalty_tier": "gold"
}
```

**Rules:**

- Cap **`addresses`** length in **app** or validation—**unbounded arrays** hurt document size and **partial updates**.  
- If “list all users in ZIP 94107” is **hot**, a **relational** side table or **secondary index** per DB matters; don’t assume one document answers every query.

**Cross-document reference:** `order` documents might store `customer_id: "user_8a7f6e"` **without** DB-enforced FK—**eventual** consistency and **orphan** checks are on you.

---

## 8.3 Wide-column (Cassandra-style) — time-ordered events

**Access pattern:** “All **impression** records for **`campaign_id = CMP-1`** between **t1** and **t2**,” high write volume.

**Partition key** = `campaign_id` — all events for that campaign **co-locate** on replicas.  
**Clustering key** = `event_time` — sorted **within** partition.

Illustrative **CQL**:

```sql
CREATE TABLE impression_events (
  campaign_id text,
  event_time timestamp,
  impression_id text,
  placement_id text,
  cost_usd decimal,
  PRIMARY KEY (campaign_id, event_time, impression_id)
) WITH CLUSTERING ORDER BY (event_time ASC);
```

**Query that works:**

```sql
SELECT * FROM impression_events
WHERE campaign_id = 'CMP-1'
  AND event_time >= '2025-03-20 00:00'
  AND event_time <  '2025-03-21 00:00';
```

**Query that fails efficiently (it doesn’t):** `WHERE placement_id = 'pl_xyz'` without a **secondary index**, **materialized view**, or **duplicate table** keyed by placement becomes a **cluster-wide** scan.

**Modeling lesson:** **Query-first**: list **exact** reads/writes, then choose **partition key**.

---

## 8.4 Key-value — session and idempotency

**Key:** `session:8a7f6e`  
**Value:** JSON blob or protobuf, **TTL** 24h.

**Key:** `idempotency:payment:req_99281`  
**Value:** `{"status":"completed","charge_id":"ch_123"}` — second POST with same key **returns** cached outcome.

Not for **`JOIN`**; for **O(1) cache semantics**.

---

## 8.5 Graph — “who introduced whom” (sketch)

**Nodes** `Person`, **edge** `KNOWS` with `since` property.

**Query:** “Degrees of separation from **Ada** to **Bob**” — relational **recursive CTE** works for **moderate** depth; **Neo4j**/TigerGraph win for **long** paths and **frequent** graph analytics.

**Cost:** Another **datastore** to operate; justify with **unbounded** traversals.

---

## 8.6 Search (Elasticsearch) — product catalog

**Document** per SKU with **analyzed** `title`, **facets** `category`, `brand`, **sort** `price`.

**Pattern:** **PostgreSQL** = **source of truth**; **async** indexer on **change**; **reconcile** job if queue missed events.

**Don’t:** Use ES as **only** ledger for money without a strong **relational** backup.

---

## 8.7 Bounded contexts (microservices)

**Service A** — `Customer` with `loyalty_points`.  
**Service B** — `BillingAccount` with `invoice_email`.

Both say “customer” in English but **different tables**—integrate with **`customer_id`** + **events** (`CustomerMerged`, `EmailUpdated`), not **one shared** physical schema.

---

## 8.8 DynamoDB-style single-table design (mental model)

**One table** holds many entity types with **composite key** `PK` / `SK`:

| PK | SK | attrs |
|----|-----|--------|
| `USER#abc` | `PROFILE` | email, name |
| `USER#abc` | `ORDER#o1` | placed_at, total |
| `ORDER#o1` | `LINE#1` | product_id, qty |

**Query:** Get user + orders in **few** `Query` calls by **partition key** `USER#abc`.

**Cost:** **No** ad-hoc SQL; **access pattern** is **designed in** upfront; **migrations** are **painful**.

---

## 8.9 MongoDB: `$lookup` vs embedding tradeoff

**Embedded** addresses: one read; hard to query “**all users in ZIP 94107**” without **multikey** index.

**Normalized** `addresses` collection + `user_id`: two reads or **`$lookup`**—**join-like** latency and **operator** cost.

Rule: **embed** if **bounded** and **always** loaded with parent; **reference** if **queried independently** or **unbounded**.

---

## 8.10 Cassandra: tombstones and TTL

Deletes write **tombstones**; **TTL** on rows for **session** data avoids **manual** delete. Model **expiration** in the **schema** for GDPR **ephemeral** caches.

---

## 8.11 Lightweight transactions (LWT) / compare-and-swap

Reserve **inventory** or **counter** with **`IF`** conditions—**linearizable** small scope, **expensive** vs normal write. Relational **`SELECT FOR UPDATE`** is the cousin.

---

## 8.12 What you should be able to say

- “**Document** when **aggregate** = **unit of read**; watch **array bounds**.”  
- “**Cassandra**: partition key = **co-access** group; everything else is **secondary** design.”  
- “**Single-table** NoSQL trades **SQL flexibility** for **known access paths**; **`$lookup`** costs like joins.”

**Next:** [Evolution & multi-tenancy](./09-evolution-versioning-multitenancy.md).

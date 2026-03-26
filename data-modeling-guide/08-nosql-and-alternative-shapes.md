# 8 — NoSQL & alternative shapes

**Goal:** Choose **document**, **wide-column**, and **graph** models when they match **access patterns**—not because SQL is “boring.”

---

## 8.1 When relational still wins

**Ad-hoc joins**, **strong constraints**, **complex reporting** across many entities—**PostgreSQL** and friends are hard to beat.

---

## 8.2 Document stores (e.g. MongoDB-style)

**Good for:** **Aggregate** = **one unit of consistency** (user profile + preferences + flags) loaded by **`user_id`** in **one round trip**.

**Watch:** **Embeddings** that grow **without bound** (unbounded arrays); **cross-document** transactions are **possible** but **costly**; **reporting** often still lands in SQL elsewhere.

**Example:** `users` document with embedded `addresses[]` capped by business rule; or separate `addresses` collection with `user_id` index if **many** and **queried independently**.

---

## 8.3 Wide-column (e.g. Cassandra-style)

**Good for:** **Known** query patterns: `WHERE pk = X AND ck IN range`—**partition key** design is **the** modeling task.

**Anti-pattern:** Need **efficient** “give me all rows where secondary_column = Y” without proper **secondary index / MV** strategy—becomes full **cluster** scan.

**Example:** `(campaign_id, event_time)` for **time-ordered** events per campaign—supports **partition** read + **time slice**.

---

## 8.4 Key-value

**Good for:** **Cache**, **sessions**, **feature flags**, **idempotency** keys. Rarely your **only** source of structured truth for complex domains.

---

## 8.5 Graph databases

**Good for:** **Deep** relationship queries—degrees of separation, permissions on nested groups, fraud rings.

**Cost:** **Operational** complexity; not always needed if depth is **fixed** (join twice, not **unbounded** traversals).

---

## 8.6 Search engines (Elasticsearch, etc.)

**Good for:** **Full-text**, **facets**, **ranking**. **Not** default **OLTP**—**eventual consistency**, **merge** semantics, **double-write** risk.

**Pattern:** **DB of record** + **async index**; reconcile on failure.

---

## 8.7 Bounded contexts (DDD hint)

Different services may **legitimately** model the “same” real-world noun **differently**. **Integration** via **events** and **API contracts**, not **one global schema**.

---

## 8.8 What you should be able to say

- “I’d choose **document** when **aggregate boundary** matches **UI/API** boundary.”  
- “**Wide-column** starts with **query-first partition** design, not entity ER diagrams alone.”

**Next:** [Evolution & multi-tenancy](./09-evolution-versioning-multitenancy.md).

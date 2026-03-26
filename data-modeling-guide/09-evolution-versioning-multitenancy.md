# 9 — Evolution, versioning, multi-tenancy

**Goal:** Change schemas **without** wrecking deploys; isolate **tenants**; keep **integrations** stable.

---

## 9.1 Expand–contract migrations

**Expand:** Add **nullable** column or **new** table; **deploy** code that **writes** both old and new (or backfills async).  
**Contract:** Stop reading old; **drop** column/table after **safe**.

**Avoid:** “Rename column in place” without a **migration** story in **rolling** deploys—two versions of app run at once.

---

## 9.2 Backfills

Large table **UPDATE** can **lock** and **hurt** replicas.

**Patterns:** Chunked updates by PK range; **background** workers; **dual-write** then **verify** counts; **ETL** job in warehouse for **analytics-only** columns.

---

## 9.3 Schema registry (events & microservices)

For **Avro/Protobuf/JSON Schema**: **backward/forward** compatibility checks in **CI**; **versioned** subjects; **consumers** don’t break on **new** optional fields.

---

## 9.4 Multi-tenancy strategies

| Strategy | Isolation | Cost |
|----------|-----------|------|
| **Shared DB, shared schema** + `tenant_id` on rows | Weakest isolation; simplest ops | **RLS** or strict app checks mandatory |
| **Shared DB, schema per tenant** | Better logical isolation | Many schemas to migrate |
| **Database per tenant** | Strongest | Ops explosion at scale |

**Indexing:** `(tenant_id, …)` **leading** columns for **partition pruning** and **locality**.

**Staff:** **Noisy neighbor** (one tenant’s query kills shared **cluster**) → **rate limits**, **resource groups**, or **silos** for **VIP** tenants.

---

## 9.5 API versioning vs data versioning

**API:** `/v2/customers` can **map** to new columns while **v1** reads **compat** view.  
**Data:** **Deprecation** timeline; **feature flags** for **migrating** writers.

---

## 9.6 Soft delete vs hard delete

**Soft delete:** `deleted_at`; helps **support** and **replay**; complicates **unique** constraints (`UNIQUE WHERE deleted_at IS NULL`).

**Hard delete:** Cleaner; may violate **retention** laws if you remove **too early**—**legal** owns policy.

---

## 9.7 What you should be able to say

- “**Rolling deploys** force **expand–contract**; I’d never assume one version in prod.”  
- “**Tenant_id** everywhere is **cheap** until **compliance** demands **hard** isolation—then **shard** or **silos**.”

**Next:** [Patterns & Staff tradeoffs](./10-patterns-anti-patterns-staff-tradeoffs.md).

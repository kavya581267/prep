# 9 — Evolution, versioning, multi-tenancy

**Goal:** Change **schemas** safely during **rolling deploys**, version **event contracts**, and model **tenants** without **silent** data leaks.

---

## 9.1 Expand–contract (week-by-week story)

**Requirement:** Rename `users.phone` → `users.mobile_phone` and switch to **E.164** format.

### Week 1 — Expand

```sql
ALTER TABLE users ADD COLUMN mobile_phone text NULL;
```

Deploy app **version 2**: on signup/update, **write** both `phone` (legacy) and `mobile_phone` (new). **Read** still prefers legacy if new is null.

### Week 2 — Backfill

```sql
-- Batch job: chunked by id
UPDATE users SET mobile_phone = normalize_e164(phone)
WHERE mobile_phone IS NULL AND phone IS NOT NULL;
```

**Verify:** counts + sample checks.

### Week 3 — Flip reads

Deploy **version 3**: read **`mobile_phone`** only; stop writing `phone`.

### Week 4 — Contract

```sql
ALTER TABLE users DROP COLUMN phone;
```

**Never** “rename in place” on Friday if **v1** pods still **SELECT** `phone` Saturday morning—they **500**.

---

## 9.2 Backfills at scale

**Problem:** `ten million` rows, `UPDATE` sets **new** column.

**Tactics:**

- **Chunk** `WHERE id BETWEEN … AND …` or keyset pagination  
- **Throttle** to protect replication lag  
- **Dual-write** new table + **cutover** for **huge** rewrites (safer than mega-ALTER on some engines)

Analytics-only columns can be **filled in warehouse** first, then **promoted** to OLTP if needed.

---

## 9.3 Schema registry (events)

**Avro** subject `OrderCreated` **v3** adds **optional** field `delivery_window`:

```json
{
  "type": "record",
  "name": "OrderCreated",
  "fields": [
    { "name": "order_id", "type": "string" },
    { "name": "customer_id", "type": "string" },
    { "name": "delivery_window", "type": ["null", "string"], "default": null }
  ]
}
```

**CI rule:** **BACKWARD** compatible—old consumers **ignore** `delivery_window`. **Forward** compatibility is stricter (old data + new reader)—test both.

---

## 9.4 Multi-tenancy — shared schema with `tenant_id`

**Table:**

```sql
CREATE TABLE projects (
  id          uuid PRIMARY KEY,
  tenant_id   uuid NOT NULL,
  name        text NOT NULL,
  created_at  timestamptz NOT NULL,
  UNIQUE (tenant_id, name)
);

CREATE INDEX projects_tenant_idx ON projects (tenant_id);
```

**Sample rows:**

| id | tenant_id | name |
|----|-----------|------|
| p1 | t_acme | Mobile app |
| p2 | t_acme | Data lake |
| p3 | t_globex | Intranet |

**Every query** from app code: `WHERE tenant_id = :current_tenant` — **tests** enforce this.

**Postgres RLS (sketch):**

```sql
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON projects
  USING (tenant_id = current_setting('app.tenant_id')::uuid);
```

Set `app.tenant_id` per **DB session** from auth middleware.

**Risk:** **Noisy neighbor** — one tenant runs giant report; consider **warehouses** per tier, **rate limits**, or **silos** for **enterprise** accounts.

---

## 9.5 Schema-per-tenant vs DB-per-tenant

| Model | When |
|-------|------|
| **Shared + tenant_id** | Default SaaS; simpler ops |
| **Schema per tenant** | Stronger isolation; painful **migrations × N** |
| **DB per tenant** | Regulated; **few** large tenants; ops cost high |

---

## 9.6 API vs data versioning

**API v1** exposes `phone`; **API v2** exposes `mobile_phone`. Same table, **mapping layer**.  
**Deprecate v1** with **timeline** and **metrics** on remaining traffic.

---

## 9.7 Soft delete and uniqueness

```sql
ALTER TABLE users ADD COLUMN deleted_at timestamptz NULL;

CREATE UNIQUE INDEX users_email_active ON users (email)
  WHERE deleted_at IS NULL;
```

**Allows** reusing email after account **closed**—product decision, not default.

---

## 9.8 Compatibility views (rolling deploys)

```sql
CREATE VIEW users_v1_compat AS
SELECT id, email, phone AS phone_legacy, mobile_phone
FROM users;
```

Old code reads **`users_v1_compat`**; new code reads **base table**—buys time before **drop**.

**Watch:** views **hide** performance traps (joins); prefer **simple** column aliases for migrations.

---

## 9.9 Column defaults and NOT NULL—order of operations

Adding **`NOT NULL`** without default **fails** if rows exist.

**Sequence:**

1. `ADD COLUMN foo TYPE NULL DEFAULT 'x'`  
2. Backfill any stragglers  
3. `ALTER COLUMN foo SET NOT NULL`  
4. Optionally **drop default** if app supplies values

---

## 9.10 Online DDL differences (operations reality)

**Postgres:** many alters **rewrite** tables—monitor **lock**; use **`pg_rewrite`**-aware tools for big tables.

**MySQL 8 / InnoDB:** **instant** `ADD COLUMN` in some cases; **`ALGORITHM=INPLACE`** where supported.

**Staff:** “We test migration **duration** and **replication lag** on a **clone** of prod size.”

---

## 9.11 Tenant migration (moving a whale to silo)

1. **Dual-write** to shared + dedicated DB.  
2. **Verify** diff counters (rows, checksum samples).  
3. **Cut read** traffic feature-flag per tenant.  
4. **Stop** dual-write; **archive** old rows after retention.

**Model:** `tenant.routing = shared|dedicated` in **control plane**.

---

## 9.12 What you should be able to say

- “**Expand–contract** because **two** binary versions run during deploy.”  
- “**tenant_id** on every row + **RLS** or **obsessive** code review; **unique** includes tenant.”  
- “**NOT NULL** comes **after** backfill; **compat views** buy time; **online DDL** is validated on **prod-sized** clones.”

**Next:** [Patterns & Staff tradeoffs](./10-patterns-anti-patterns-staff-tradeoffs.md).

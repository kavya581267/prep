# 2 — Keys, identifiers, uniqueness

**Goal:** Choose **primary keys**, **foreign keys**, and **uniqueness rules** so rows stay identifiable, relationships stay honest, and integrations don’t break when emails change.

---

## 2.1 Primary key (PK)

A **primary key** uniquely identifies **one row** in a table. In SQL you declare it with `PRIMARY KEY`; the DB rejects inserts that collide on that key.

**What you usually want:**

| Property | Meaning |
|----------|---------|
| **Uniqueness** | No two rows share the same PK value(s) |
| **Stability** | Values rarely change (FKs elsewhere point here) |
| **Non-null** | Every row has a PK |

**Example — bookstore `orders`:**

```sql
CREATE TABLE orders (
  id              uuid PRIMARY KEY,
  customer_id     uuid NOT NULL REFERENCES customers(id),
  placed_at       timestamptz NOT NULL,
  status          text NOT NULL,
  currency        text NOT NULL DEFAULT 'USD'
);
```

Here `id` is the **surrogate** PK; it has no business meaning beyond identity.

---

## 2.2 Foreign key (FK)

A **foreign key** column stores the **primary key of another row**, encoding “this row **belongs to** / **references** that row.”

**Example:** `orders.customer_id` → `customers.id`.

**With referential action (illustrative):**

```sql
CREATE TABLE orders (
  id              uuid PRIMARY KEY,
  customer_id     uuid NOT NULL
    REFERENCES customers(id)
    ON DELETE RESTRICT,
  ...
);
```

- **`RESTRICT`** — cannot delete a customer who still has orders (safe default for money).  
- **`CASCADE`** — delete orders when customer deleted (rare for paid orders).  
- **`SET NULL`** — only if `customer_id` is **nullable** and orphan orders are allowed.

**Why DB-enforced FK beats app-only checks:** A buggy batch job or manual SQL can’t accidentally create `customer_id` pointing to a missing row—if you enforce constraints.

---

## 2.3 Natural vs surrogate keys (worked comparison)

| Kind | Example | Pros | Cons |
|------|---------|------|------|
| **Natural** | Country code `DE`, ISBN | Human-readable in reports | Changes (email), long strings, PII exposure |
| **Surrogate** | `uuid`, `bigserial` | Stable, short join to APIs | Useless without a lookup |

**Scenario — `users` with email login:**

- **Bad as sole PK:** `PRIMARY KEY (email)` — when `bob@co.com` renames to `rob@co.com`, you either **update PK** (cascade pain) or **break** external references.  
- **Better:** `id uuid PRIMARY KEY`, `email text NOT NULL UNIQUE` — change email freely; **orders** reference `user_id`.

**When natural PK is fine:** `currency_code = 'USD'` in a 200-row reference table that never mutates keys.

**Staff pattern:** Public APIs return **opaque** `user_id` (UUID); **support tools** search by **email** (unique index); **analytics** may join on **hashed** id for privacy.

---

## 2.4 UUIDs, ULIDs, and integers (concrete tradeoffs)

**Random UUID v4**

- **Good:** Safe **merge** from many writers without collision; fine for **public** ids.  
- **Watch:** In **B-tree** indexes, random inserts can cause **worse page locality** than sequential ints on some engines (often fine at moderate scale).

**Time-sortable (ULID, Snowflake ID, KSUID)**

- **Good:** Rough **time ordering** in id string; can help **range scans** on “recent rows.”  
- **Watch:** Ids are **guessable** in sequence—don’t rely on them as **security**; pair with auth.

**`bigserial` / identity column**

- **Good:** Tight storage, fast joins in single-region monolith.  
- **Watch:** **Shard** or **multi-region** writers need a **sequence service** or Snowflake-style ids to avoid collisions.

**Tiny example — two tables keyed by UUID:**

```sql
CREATE TABLE customers (
  id    uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  email text NOT NULL UNIQUE
);

CREATE TABLE orders (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id uuid NOT NULL REFERENCES customers(id)
);
CREATE INDEX orders_customer_id_idx ON orders (customer_id);
```

---

## 2.5 Unique constraints beyond the primary key

The PK is **one** uniqueness rule; the business often needs **more**.

**Single-column unique — one email per account:**

```sql
ALTER TABLE users ADD CONSTRAINT users_email_key UNIQUE (email);
```

**Composite unique — idempotency for partner webhooks:**

Your partner sends `external_order_id` per tenant; the same string must not insert twice **for that tenant**:

```sql
CREATE UNIQUE INDEX orders_tenant_partner_idempotency
  ON orders (tenant_id, external_order_ref)
  WHERE external_order_ref IS NOT NULL;
```

(Postgres-style partial unique; **syntax varies** by database.)

**Effect:** Retry the same webhook with the same `(tenant_id, external_order_ref)` → **unique violation** → treat as success (idempotent consumer).

---

## 2.6 Composite primary keys (junction table)

**Enrollment** — one row per **(student, course)** pair:

```sql
CREATE TABLE enrollment (
  student_id   uuid NOT NULL REFERENCES students(id),
  course_id    uuid NOT NULL REFERENCES courses(id),
  enrolled_at  timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (student_id, course_id)
);
```

The **composite PK** guarantees you never store two rows for the same **pair**, which would double-count the relationship.

---

## 2.7 Worked mini-design — “soft delete + unique email”

**Rule:** Active users have unique email; **deleted** users can free the email for reuse.

Approach A — **partial unique index** (Postgres):

```sql
CREATE UNIQUE INDEX users_email_active_unique
  ON users (email)
  WHERE deleted_at IS NULL;
```

Approach B — append suffix on delete in app: `ada+deleted_20250325@` — ugly but works without partial indexes.

**Modeling lesson:** Uniqueness rules are **business rules**; they belong in the **schema** when the DB supports them.

---

## 2.8 Superkeys, candidate keys, alternate keys (terminology)

- **Superkey:** Any column set that **uniquely** identifies a row (`id` alone, or `(id, email)`—overkill).  
- **Candidate key:** **Minimal** superkey—remove any column and you **lose** uniqueness.  
- **Primary key:** The **one** candidate key you elect for FK references.  
- **Alternate key:** Other candidates enforced with **UNIQUE** (e.g. `email` when PK is `id`).

Interviewers sometimes use this vocabulary; **PK vs unique** is enough for most.

---

## 2.9 Indexing for FK lookups (covering indexes)

Child tables are read as **`WHERE parent_id = ?`**. You almost always want:

```sql
CREATE INDEX orders_customer_placed_idx
  ON orders (customer_id, placed_at DESC);
```

**Leading** `customer_id` matches the FK filter; **`placed_at`** supports “recent orders for this customer” without **sort** in memory.

**Covering index** (Postgres `INCLUDE`, SQL Server `INCLUDE`): add non-key columns to satisfy **`SELECT`** without heap lookup—use when one query dominates.

---

## 2.10 Surrogate leakage and API design

If URLs are `/users/42` with **sequential** ids, competitors **scrape** signups. **Mitigations:** opaque UUID in API; **rate limit**; **authorization** on every read.

If **natural** keys leak PII (`/invoice/2025-INV-bob@co.com`), use **opaque** `invoice_id`.

---

## 2.11 Deferred and deferrable FKs (when you must)

**Problem:** Circular insert—`departments.head_employee_id` → `employees`, but **`employees.department_id`** → `departments`.  

**Tactics:** insert with **NULL** head, then update; or use **`DEFERRABLE INITIALLY DEFERRED`** (Postgres) so FK checks run at **commit** after both rows exist.

Rare in greenfield; appears in **legacy** migrations.

---

## 2.12 CHECK constraints as mini data models

```sql
ALTER TABLE orders ADD CONSTRAINT orders_status_check
  CHECK (status IN ('pending', 'paid', 'shipped', 'refunded'));

ALTER TABLE order_lines ADD CONSTRAINT order_lines_qty_pos
  CHECK (qty > 0);
```

**Cross-column:** `CHECK (end_at > start_at)` on campaigns.

Keeps **invalid** states out even when **app** has a bug.

---

## 2.13 What you should be able to say

- “**PK** is stable row identity; **UNIQUE** encodes business rules; **FK** encodes relationships with **ON DELETE** chosen on purpose.”  
- “I use **surrogate PKs** for mutable entities and **natural keys** as **unique constraints** where integrations need them.”  
- “I **index FK columns** with leading keys that match filter + sort; I use **CHECK** for state machines and quantities.”

**Next:** [Normalization & denormalization](./03-normalization-and-denormalization.md).

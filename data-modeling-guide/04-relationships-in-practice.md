# 4 — Relationships in practice

**Goal:** Map **1:1**, **1:N**, and **M:N** to **tables**, **documents**, and **integrity** choices.

---

## 4.0 Minimum tables & how to define each cardinality (relational OLTP)

Use this as a **checklist** when turning an ER diagram into **DDL**. Counts below assume **one relationship** between two **entity types** in **third-normal** form (file `03`).

### At a glance

| Cardinality | **Minimum** tables | What you add to enforce the rule |
|-------------|-------------------|-----------------------------------|
| **1:N** | **2** | Parent: **PK**. Child: **FK → parent.PK** (usually `NOT NULL`). Index on FK for joins. |
| **1:1** | **1** *or* **2** | **1 table:** all attributes in one row. **2 tables:** **FK** on one side + **`UNIQUE(FK)`** so each parent ties to **at most one** child. |
| **M:N** | **3** | Table A: **PK**. Table B: **PK**. **Junction:** **PK (`a_id`, `b_id`)** (or surrogate PK + **`UNIQUE(a_id, b_id)`**), both **FKs** to A and B. |

**Why M:N needs three:** With only two tables you cannot list *which* A links to *which* B without **repeating** rows or **multi-valued** cells—either you break **1NF** or you lose **pair-level** facts (e.g. “priority of **this** campaign on **this** placement”).

---

### 4.0.1 One-to-many (1:N) — define it in four steps

**Minimum structure: 2 tables** — **one** (parent) and **many** (child).

1. **`parent`:** single-column **PK** (e.g. `parent.id`).  
2. **`child`:** own **PK** (e.g. `child.id`); add **`parent_id`** referencing `parent(id)`.  
3. **Rule:** If every child **must** have a parent → **`parent_id NOT NULL`** + **`FOREIGN KEY (parent_id) REFERENCES parent(id)`**.  
4. **Index** `child(parent_id)` for lookups: “all orders for this customer.”

```text
-- Minimum viable 1:N
parent   (id PK, ...)
child    (id PK, parent_id NOT NULL REFERENCES parent(id), ...)
-- index: CREATE INDEX ON child(parent_id);
```

**Optional:** `ON DELETE RESTRICT | CASCADE | SET NULL` on the FK to match **business** (file `09` soft-delete may prefer `RESTRICT` + `deleted_at` on parent).

**Not counted as “fewer tables” for modeling:** stuffing many children’s data into **JSON arrays** on the parent—you still **conceptually** have a 1:N; you’ve only **collapsed** storage and usually pay on **query** and **integrity**.

---

### 4.0.2 One-to-one (1:1) — one table vs two tables

You need **either 1 or 2** tables—not zero, not three for a **single** 1:1 between two entity types.

#### Option A — **1 table** (minimum: **1 table**)

**When:** Both sides **always** exist together, similar **lifecycle**, row isn’t **huge**, no **regulatory** need to isolate columns.

**Define:** One **PK**; all attributes as **columns** (nullable columns OK if **some** fields are optional).

```text
user_account (
  id PK,
  email NOT NULL,
  legal_name NULL,        -- optional "extension" fields inline
  tax_id_encrypted NULL
)
```

**Limit:** Wide rows + mixed security (PII next to public fields) can be wrong long-term.

#### Option B — **2 tables** (minimum: **2 tables**)

**When:** **Split** for size (BLOBs), **PCI/PII** isolation, **optional** “extension” (not every row has a profile), or **different** retention.

**Define:**

1. **Primary** table: `user(id PK, email, ...)`.  
2. **Secondary** table: FK to user, and **at most one** row per user.

**Two common DDL shapes:**

**Shape 1 — FK side uses same value as PK (strong 1:1)**

```text
user      (id PK, email, ...)
user_pii  (user_id PK REFERENCES user(id), legal_name, ...)  -- PK = FK ⇒ ≤1 row per user
```

**Shape 2 — surrogate PK on secondary, unique FK**

```text
user      (id PK, email, ...)
user_pii  (id PK, user_id NOT NULL UNIQUE REFERENCES user(id), ...)
```

**Critical constraint:** **`UNIQUE(user_id)`** (or **`user_id` as PK**)—without it, the DB allows **many** PII rows per user → relationship becomes **1:N** by mistake.

**Where to put the FK:** Usually on the **optional** or **sensitive** side (`user_pii.user_id → user`) so the core entity exists without the extension.

---

### 4.0.3 Many-to-many (M:N) — define it in five steps

**Minimum structure: 3 tables** — **entity A**, **entity B**, **junction** (link / associative).

1. **`a`:** `id PK`, …  
2. **`b`:** `id PK`, …  
3. **`a_b` (junction):**  
   - `a_id FK → a(id) NOT NULL`  
   - `b_id FK → b(id) NOT NULL`  
   - **`PRIMARY KEY (a_id, b_id)`** *or* surrogate `id PK` plus **`UNIQUE (a_id, b_id)`**  
4. **Indexes:** at least **PK/unique**; often add **`INDEX (b_id, a_id)`** if you query “all `a` for this `b`” heavily.  
5. **Edge attributes:** columns that describe **this pair** (e.g. `assigned_at`, `role_on_project`) live **on the junction**.

```text
student      (id PK, name, ...)
course       (id PK, title, ...)
enrollment   (
  student_id  NOT NULL REFERENCES student(id),
  course_id   NOT NULL REFERENCES course(id),
  enrolled_at NOT NULL,
  PRIMARY KEY (student_id, course_id)
)
```

**Variations:**

- **Surrogate PK** on `enrollment(id)` when ORMs or APIs want a single-column id—keep **`UNIQUE(student_id, course_id)`**.  
- **Soft delete** of a link: `removed_at` on junction + **partial unique** `WHERE removed_at IS NULL` if your DB supports it—otherwise **business rules** in app.

---

### 4.0.4 Quick validation questions

- **1:N:** Can multiple child rows share the same `parent_id`? If **yes**, schema matches **1:N**. If you need “only one child ever,” see **1:1**.  
- **1:1:** Is **`UNIQUE`** on the FK column enforced in the DB? If not, you may have accidentally built **1:N**.  
- **M:N:** Is there a **single row** per **(A, B)** pair in the junction? **PK/unique** on `(a_id, b_id)` enforces that.

---

## 4.1 One-to-many (1:N) — default pattern

**Pattern:** Parent table PK; child table has **FK** to parent.

```text
customers (id, ...)
orders (id, customer_id FK -> customers.id, ...)
```

**Delete behavior:** `ON DELETE RESTRICT` vs `CASCADE` vs `SET NULL`—**model the business rule** (never orphan paid orders; or soft-delete parent).

---

## 4.2 Many-to-many (M:N) — junction table

**Pattern:** Third table whose PK is usually **(a_id, b_id)** or a **surrogate** + **unique (a_id, b_id)**.

```text
campaigns (id, ...)
placements (id, ...)
campaign_placement (
  campaign_id FK,
  placement_id FK,
  -- optional: priority, deal terms, created_at
  PRIMARY KEY (campaign_id, placement_id)
)
```

**Payload on the edge:** Extra columns belong on the **junction** when they describe **this link** (“bid adjustment for this placement type”), not the whole campaign.

---

## 4.3 One-to-one (1:1) — split for clarity or security

**When:** Optional profile, **PCI** separation, **huge** rarely-used BLOB columns.

```text
users (id, email, ...)
user_pii (user_id PK/FK -> users.id, legal_name, tax_id_encrypted, ...)
```

Enforce **1:1** with `user_pii.user_id` as **PK** so each user has **at most one** row.

---

## 4.4 Optional / polymorphic references (danger)

**Polymorphic FK:** `comment.target_type + target_id` pointing to **either** posts **or** photos.

**Problems:** DB **can’t enforce** referential integrity without triggers/checks; queries get messy.

**Alternatives:**

- **Separate** comment tables per target type  
- **Single** “commentable” supertable (shared PK space—often awkward)  
- **Graph** store for true many-types-many-links  

**Staff:** If you use polymorphic, document **invariants** and **test** them ruthlessly.

---

## 4.5 Soft references

Some systems store `external_id` **without** FK to another service’s DB—**no FK constraint** across boundaries.

**Mitigations:** Async **reconciliation**, **idempotent** upserts, **tombstone** events when external entity disappears.

---

## 4.6 Self-referential relationships

**Org chart:** `employees.manager_id` → `employees.id`  
**Threaded comments:** `comments.parent_comment_id` → `comments.id`

Watch **cycles**, **depth limits**, and **path** queries—may need **closure table** or **graph** later.

---

## 4.7 What you should be able to say

- “**1:N** needs **two** tables minimum; **M:N** needs **three** (two entities + junction); **1:1** is **one merged table** or **two** with **UNIQUE** on the FK.”  
- “**M:N** always gets a **junction** in relational OLTP unless I collapse into documents with clear bounds.”  
- “**Polymorphic** FKs are a **last resort**; I’d prefer explicit tables or **event-first** modeling.”

**Next:** [OLTP vs OLAP](./05-oltp-vs-olap.md).

# 2 — Keys, identifiers, uniqueness

**Goal:** Pick **stable identifiers**, enforce **uniqueness** where the business requires it, and understand **natural** vs **surrogate** keys.

---

## 2.1 Primary key (PK)

A **primary key** uniquely identifies **one row** in a table. A table should have **exactly one** primary key constraint (possibly **composite**—multiple columns together).

**Properties you usually want:**

- **Uniqueness** — no two rows share the PK  
- **Stability** — rarely changes (changing PKs ripples through FKs)  
- **Non-null** — every row must have it  

---

## 2.2 Foreign key (FK)

A **foreign key** stores a reference to another row’s **primary key** (or unique key). It encodes the **relationship** at the database level.

**Example:** `orders.customer_id` → `customers.id`

**Why it matters:** The DB can enforce **referential integrity** (no orphan orders) if you enable constraints—application-only checks are easier to bypass.

---

## 2.3 Natural vs surrogate keys

| Kind | Definition | Example |
|------|------------|---------|
| **Natural key** | Meaningful in the real world | Country code `US`, ISBN (with caveats), government id (privacy!) |
| **Surrogate key** | Meaningless integer/UUID assigned by the system | `user_id = 918273645` |

**When surrogate wins:** Natural keys **change** (email edits), are **long**, or are **sensitive** to expose in URLs/logs.  
**When natural wins:** Small dimension tables, stable codes (`currency = USD`), regulatory reporting expects a **business key**.

**Staff pattern:** Expose **opaque IDs** externally (`uuid`); keep **business keys** as **unique constraints** for integration and support tools.

---

## 2.4 UUIDs and ordered IDs

**Random UUID (v4):** Great for **uniqueness across shards**; bad for **B-tree insert locality** in some databases (index fragmentation / hot pages).

**Time-sortable IDs (ULID, Snowflake-style):** Approximate **time ordering** and spread load better for some storage engines—at the cost of **predictability** (enumeration risk if exposed).

**Auto-increment integers:** Simple joins and storage; **coordination** across regions or shards requires **sequences** or **ID generation services**.

---

## 2.5 Unique constraints beyond PK

Not every uniqueness rule is the PK.

**Examples:**

- `users.email` **UNIQUE**  
- `(tenant_id, external_order_ref)` **UNIQUE** for multi-tenant ingestion  

**Partial uniqueness** (where supported): “one **default** address per customer” via unique index with `WHERE is_default`.

---

## 2.6 Composite keys

**Composite PK** on a junction table is common:

```text
campaign_placement (campaign_id, placement_id)  PK
```

Ensures you don’t duplicate the **same** link; both columns are FKs.

---

## 2.7 What you should be able to say

- “PK for **row identity**; **unique** constraints for **business** rules; **FK** for **relationships**.”  
- “I’d avoid using **PII** as PK; I’d use **surrogate** + **unique** on natural key when needed.”

**Next:** [Normalization & denormalization](./03-normalization-and-denormalization.md).

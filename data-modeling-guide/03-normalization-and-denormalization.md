# 3 — Normalization & denormalization

**Goal:** Recognize **redundancy** that causes **bugs**, know **1NF–3NF** with **before/after tables**, and know **when** production systems **denormalize** on purpose with a **clear contract**.

---

## 3.1 Why normalize?

**Normalization** splits tables so each fact lives in **one place**. When the same fact is copied in many rows, **updates** can miss a copy → **inconsistent** data.

**Tradeoff:** More tables → more **JOINs** on OLTP paths. Analytics and read models often **denormalize** again for speed (files `05`, `06`).

---

## 3.2 First normal form (1NF) — atomic values, no repeating groups

**Rule (practical):** One value per “cell”; don’t hide a **list of tags** in a comma string without a plan.

### Bad (violates 1NF spirit)

`orders` table:

| order_id | customer_id | tags |
|----------|-------------|------|
| o1 | c1 | `gift,urgent,vip` |

To filter “all urgent orders” you need string parsing; you can’t index `urgent` cleanly.

### Good

`order_tags`:

| order_id | tag |
|----------|-----|
| o1 | gift |
| o1 | urgent |
| o1 | vip |

**Query:** `SELECT order_id FROM order_tags WHERE tag = 'urgent'`.

**Exception:** Postgres `text[]` or JSON array **can** be 1NF if you treat the array as **one** typed value and your queries use **array ops**—still document the **access pattern**.

---

## 3.3 Second normal form (2NF) — no partial dependency on composite PK

**Only matters** when the PK is **composite** `(A, B)`.

**Rule:** Every **non-key** column must depend on **the whole** `(A, B)`, not just `A` or just `B`.

### Bad

Assume PK `(order_id, product_id)` on a line-like table:

| order_id | product_id | qty | product_title |
|----------|------------|-----|----------------|
| o1 | p900 | 1 | Algorithms 101 |
| o1 | p901 | 2 | Data Modeling |

`product_title` depends only on **`product_id`**, not on **`order_id`**. If “Algorithms 101” is renamed, you must update **every line row** — **update anomaly**.

### Good — split

`order_lines (order_id, product_id) PK`, `qty`, …  
`products (product_id) PK`, `title`, …

Join when you need the title:

```sql
SELECT ol.order_id, p.title, ol.qty
FROM order_lines ol
JOIN products p ON p.id = ol.product_id;
```

---

## 3.4 Third normal form (3NF) — no transitive dependency through non-key columns

**Rule:** Non-key columns must **not** depend on **other** non-key columns—only on the **PK**.

### Bad

`orders`:

| id | customer_id | customer_email | placed_at |
|----|-------------|----------------|-----------|
| o1 | c1 | ada@example.com | 2025-03-20 |

`customer_email` is determined by `customer_id`. If Ada updates her email, old orders show the **wrong** email unless you **backfill** every order—**update anomaly**. For **current** reporting, a JOIN to `customers` is correct.

### When denormalizing email on orders is **intentional**

If finance needs “**email we used for receipt** on that date,” store `customer_email_snapshot` **on purpose** and name it so people know it’s **historical**, not live profile.

---

## 3.5 Anomalies with a story

| Type | Story |
|------|--------|
| **Insert** | You can’t insert a `product` without an `order line` because you crammed products only inside line rows with weird nulls |
| **Update** | Product title duplicated on 50k lines; one bulk update fails halfway → half wrong titles |
| **Delete** | Deleting last line for a product **loses** the product definition if you stored product only embedded in lines |

Normalization **removes** redundancy that causes these when the redundancy came from **repeating the same fact**.

---

## 3.6 When teams denormalize (on purpose) — examples

| Situation | What you duplicate | Why | Guardrail |
|-----------|-------------------|-----|-----------|
| **Analytics mart** | Country name on fact table | Avoid join on billion-row scan | ETL rebuilds nightly; dimension is versioned |
| **Order line** | `unit_price`, `product_title_snapshot` | Legal / invoice “as sold” | Document: **not** live catalog |
| **Read model (CQRS)** | Customer name on “order summary” document | One round trip for mobile API | **Project** from events; replay if wrong |
| **Star schema** | Region inside `dim_geo` | Fewer joins for analysts | `dim_geo` is **the** place to update region names |

**Staff line:** “We denormalize only with a **named contract**: who **owns** freshness, how we **rebuild**.”

---

## 3.7 Quick exercise — normalize this

**Table `sales_clunky`:**

| sale_id | product_id | product_name | category_name | qty | customer_city | customer_city_pop |
|---------|------------|--------------|---------------|-----|---------------|---------------------|
| s1 | p1 | Shirt | Apparel | 2 | SF | 815000 |

**Issues:** `product_name`/`category_name` depend on `product_id`; `customer_city_pop` depends on city, not on `sale_id`.

**Normalized sketch:**

- `sales (id, product_id, qty, …)`  
- `products (id, name, category_id)`  
- `categories (id, name)`  
- `customers` + `addresses` (or `city_id` → `cities (id, name, population)`)

---

## 3.8 Boyce–Codd normal form (BCNF) — when 3NF is not enough

**3NF allows** a weird case: a **determinant** (something that uniquely determines another column) might not be a **candidate key**.

**Classic exam table** — rooms, subjects, teachers; **rule:** each teacher teaches **one** subject, but a subject can have **many** teachers; a room holds **at most one** subject’s class at a time (simplified):

| teacher | subject | room |
|---------|---------|------|
| Smith | Math | 101 |
| Jones | Math | 102 |
| Adams | Physics | 101 |

Here **`teacher → subject`** (functional dependency). **PK** could be `(teacher, room)` or you spot redundancy in **subject** repeating per teacher.

**BCNF:** Every determinant is a **candidate key**. If not, **split** tables so each FD is “key determines facts.”

Most production **OLTP** stops at **3NF** or **BCNF**—worth naming in deep interviews.

---

## 3.9 Fourth normal form (4NF) — multi-valued facts (sketch)

**Bad pattern:** One row stores **two unrelated lists**—“**skills**” and **languages**”—in one wide row without cross-product meaning → **join dependency** anomalies.

**Fix:** Separate **tables** for **`employee_skill`** and **`employee_language`**, or one **attribute** column with **type** discriminator.

If someone says “**4NF**,” they mean: **don’t** encode two **independent** M:N relationships in one table without a reason.

---

## 3.10 Solutions — normalize `sales_clunky` (full target schema)

**Starting clutter:**

| sale_id | product_id | product_name | category_name | qty | customer_city | customer_city_pop |

**Target (3NF-ish OLTP):**

```sql
CREATE TABLE categories (id uuid PRIMARY KEY, name text NOT NULL UNIQUE);
CREATE TABLE cities (id uuid PRIMARY KEY, name text NOT NULL, population bigint);
CREATE TABLE products (
  id uuid PRIMARY KEY,
  category_id uuid NOT NULL REFERENCES categories(id),
  name text NOT NULL
);
CREATE TABLE customers (id uuid PRIMARY KEY, city_id uuid REFERENCES cities(id));
CREATE TABLE sales (
  id uuid PRIMARY KEY,
  product_id uuid NOT NULL REFERENCES products(id),
  customer_id uuid NOT NULL REFERENCES customers(id),
  qty int NOT NULL CHECK (qty > 0)
);
```

**Migrate:** backfill **dimension** tables from distinct values, then **replace** `sales` FKs. **ETL** can populate `dim_product` / `dim_city` in parallel for analytics.

---

## 3.11 Denormalization “math” — when joins hurt

Suppose checkout reads **customer display name** on **every** order confirmation: **1 extra join** × **5000 QPS** × **avg 2ms** join cost = **capacity** problem.

**Options (in order of preference):**

1. **Cache** Redis with TTL (not source of truth).  
2. **Read replica** + query tuned covering index.  
3. **Snapshot** `customer_display_name` on `orders` **at place time** (document **snapshot**).  
4. **Materialized view** refreshed every N minutes for **non-real-time** pages.

Naming the **consistency** you accept beats silent duplication.

---

## 3.12 Normal forms beyond 4NF (names only)

**5NF / PJNF** decompose to remove **join dependencies** that aren’t obvious. Rare in hand-modeled OLTP; more **academic** than day-to-day.

---

## 3.13 What you should be able to say

- “**1NF** splits multi-valued junk; **2NF** removes facts that depend on half a composite key; **3NF** removes facts that transitively depend on non-keys.”  
- “**Snapshots** on order lines aren’t sloppy—they’re **auditable history**.”  
- “**BCNF/4NF** come up for interview depth; I’d split tables when **multiple independent** M:N facts get squashed together.”

**Next:** [Relationships in practice](./04-relationships-in-practice.md).

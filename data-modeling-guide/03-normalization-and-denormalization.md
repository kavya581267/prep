# 3 — Normalization & denormalization

**Goal:** Know **1NF–3NF**, what **anomalies** they prevent, and **when** production systems **denormalize** on purpose.

---

## 3.1 Why normalize?

**Normalization** splits data to **reduce redundancy** and **avoid update anomalies**—contradictions where two copies of the “same fact” disagree.

**Tradeoff:** More tables → more **joins**; sometimes you **duplicate** data for **read latency** or **query simplicity**.

---

## 3.2 First normal form (1NF)

**Roughly:** Atomic values in columns; **no repeating groups** hidden in one cell.

**Bad:** `order_tags = "gift,urgent,vip"` as a single string you must parse.  
**Better:** `order_tags` table `(order_id, tag)` or a proper array type with documented semantics.

**1NF + SQL:** Avoid “multi-valued columns” without structure unless your DB type and access patterns are explicit about it.

---

## 3.3 Second normal form (2NF)

**Relevant only with **composite** PK.** All **non-key** attributes must depend on the **full** composite key—not half of it.

**Classic example:** OrderLine `(order_id, product_id)` with `product_name` on the line.

- `{order_id, product_id}` is the key.  
- `product_name` depends on **`product_id` only** → **partial dependency** → not 2NF.  

**Fix:** Move `product_name` to **Product**; keep `product_id` on the line.

---

## 3.4 Third normal form (3NF)

**No transitive dependencies:** Non-key fields must depend **only** on the **whole PK**, not on **other** non-key fields.

**Bad:** On `orders`, storing `customer_email` when `customer_id` is already there—email changes with customer profile → risk of **stale copy** on `orders`.  
**Better:** Join to `customers` when needed; or **snapshot** email only if you need **historical** “what we knew at order time” (that is a **conscious denormalization** with meaning).

---

## 3.5 Anomalies in one picture

| Anomaly | What goes wrong |
|---------|------------------|
| **Insert** | Can’t insert a fact without inserting unrelated facts |
| **Update** | Must touch many rows to change one fact |
| **Delete** | Deleting something loses unrelated facts you meant to keep |

Normalization **reduces** these when redundancy is the cause.

---

## 3.6 When teams denormalize (on purpose)

- **Read-heavy** dashboards: **materialized** counts, **pre-joined** tables in a warehouse  
- **Low-latency** APIs: **cached** user fields on a document  
- **Event immutability:** **Denormalized** fields on an event for **replay** without joins  
- **Wide** analytics tables: **Star schema** is intentionally **denormalized** dimensions (file `06`)

**Staff framing:** “We normalize **OLTP** for integrity; we denormalize **OLAP** for scan performance—with **ETL** contracts so drift is visible.”

---

## 3.7 Normal forms beyond 3NF (names only)

**BCNF**, **4NF**, **5NF** remove subtler dependencies. Many OLTP systems **stop at 3NF**; know they exist for **interview depth**, not daily work.

---

## 3.8 What you should be able to say

- “**3NF** fights redundant facts; **star schemas** **denormalize** dimensions for analytics.”  
- “I’d **snapshot** prices on order lines **intentionally**—that’s not sloppy 3NF; it’s **temporal** correctness.”

**Next:** [Relationships in practice](./04-relationships-in-practice.md).

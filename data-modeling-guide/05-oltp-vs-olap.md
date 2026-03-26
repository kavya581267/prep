# 5 — OLTP vs OLAP: two kinds of “truth”

**Goal:** See **one business scenario** split into **operational** vs **analytical** schemas, and why **pipelines** connect them.

---

## 5.1 OLTP (online transaction processing)

**Workload:** Many **short** transactions: place order, update inventory, pause campaign. Users expect **milliseconds** to low hundreds of ms.

**Schema goals:**

- **Few or no** NULL forests in wide tables  
- **Foreign keys** and constraints for **correctness**  
- **Indexes** for “fetch order by id,” “list orders for customer last 30 days”

**Example — operational tables (simplified):**

```sql
-- Operational "source of truth" for current order state
CREATE TABLE orders (
  id              uuid PRIMARY KEY,
  customer_id     uuid NOT NULL REFERENCES customers(id),
  placed_at       timestamptz NOT NULL,
  status          text NOT NULL,  -- pending | paid | shipped | refunded
  currency        text NOT NULL
);

CREATE TABLE order_lines (
  id              uuid PRIMARY KEY,
  order_id        uuid NOT NULL REFERENCES orders(id),
  product_id      uuid NOT NULL REFERENCES products(id),
  qty             int NOT NULL CHECK (qty > 0),
  unit_price_usd  numeric(12,2) NOT NULL
);
```

**Typical OLTP query — “show me order o_5001 with line items”:**

```sql
SELECT o.id, o.status, ol.product_id, ol.qty, ol.unit_price_usd
FROM orders o
JOIN order_lines ol ON ol.order_id = o.id
WHERE o.id = 'o_5001';
```

Touches **few rows**; uses **PK/FK indexes**.

---

## 5.2 OLAP (online analytical processing)

**Workload:** **Aggregations** over **millions or billions** of rows: revenue by week by country, conversion by campaign, cohort retention.

**Schema goals:**

- **Fact** table at a clear **grain** (file `06`)  
- **Denormalized** dimensions for **fewer joins** during huge scans  
- Often **columnar** storage (BigQuery, Redshift, Snowflake, Parquet)

**Example — analytical fact (daily revenue by country):**

| order_date | country_code | orders_count | revenue_usd |
|------------|--------------|--------------|-------------|
| 2025-03-20 | US | 12_400 | 1_842_900.00 |
| 2025-03-20 | CA | 2_100 | 401_200.00 |

**Typical OLAP query:**

```sql
SELECT country_code, SUM(revenue_usd) AS rev
FROM fct_orders_daily
WHERE order_date BETWEEN '2025-03-01' AND '2025-03-31'
GROUP BY country_code;
```

**No join to `order_lines`** in the hot path—the ETL **already rolled up**.

---

## 5.3 Why one schema rarely does both well

| Concern | OLTP | OLAP |
|---------|------|------|
| **Indexes** | B-tree on keys | Column stats, zone maps, sort keys |
| **Writes** | Many small rows | Bulk **append** / partition replace |
| **Freshness** | **Now** | **Minutes** behind (near real-time) or **daily** |
| **Schema changes** | Weekly releases | Careful contracts for BI |

Running `SELECT sum(amount) FROM order_lines JOIN …` over **years** on the **primary** OLTP cluster can **hurt** checkout latency and **replication**—hence **isolation**.

---

## 5.4 How data flows (reference architecture)

```text
[OLTP Postgres] --CDC or hourly extract--> [Staging / lake]
      |                                        |
      v                                        v
 [App reads/writes]                    [dbt / Spark transforms]
                                               |
                                               v
                                    [Warehouse marts: fct_*, dim_*]
                                               |
                                               v
                                    [Looker / Mode / internal BI]
```

- **CDC (change data capture):** Debezium, Fivetran, native logical replication — **streams** row changes.  
- **Contracts:** Staging schemas versioned; **breaking jobs** alert owners.

---

## 5.5 Two “sources of truth” (without contradiction)

| Layer | Truth it owns |
|-------|----------------|
| **OLTP** | “Does order **o1** exist and is it **paid** right **now**?” |
| **Warehouse** | “What was **recognized revenue** for Q1 **per policy** X?” (depends on ETL + **definitions**) |

If ETL has a bug, **BI** is wrong while **OLTP** is right—hence **data quality** tests on marts (`revenue_daily` ≈ sum of OLTP for same scope).

---

## 5.6 CQRS — same business, two models

**Commands** update the **write model** (`orders`, `inventory`).  
**Queries** for “customer order history page” might read a **projection** table `order_summary_by_customer` rebuilt from events.

**Modeling impact:** You **accept duplication**. You need **replay** or **CDC** to rebuild projections after bugs.

---

## 5.7 Mini example — “one day in the life”

1. **10:42** Customer checks out → **OLTP** inserts `orders` + `order_lines` in one transaction.  
2. **10:43** CDC captures insert → **stream** → **warehouse staging**.  
3. **11:00** Batch job runs `MERGE` into `fct_order_lines` with `order_date`, `country_code`, `gross_usd`.  
4. **09:00 next day** Finance runs month-to-date report from **`fct_*`** only.

**Staff answer:** “We never let the **CFO query** hit **Postgres primaries**; we expose **curated marts** with **SLAs** and **tests**.”

---

## 5.8 Read replicas vs marts (subtle)

**Read replica** of OLTP: **same schema**, **async** replication lag (seconds), good for **internal** tools **if** queries are bounded.

**Mart:** **Different grain**, **denormalized**, **batch** or **stream** built; tuned for **BI**. Finance usually wants **marts**, not raw replica tables—definitions live in **dbt** / **semantic layer**.

**Danger:** Analysts join **10 tables** on replica → still hurts production **I/O** if replica co-located; use **warehouse** with **workload isolation**.

---

## 5.9 Idempotent MERGE into a mart (illustrative SQL)

**Problem:** CDC delivers **duplicates** or **replays**; fact loader must be **idempotent**.

```sql
MERGE INTO fct_orders_daily AS t
USING staging_orders_daily AS s
ON t.order_date = s.order_date AND t.country_code = s.country_code
WHEN MATCHED THEN UPDATE SET
  orders_count = s.orders_count,
  revenue_usd = s.revenue_usd,
  etl_batch_id = s.etl_batch_id
WHEN NOT MATCHED THEN INSERT (order_date, country_code, orders_count, revenue_usd, etl_batch_id)
VALUES (s.order_date, s.country_code, s.orders_count, s.revenue_usd, s.etl_batch_id);
```

(Syntax **varies** by engine—**BigQuery** `MERGE`, **Snowflake** `MERGE`, etc.)

**Key:** **Match keys** = **grain** of the fact.

---

## 5.10 Late-arriving and corrected facts

- **Late data:** Order placed **23:58** UTC shows next day in **Tokyo**—define **`order_date`** rule (**reporting timezone** vs **UTC**).  
- **Corrections:** Refund posted **after** close—finance uses **`transaction_dt`** vs **`recognized_dt`** or **adjustment** fact table.

Model **two timestamps** when **audit** and **recognition** diverge.

---

## 5.11 Lakehouse / medallion (names in one picture)

```text
Bronze  — raw JSON/Avro as landed (immutable)
Silver  — cleaned, typed, deduped (still fairly wide)
Gold    — business marts (fct/dim contracts)
```

**Modeling impact:** **Bronze** tolerates **schema drift**; **Gold** enforces **grain** and **keys**.

---

## 5.12 What you should be able to say

- “**OLTP** = row-level correctness + constraints; **OLAP** = scan-heavy aggregates at defined **grain**.”  
- “I connect them with **CDC/ELT** and treat **marts** as a **product** with tests.”  
- “**Replicas** aren’t **marts**; I **MERGE** facts on **grain keys** and document **timezone** and **late arrival**.”

**Next:** [Dimensional modeling](./06-dimensional-modeling-star-schema.md).

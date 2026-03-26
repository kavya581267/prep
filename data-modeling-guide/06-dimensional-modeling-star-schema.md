# 6 — Dimensional modeling & star schema

**Goal:** Build a **star schema** with an explicit **grain**, realistic **dimension** rows, and **fact** rows you can **aggregate** without double-counting.

---

## 6.1 Grain (define it first)

The **grain** completes: “**One row in this fact table = one ___**. ”

**Examples:**

- One **impression** (event-level; huge)  
- One **order line** (fine-grained commerce)  
- One row per **calendar day × campaign × country** (rolled-up for executive dashboards)

**Rule:** Mixing grains (some rows per impression, some per day) in the **same** fact without flags is a **reporting bug** waiting to happen.

For this file we use **daily roll-up**:

> **Grain:** `fct_campaign_impressions_daily` = one row per **date** × **campaign** × **country**.

---

## 6.2 Dimension tables (with sample data)

### `dim_date`

| date_key | calendar_date | day_of_week | is_weekend | fiscal_period |
|----------|---------------|-------------|--------------|----------------|
| 20250320 | 2025-03-20 | Thu | 0 | 2025-Q1 |
| 20250321 | 2025-03-21 | Fri | 0 | 2025-Q1 |

In warehouses people often use **`YYYYMMDD`** integer or the **date** type as key.

### `dim_campaign` (Type 1 for simplicity — no history columns here)

| campaign_key | campaign_id_natural | advertiser_name | campaign_name | channel |
|----------------|---------------------|-----------------|-----------------|---------|
| 101 | CMP-9001 | Acme | Spring Promo | CTV |
| 102 | CMP-9002 | Acme | Retargeting | Display |

`surrogate` `campaign_key` is what facts store—stable if natural `campaign_id` is reused in tests.

### `dim_geo`

| geo_key | country_code | region_name |
|---------|----------------|-------------|
| 501 | US | West |
| 502 | US | Northeast |
| 503 | CA | Ontario |

---

## 6.3 Fact table (measures + foreign keys to dimensions)

**`fct_campaign_impressions_daily`**

| date_key | campaign_key | geo_key | impressions | clicks | cost_usd | revenue_usd |
|----------|--------------|---------|-------------|--------|-----------|---------------|
| 20250320 | 101 | 501 | 120000 | 900 | 1800.00 | 2100.00 |
| 20250320 | 101 | 502 | 85000 | 620 | 1240.00 | 1450.00 |
| 20250320 | 102 | 501 | 40000 | 300 | 600.00 | 720.00 |

**Measures** `impressions`, `clicks`, `cost_usd`, `revenue_usd` are **additive** — you **SUM** them across rows of **same grain**.

**Non-additive** examples (don’t sum blindly): distinct users (use **approx distinct** or **special grain**), ratios like CTR (compute `SUM(clicks)/SUM(impressions)`).

---

## 6.4 Star shape (how joins work)

```text
                    dim_date
                        |
                        | date_key
                        v
dim_campaign -----> fct_campaign_impressions_daily <----- dim_geo
   campaign_key                                      geo_key
```

**Query — revenue by country for March 2025:**

```sql
SELECT g.country_code, SUM(f.revenue_usd) AS revenue
FROM fct_campaign_impressions_daily f
JOIN dim_date d ON d.date_key = f.date_key
JOIN dim_geo g ON g.geo_key = f.geo_key
WHERE d.calendar_date >= '2025-03-01'
  AND d.calendar_date <  '2025-04-01'
GROUP BY g.country_code;
```

---

## 6.5 Snowflake (normalized dimensions) — quick contrast

Instead of wide `dim_geo` with `region_name` and `subregion_name`, you **split**:

- `dim_region (region_key, region_name)`  
- `dim_country (country_key, country_code, region_key)`  

**Fact** still points to **`country_key`**. More **normalized**; **more joins** for analysts. Some teams **snowflake** only huge shared hierarchies.

---

## 6.6 Degenerate dimension

If you need **order_id** on a fact but there’s no stable `dim_order` with attributes you slice by, put **`order_id`** **on the fact** as a **degenerate** dimension—no separate table.

**Example fact row:** `…, order_id = 'ORD-2025-0842', …`

Useful for **drill-through** to operational systems.

---

## 6.7 Role-playing dimension

**Ship date** and **arrival date** both reference **`dim_date`**. In SQL you **join `dim_date` twice** with aliases:

```sql
FROM fct_shipments f
JOIN dim_date d_ship ON d_ship.date_key = f.ship_date_key
JOIN dim_date d_arrive ON d_arrive.date_key = f.arrival_date_key
```

Model **one** `dim_date`; **semantic layer** names the roles.

---

## 6.8 Bridge between this and OLTP

Operational DB has **events** or **lines**; ETL **aggregates** into `fct_*`:

```text
impression_events (millions per day)
    --> aggregate by day, campaign_id, country
    --> load into fct_campaign_impressions_daily
```

If someone asks “why numbers don’t match,” you compare **grain** and **filters** (bots removed? timezone?).

---

## 6.9 Junk / mystery dimensions (flags basket)

Many **low-cardinality** booleans (`is_weekend`, `is_holiday`, `is_fraud_reviewed`) clutter the fact. Alternative: pack them into **`dim_misc_flags`**.

| flag_key | is_weekend | is_public_holiday |
|----------|------------|-------------------|
| 1 | 0 | 0 |
| 2 | 1 | 0 |
| 3 | 1 | 1 |

**Tradeoff:** Fewer columns on fact; **obscure** flags table—**document** in semantic model.

---

## 6.10 Semi-additive facts (account balance)

**Grain:** one row per **account** × **day**; measure **`closing_balance`**.

**Wrong:** `SUM(closing_balance)` across days **double-counts** the same money.

**Right:** **`SUM`** across **additive** dims (accounts?) only **after** taking **last day of period** or **avg** by policy—often `closing_balance` is **snapshot** at month-end per account.

---

## 6.11 Bridge table — many-to-many dimensions

**Fact:** impression; **creative** belongs to **many “concepts”** (car, summer, promo); analysts want “any creative tagged **car**.”

**Bridge:** `bridge_creative_concept (creative_key, concept_key, weight)` — fact stays at **impression × creative**; concepts attach via bridge with **allocation** or **double-count** warnings in the **semantic layer**.

**Interview depth:** **Impact reports** and **allocated** metrics need **explicit** rules (“divide evenly,” “use impression-level tag”).

---

## 6.12 Conformed dimensions (drill-across)

**Same** `dim_date` and **`dim_geo`** used in **`fct_sales`** and **`fct_impressions`** → you can **divide** revenue by impressions **by week × country** in one query **if** definitions align.

**Staff:** Without **conformed** keys, executives compare **apples** to **oranges**.

---

## 6.13 Partitioning facts (operations)

Daily grain facts → **`PARTITION BY RANGE (date_key)`** (monthly files in lake). **Pruning** pushes filters to storage.

**Dimension tables** usually **small**—**replicated** / **broadcast**; **not** partitioned like facts.

---

## 6.14 What you should be able to say

- “I’d write the **grain** on the fact table doc: one row = **day × campaign × country**.”  
- “I only **SUM** additive measures; **CTR** = ratio of **sums**, not sum of ratios.”  
- “**Semi-additive** balances need last-day-of-period rules; **bridge** tables encode **M:N** dims with **double-count** policy.”

**Next:** [Events, time, SCD](./07-events-time-history-scd.md).

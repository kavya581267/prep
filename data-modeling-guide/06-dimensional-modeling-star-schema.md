# 6 — Dimensional modeling & star schema

**Goal:** Design **facts** (measurements) and **dimensions** (context) so analytics queries stay **fast** and **understandable**.

---

## 6.1 Grain (the most important sentence)

The **grain** of a fact table is **one row = one ___**.

**Examples:**

- One **impression** event  
- One **order line** per day per SKU (aggregated grain—valid if documented)  
- One **daily** rollup per campaign per geo  

**Rule:** Never mix grains in one fact table without clear **derived** semantics.

---

## 6.2 Fact table

Holds **numeric measures** you **add** (mostly): `revenue`, `impression_count`, `click_count`, `cost_usd`.

Holds **foreign keys** to dimensions: `date_key`, `campaign_key`, `geo_key`, …

Often **skinny on text**—descriptive text lives in **dimensions**.

---

## 6.3 Dimension table

Describes **who/what/where/when**: **customer**, **product**, **campaign**, **date**, **inventory type**.

**Surrogate keys** in dimensions (`campaign_key`) let you track **history** (SCD—file `07`) without rewriting fact rows.

---

## 6.4 Star vs snowflake

| Shape | Structure | Pros / cons |
|-------|-----------|-------------|
| **Star** | Dimensions **denormalized** (wide): `dim_geo` has country + region columns | Simpler joins; wider tables |
| **Snowflake** | Dimensions **normalized** (country table, region table) | Saves space; more joins |

**Reality:** Warehouses often **star** + **nested** types for semi-structured bits.

---

## 6.5 Mini star example — ad reporting

**Fact:** `fct_impressions_daily` — **grain:** one row per **day** × **campaign** × **placement_type** × **country**

**Measures:** `impressions`, `clicks`, `revenue_usd` (allocated)

**Dimensions:**

- `dim_date` (calendar, fiscal week)  
- `dim_campaign` (advertiser, line item name, channel)  
- `dim_placement` (app vs web, size)  
- `dim_geo` (country, region)  

**Drill path:** Start coarse (country × week); drill to campaign when questions need it.

---

## 6.6 Degenerate dimensions

Sometimes there is no good dimension row—store on the fact: `order_id`, `transaction_id`. **Degenerate dimension** = readable key **in** the fact, no separate table.

---

## 6.7 Role-playing dimensions

Same **dim_date** joined **twice** as `ship_date` and `arrival_date` (aliases in SQL). Model once; **document** roles in semantic layer.

---

## 6.8 What you should be able to say

- “I’d state **grain** first; everything else flows from that.”  
- “**Dimensions** answer **slice and dice**; **facts** answer **how much**.”

**Next:** [Events, time, SCD](./07-events-time-history-scd.md).

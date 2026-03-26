# 1 — Entities, attributes, relationships (beginner)

**Goal:** Think in **things** (entities), **properties** (attributes), and **connections** (relationships) before worrying about SQL or NoSQL.

---

## 1.1 Entity

An **entity** is a **category of thing** your system cares about that you can **name** and **tell apart** from others (usually with an **identifier**).

**Examples:**

- **Customer** — a person or org you sell to  
- **Order** — one checkout transaction  
- **Campaign** — an advertiser’s delivery unit with budget and dates  
- **Impression event** — one logged “ad was shown” occurrence  

**Not everything is an entity.** A calculated “total revenue this month” is often a **metric**, not a stored entity—unless you **materialize** it for performance.

---

## 1.2 Attribute

An **attribute** is **data describing** an entity at a point in time or for its lifetime.

**Examples:**

| Entity | Example attributes |
|--------|---------------------|
| Customer | email, country, created_at |
| Order | placed_at, currency, status |
| Campaign | budget_usd, start_date, advertiser_id |

**Simple vs composite:** `address_line1` is simple; a full **postal address** might be modeled as several columns or a structured type depending on your DB.

---

## 1.3 Relationship

A **relationship** connects entities with a **rule**:

- “Each **Order** belongs to **one Customer**.”  
- “Each **Campaign** has many **Creatives**.”  

Relationships have **cardinality** (next section) and often become **foreign keys** or **embedded documents** when you implement.

---

## 1.4 Cardinality (say it clearly)

| Pattern | Meaning | Example |
|---------|---------|---------|
| **One-to-one (1:1)** | Each A links to at most one B | User ↔ user_preferences (split tables for size or security) |
| **One-to-many (1:N)** | One A, many B’s | Customer → many Orders |
| **Many-to-many (M:N)** | Many A’s ↔ many B’s | Orders ↔ Products (via **line items**) |

**Interview tip:** If you say “many-to-many,” be ready to mention a **junction** (associative) table or equivalent.

---

## 1.5 Mini example — e-commerce (conceptual)

**Entities:** Customer, Order, Product, OrderLine  

**Relationships:**

- Customer **1:N** Order  
- Order **1:N** OrderLine  
- Product **1:N** OrderLine (each line is one product on one order)  

**Attributes (sample):**

- Order: `id`, `customer_id`, `placed_at`, `status`  
- OrderLine: `order_id`, `product_id`, `qty`, `unit_price_snapshot`  

Notice **`unit_price_snapshot`**: price **at order time** may differ from today’s catalog price—modeling **time** matters (see file `07`).

---

## 1.6 Mini example — ad tech (conceptual)

**Entities:** Advertiser, Campaign, Creative, Placement  

- Advertiser **1:N** Campaign  
- Campaign **1:N** Creative  
- **Serving** might link Campaign to Placement via **targeting rules**—often **M:N** (“this campaign can run on these placement types”) with a junction row of constraints.

---

## 1.7 What you should be able to say

- “I’d name **entities** from **nouns** in requirements, then draw **relationships** and **cardinality** before choosing tables.”  
- “I’d separate **who** (customer), **what happened** (order), and **line detail** (order line) for clarity and reporting.”

**Next:** [Keys & identifiers](./02-keys-identifiers-uniqueness.md).

# Data Modeling — Start Here

This folder is a **structured path** from **beginner** (entities, keys, normalization) through **intermediate** (dimensional models, events, NoSQL shapes) to **advanced / Staff** (evolution, multi-tenancy, correctness, tradeoffs).

Each numbered file includes **worked examples** and **deeper sections**: sample rows, **SQL/DDL**, mermaid ER where useful, normalization beyond 3NF (BCNF/4NF sketches), star-schema edge cases (semi-additive, bridges), **outbox/CDC**, NoSQL single-table design, migration operations, concurrency (inventory), data-quality gates, and Staff-style ownership—not just bullet pointers.

**Read in numbered order** the first time through. After that, use any file as a **reference**.

## Who this is for

- Engineers who want **shared vocabulary** for “how we structure data.”  
- Interview prep for roles that touch **OLTP services**, **analytics**, or **data platforms**.  
- Anyone bridging **application schemas** and **warehouse** design.

## Reading order

| # | File | Level | What you’ll learn |
|---|------|-------|-------------------|
| 1 | [Entities, attributes, relationships](./01-entities-attributes-relationships.md) | Beginner | Bookstore + ad-tech ER; sample tables; process to sketch |
| 2 | [Keys, identifiers, uniqueness](./02-keys-identifiers-uniqueness.md) | Beginner+ | DDL, natural vs surrogate, UUIDs, composite & partial UNIQUE |
| 3 | [Normalization & denormalization](./03-normalization-and-denormalization.md) | Beginner→Int. | 1NF–3NF before/after; anomaly stories; exercise |
| 4 | [Relationships in practice](./04-relationships-in-practice.md) | Intermediate | Min tables; join examples for 1:N, M:N, 1:1 |
| 5 | [OLTP vs OLAP — two kinds of “truth”](./05-oltp-vs-olap.md) | Intermediate | Side-by-side schemas; CDC path; “one day” narrative |
| 6 | [Dimensional modeling & star schema](./06-dimensional-modeling-star-schema.md) | Intermediate | `dim_*` / `fct_*` sample data + star query |
| 7 | [Events, time, and history (SCD)](./07-events-time-history-scd.md) | Intermediate+ | Event rows, Type 2 rows, as-of SQL, idempotent JSON |
| 8 | [NoSQL & alternative shapes](./08-nosql-and-alternative-shapes.md) | Intermediate+ | Mongo-style doc, Cassandra-style PK, KV, search pattern |
| 9 | [Evolution, versioning, multi-tenancy](./09-evolution-versioning-multitenancy.md) | Advanced | Expand–contract week story; RLS sketch; tenant rows |
| 10 | [Patterns, anti-patterns, Staff tradeoffs](./10-patterns-anti-patterns-staff-tradeoffs.md) | Advanced | Catalog + full design-review narrative |

## How to use with the rest of `prep`

- **Streaming / lake:** [`../staff-deep-dive-guide/04-data-streaming-kafka-spark-lakehouse.md`](../staff-deep-dive-guide/04-data-streaming-kafka-spark-lakehouse.md)  
- **Ad tech domain examples:** [`../adtech-beginner-guide/00-START-HERE.md`](../adtech-beginner-guide/00-START-HERE.md)

## One-sentence picture

**Data modeling is choosing structures so the right questions stay fast, safe, and honest as data grows and rules evolve.**

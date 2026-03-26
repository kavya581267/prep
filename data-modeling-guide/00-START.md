# Data Modeling — Start Here
This folder is a **structured path** from **beginner** (entities, keys, normalization) through **intermediate** (dimensional models, events, NoSQL shapes) to **advanced / Staff** (evolution, multi-tenancy, correctness, tradeoffs).
**Read in numbered order** the first time through. After that, use any file as a **reference**.
## Who this is for
- Engineers who want **shared vocabulary** for “how we structure data.”  
- Interview prep for roles that touch **OLTP services**, **analytics**, or **data platforms**.  
- Anyone bridging **application schemas** and **warehouse** design.
## Reading order
| # | File | Level | What you’ll learn |
|---|------|-------|-------------------|
| 1 | [Entities, attributes, relationships](./01-entities-attributes-relationships.md) | Beginner | ER mindset; cardinality in words; example domains |
| 2 | [Keys, identifiers, uniqueness](./02-keys-identifiers-uniqueness.md) | Beginner+ | PK/FK; natural vs surrogate; UUIDs and time-sortable ids |
| 3 | [Normalization & denormalization](./03-normalization-and-denormalization.md) | Beginner→Int. | 1NF–3NF; anomalies; when to break rules |
| 4 | [Relationships in practice](./04-relationships-in-practice.md) | Intermediate | Min tables per cardinality; how to define PK/FK/UNIQUE; junctions |
| 5 | [OLTP vs OLAP — two kinds of “truth”](./05-oltp-vs-olap.md) | Intermediate | Workloads; why one schema rarely serves both |
| 6 | [Dimensional modeling & star schema](./06-dimensional-modeling-star-schema.md) | Intermediate | Facts, dimensions, grains; star vs snowflake |
| 7 | [Events, time, and history (SCD)](./07-events-time-history-scd.md) | Intermediate+ | Event logs vs snapshots; SCD types; bi-temporal ideas |
| 8 | [NoSQL & alternative shapes](./08-nosql-and-alternative-shapes.md) | Intermediate+ | Document, wide-column, graph; aggregate boundaries |
| 9 | [Evolution, versioning, multi-tenancy](./09-evolution-versioning-multitenancy.md) | Advanced | Migrations; schema registries; tenant strategies |
| 10 | [Patterns, anti-patterns, Staff tradeoffs](./10-patterns-anti-patterns-staff-tradeoffs.md) | Advanced | Idempotency, hot keys, privacy boundaries, interview cues |
## How to use with the rest of `prep`
- **Streaming / lake:** [`../staff-deep-dive-guide/04-data-streaming-kafka-spark-lakehouse.md`](../staff-deep-dive-guide/04-data-streaming-kafka-spark-lakehouse.md)  
- **Ad tech domain examples:** [`../adtech-beginner-guide/00-START-HERE.md`](../adtech-beginner-guide/00-START-HERE.md)
## One-sentence picture
**Data modeling is choosing structures so the right questions stay fast, safe, and honest as data grows and rules evolve.**

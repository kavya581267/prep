# 10 — Patterns, anti-patterns, Staff tradeoffs

**Goal:** Short catalog for **design reviews** and **interviews**—what good looks like, what fails at scale, what to **say out loud**.

---

## 10.1 High-value patterns

| Pattern | What it solves |
|---------|----------------|
| **Surrogate PK + business unique key** | Stable joins + integration keys |
| **Junction table with edge attributes** | M:N with **context** on the relationship |
| **Explicit grain in fact tables** | Prevents **double-count** in BI |
| **Type 2 dimensions** | **As-of** reporting without lying |
| **Event + projection** | Fast reads **without** losing **history** |
| **Outbox / transactional messaging** | Consistent **DB write** + **event** |
| **Idempotency keys** | Safe **retries** and **pipelines** |

---

## 10.2 Anti-patterns (red flags)

| Smell | Why it hurts |
|-------|----------------|
| **God table** (80 nullable columns) | No ownership, **sparse** data, scary migrations |
| **Unbounded JSON blob** as “schema” | **Query** pain; **validation** only in app |
| **Polymorphic FK everywhere** | **Integrity** and **join** hell |
| **Dual sources of truth** for same field | **Drift** between services or cache/DB |
| **Derived metrics in OLTP** without job story | **Stale** numbers under load |
| **Hot row / hot partition** (e.g. global counter) | **Write** throughput ceiling |

---

## 10.3 Staff interview prompts — crisp answers

**“How do you model X?”**  
Start with **access patterns** and **consistency** needs, **not** tables. Then entities, keys, **normalization vs read model**.

**“OLTP vs warehouse?”**  
**CDC** or **events** out; **contracts**; **don’t** scan primary for exec dashboards.

**“Multi-region?”**  
**ID** generation, **FK** across regions (avoid if possible), **conflict** strategy for writes, **RPO/RTO** for data.

**“GDPR / delete?”**  
**PII map**, **retention**, **erase** vs **anonymize**, **cascade** to warehouse and **logs** (often **hardest**).

---

## 10.4 Tradeoff matrix (soundbyte)

| If you optimize for… | You often accept… |
|----------------------|-------------------|
| **Join flexibility** | Slower **wide** scans unless **warehouse** |
| **Write throughput** | **Denormalized** reads or **async projection** |
| **Schema agility** | **Versioned** events; **tolerance** for **nullable** phases |
| **Tenant isolation** | **Higher** infra cost |
| **Analytical ergonomics** | **Duplicated** data and **ETL** complexity |

---

## 10.5 Checklist before signing off on a model

1. **Who queries it** (service, BI, ML)? With what **latency**?  
2. **Grain** of facts / events documented?  
3. **Keys** and **uniqueness** rules explicit?  
4. **Migration path** for next quarter’s field?  
5. **Deletion / retention** story for **PII**?  
6. **Hot spots** identified (tenant, shard key, counter)?  
7. **Failure** of async projection—**replay** possible?

---

## 10.6 Where to go deeper

- **Kimball** dimensional modeling (classic texts)  
- **Martin Kleppmann** — stream processing, logs, consistency  
- Your **`data-modeling-guide`** files **01–09** for the progressive base

---

You now have a **linear path** from **ER thinking** to **Staff-level** data architecture vocabulary—reuse the **checklist** on real designs and in mock interviews.

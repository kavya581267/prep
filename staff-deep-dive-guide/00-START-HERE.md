# Staff Engineer Prep — Deep Dives (Beginner + Staff Depth)

This folder mirrors **every major topic** in [`WBD-Staff-Engineer-2-Day-Prep-Plan.md`](../WBD-Staff-Engineer-2-Day-Prep-Plan.md) with **substantive depth**, not link lists:

1. **Beginner-friendly** grounding (what the thing is, why it exists).  
2. **How it works** (mechanics: Kafka record lifecycle, outbox steps, triple-reverse rotation, etc.).  
3. **Staff-level** tradeoffs with **reasoning** you can say aloud in interviews.  
4. **Question banks** where useful.

Files **03–13** each include **worked examples** (numeric back-of-envelope, JSON/HTTP shapes, timelines, dialogues, step-by-step algorithms) after the conceptual sections.

If a section still feels thin for you, say which **file number** to grow next.

**Ad Tech vocabulary** (impressions, SSP/DSP, linear TV terms, …) lives in [`../adtech-beginner-guide/00-START-HERE.md`](../adtech-beginner-guide/00-START-HERE.md). Read that **before or alongside** the system design file below.

---

## Reading order (recommended)

| Step | Topic | File |
|------|--------|------|
| A | Loop shape & strategy | [01-interview-loop-and-calibration.md](./01-interview-loop-and-calibration.md) |
| B | Two-day schedule | [02-two-day-execution-plan.md](./02-two-day-execution-plan.md) |
| C | Ad serving + A/B **system design** | [03-system-design-ad-serving-experiments.md](./03-system-design-ad-serving-experiments.md) |
| D | Kafka, streaming, Spark, lakehouse | [04-data-streaming-kafka-spark-lakehouse.md](./04-data-streaming-kafka-spark-lakehouse.md) |
| E | API-first & contracts | [05-api-first-platforms-contracts.md](./05-api-first-platforms-contracts.md) |
| F | Modernization & legacy | [06-platform-modernization-legacy-linear.md](./06-platform-modernization-legacy-linear.md) |
| G | HA, reliability, observability, cost | [07-ha-reliability-observability-cost.md](./07-ha-reliability-observability-cost.md) |
| H | Operational & Engineering Excellence | [08-operational-engineering-excellence.md](./08-operational-engineering-excellence.md) |
| I | Leadership, influence, STAR | [09-staff-leadership-influence.md](./09-staff-leadership-influence.md) |
| J | Coding + complexity talk | [10-coding-and-complexity.md](./10-coding-and-complexity.md) |
| K | Five practice system designs | [11-alternate-system-design-prompts.md](./11-alternate-system-design-prompts.md) |
| L | Questions **you** ask | [12-reverse-interview-questions.md](./12-reverse-interview-questions.md) |
| M | Resilience & cost catalog | [13-resilience-patterns-catalog.md](./13-resilience-patterns-catalog.md) |

**Cross-reference:** Main plan **Part 10** → [`adtech-beginner-guide`](../adtech-beginner-guide/00-START-HERE.md). **Parts 3–9, 11–14** map to **C–M** above (same content as the big single file, expanded here for learning).

**Data modeling (beginner → advanced):** [`../data-modeling-guide/00-START.md`](../data-modeling-guide/00-START.md)  
**Spark & batch ETL deep dives:** [`../spark-guide/00-START-HERE.md`](../spark-guide/00-START-HERE.md) · [`../batch-etl-guide/00-START-HERE.md`](../batch-etl-guide/00-START-HERE.md)

---

## How each doc is structured

- **In one sentence** — mental model.  
- **Beginner lens** — plain definitions.  
- **Staff lens** — tradeoffs, ownership, org dynamics, SLOs, cost.  
- **Interview** — clarifying questions, red flags, strong answer patterns.

No substitute for **mocking out loud**, but this is the **reference layer** behind your mocks.

# WBD Staff Engineer — Prep Plan & Full-Depth Interview Reference

**Role focus:** Staff-level technical leadership on large-scale **AdTech** + **cloud-native** platforms, **legacy broadcast/scheduling modernization**, **API-first** design, **Kafka/Spark/data** at scale, and **influence without authority**.

**Beginner + Staff learning track (recommended):** topic-by-topic guides live in [`staff-deep-dive-guide/00-START-HERE.md`](./staff-deep-dive-guide/00-START-HERE.md) (system design, data, APIs, modernization, reliability, OE, leadership, coding, practice designs, reverse questions, resilience catalog). **Ad Tech 101** vocabulary: [`adtech-beginner-guide/00-START-HERE.md`](./adtech-beginner-guide/00-START-HERE.md). **Spark (beginner → advanced):** [`spark-guide/00-START-HERE.md`](./spark-guide/00-START-HERE.md). **Batch ETL / pipelines:** [`batch-etl-guide/00-START-HERE.md`](./batch-etl-guide/00-START-HERE.md).

**How to use this file:** Treat it as a **single-file** question bank and tradeoff catalog; use the folders above for **narrative depth** and clearer pacing.

**Note on “all possible questions”:** No list is exhaustive. This document maps **families** of prompts and **dimensions of tradeoffs** so you can compose answers to variants you have not seen.

---

## Part 1 — Reported loop calibration (summary)

- **Staff-style loop (anecdotal):** ~4 rounds; **Principal** session labeled **Operational & Engineering Excellence**; **two system-design** sessions (sometimes **CoderPad**, often **HLD + touch of LLD**). Source: [Blind — Staff SDE loop at WBD](https://www.teamblind.com/post/staff-sde-interview-loop-at-warner-bros-discovery-fruanwzs).
- **Reported practice questions (PracHub):** [Ad serving + A/B testing](https://prachub.com/interview-questions/design-an-ad-content-delivery-system-with-a-b-testing), [Array rotation + CSV keyword search](https://prachub.com/interview-questions/implement-array-rotations-and-csv-keyword-search). Index: [WBD on PracHub](https://prachub.com/companies/warner-bros-discovery).

---

## Part 2 — Two-day schedule (execution checklist)

### Day 1

| Block | Time | Focus |
|-------|------|--------|
| Morning | 3–4h | Architecture/APIs/microservices (Part 3.4–3.6); one system story with diagram |
| Midday | 1–2h | Kafka/Spark/data (Part 4) |
| Afternoon | 2–3h | AdTech + linear modernization (Part 5); domain language |
| Evening | 90–120m | OE/Principal stories (Part 6) + 6× STAR (Part 7) |

### Day 2

| Block | Time | Focus |
|-------|------|--------|
| Morning | 2–2.5h | Mock: Part 3.1 end-to-end; CoderPad API/schema typing; coding timer (Part 8) |
| Midday | 1h | Why WBD / why role (60s pitch) |
| Afternoon | 90m | Your questions for them (Part 9) |
| Final | 30m | One-line positioning, 3 strengths, 1 growth area |

### If you only have three priorities

1. **Part 3.1** (ad serving + experiments) with tradeoff tables memorized at headline level.  
2. **Part 6** — one strong incident/SLO story with numbers.  
3. **Part 8** — medium coding under time; skim Part 4 if data round is likely.

---

# Part 2 (deep) — System design: Ad serving + A/B testing

This aligns with the publicly circulated WBD-style prompt: low-latency **select + serve**, **targeting**, **delivery controls**, **impression/click logging**, and **experimentation**.

## 3.1 Clarifying questions you should ask the interviewer

**Scale & SLOs**

- Peak and average **QPS** on the ad-decision path? Read vs. write ratio?
- **Latency SLO** (p50/p95/p99)? Hard upper bound (e.g., must stay under 50ms at p99 excluding network)?
- Geographic distribution: single region vs. multi-region; **data residency** constraints?

**Functional**

- Placements: **web, mobile app, CTV, linear** simulcast, or generic “slot ID”?
- **Creative types**: static image, video VAST/VMAP, interactive?
- Targeting dimensions: **geo, device, OS, app/channel, time-of-day, logged-in segment, contextual** (content genre)?
- **Frequency cap**: per user, per device, per household, per campaign? Window (per day / rolling 7d / campaign lifetime)?
- **Pacing**: even delivery vs. ASAP until budget; **second-price** vs. **internal** auction or **waterfall**?
- **Attribution**: clicks only vs. view-through; **click redirect** through your domain?

**Experiments**

- Unit of assignment: **user ID**, **device ID**, **session**, **request-level** (not recommended for causal metrics)?
- Need **mutual exclusion** between overlapping experiments? **Layers / domains**?
- Guardrails: **revenue**, **latency**, **error rate**, **inventory exhaustion**?

**Reporting**

- What must be **real-time** (ops dashboard) vs. **T+1 / hourly** (finance, billing reconciliation)?
- **Identity** for cross-device deduplication?

Asking these signals Staff-level **problem framing** before drawing boxes.

## 3.2 High-level architecture (typical services)

| Component | Responsibility |
|-----------|----------------|
| **Edge / API gateway** | Auth, rate limits, request validation, WAF, routing |
| **Ad decision service** | Orchestrates: targeting → candidate retrieval → ranking → policy → response assembly |
| **Targeting / audience** | Resolves segment membership (online features, may call feature store) |
| **Campaign & creative store** | Source of truth for active campaigns, creatives, rules; cached for read-heavy path |
| **Ranking / selection** | Business logic: auction substitute, weighted random, epsilon-greedy for exploration |
| **Pacing / budget / freq** | Often **centralized counters** (Redis-style) or **approximate** with bounded staleness |
| **Experimentation service** | Stable **bucketing** hash; **exposure** events; ties into same decision call |
| **Event ingress** | High-throughput **impression/click/exposure** intake; buffer + async |
| **Stream / batch analytics** | Aggregation for reporting, experiment analysis, fraud checks |
| **Config / feature flags** | Kill switches, dynamic weights |

You may collapse services in a 45-minute round; Staff judgment is **when to merge vs. split** for team boundaries and blast radius.

## 3.3 Core data models (sketch)

**Campaign**

- `campaign_id`, advertiser, budget, currency, start/end, **pacing_mode**, priority, **frequency_rules[]**, geo_allowlist/blocklist, placement_allowlist, creative_ids, **status** (draft/active/paused).

**Creative**

- `creative_id`, type, asset URLs, click URL, duration (video), **policy** tags (safety, brand).

**Line item / ad group** (if needed)

- Binds targeting + creatives + bid or **internal weight**.

**Events (append-only)**

- `request_id` (immutable), `timestamp`, `placement_id`, `resolved_user_or_device_id`, `campaign_id`, `creative_id`, `experiment_key`, `variant`, `latency_ms`, `outcome` (fill/no-fill), `price_micros` (if applicable).

**Experiment definitions**

- `experiment_id`, **salt**, **audience filter**, **variants** and **allocation %**, start/end, **primary metrics**, **guardrails**, owner.

Interviewers often probe **idempotency**: same `request_id` retried must not **double-count** impressions or budget.

## 3.4 Serving path: request → decision → response

1. Resolve **identity** (or anonymous stable ID).  
2. Load **experiment assignments** (from cache/compute via hash).  
3. Retrieve **candidate ads** (indexed by placement + coarse targeting).  
4. Apply **hard filters** (brand safety, frequency cap read, geo).  
5. **Rank / select** (possibly variant-specific logic for A/B).  
6. **Record exposure** (async path — see tradeoffs).  
7. Return **creative metadata** + **tracking URLs** / **beacons**.

## 3.5 A/B testing: assignment, logging, metrics

**Assignment**

- Deterministic: `variant = hash(salt + experiment_id + stable_unit) % 100` against cumulative buckets.  
- Stable unit avoids **user hopping** variants between requests.

**Exposure logging**

- Log when the user **could have seen** the treatment (decision made), not only when asset downloads — align with your product definition; be consistent.

**Analysis**

- **Offline** batch (Hive/Spark/BQ) for **power** and confidence; near-real-time for **guardrails** (revenue drop, error spike).  
- Watch **Simpson’s paradox** (day-of-week skew), **novelty effects**, **interference** between experiments.

**Tradeoffs**

| Approach | Pros | Cons |
|----------|------|------|
| Request-level randomization | Simple | Breaks user-level causality; messy metrics |
| Stable user bucketing | Cleaner metrics | Requires stable ID; cold-start users |
| Server-side only assignment | Control | Misses client-only outcomes unless instrumented |
| Client-side bucketing | Lower server load | Tampering / inconsistency risk |

## 3.6 Caching strategy — question bank & tradeoffs

**Possible follow-ups**

- What do you cache: **campaign metadata**, **segment membership**, **pacing headroom**, **experiment assignment**?
- **TTL** vs **event-driven invalidation**?
- **Stampede** prevention: probabilistic early expiry, single-flight, lock per key?

| Strategy | Pros | Cons |
|----------|------|------|
| Aggressive edge cache | Min latency | Staleness on pause/cancel; need **short TTL** or **active purge** |
| No cache on money path | Strong consistency | Latency and DB load |
| **CDN** for creatives only | Offloads bytes | Decision still on origin |
| Application cache (local) | Fast | **Thundering herd** per instance |
| Shared cache (Redis cluster) | Consistency better than local | Network hop; **hot keys** for mega-campaigns |

Staff answer: tie cache design to **SLO** and **acceptable staleness** (e.g., 1–2s for creative swap vs. immediate for legal takedown — **feature flag kill** overrides cache).

## 3.7 Pacing, budget, frequency caps — question bank & tradeoffs

**Possible follow-ups**

- **Distributed counters**: How to avoid race conditions? (Lua scripts, CRDT-approx, sharded counters.)  
- **Under-delivery** vs **overspend** — which is worse for the business?  
- **Time zones** for day-parting?

| Approach | Pros | Cons |
|----------|------|------|
| Central Redis-style counters | Fast increments | Hot keys; failure domain |
| DB with row locks | Correct | Too slow for peak path |
| **Probabilistic** pacing | Smooth | Approximate; complex tuning |
| Async reconciliation | Offloads path | Temporary overspend risk — need caps |

## 3.8 Event pipeline & analytics

**Possible follow-ups**

- **At-least-once** delivery: dedupe by `request_id` at sink.  
- **Late events**: watermark policy in streaming aggregates.  
- **PII** in logs: hash/tokenize, retention policy.  
- **Data skew**: hot campaigns dominating partitions — **salting** or **two-phase** aggregate.

| Real-time | Batch |
|-----------|-------|
| Ops dashboards, guardrails | Finance, MAAS, experimentation **truth** |
| Flink/Kafka Streams/ksqlDB | Spark/BQ/warehouse |
| Approximate | Auditable, reconciled |

**Tradeoff:** Real-time is **faster feedback**; batch is **cheaper and more correct** for billing-grade numbers. Converged architectures often **dual-write** to stream + warehouse with **reconciliation** job.

## 3.9 Failure modes & degradation (expect these questions)

- **Campaign store down:** serve **house ads**, **cached snapshot**, or **blank**? Business call; show you’d align with legal/revenue.  
- **Pacing service slow:** **fail-open** (risk overspend) vs **fail-closed** (risk underdelivery)?  
- **Experiment service down:** default to **control**, or **freeze assignment** from cache?  
- **Logging saturated:** **sample** events, prioritize **billing-critical** vs **analytics**; backpressure producers.  
- **Partial region outage:** **drain** traffic; **health-checked** failover — **RTO/RPO** language.

## 3.10 Security & abuse (possible Staff angle)

- **Bot traffic**: rate limits, device attestation (conceptual), anomaly detection on impressions.  
- **Click fraud**: patterns, invalid traffic filters, partner integrations.  
- **SSRF** on ping/tracking URLs if server-side fetch.  
- **PII minimization** in logs and **GDPR/CCPA** delete workflows (high level).

---

# Part 4 — High-throughput events, Kafka, Spark, “petabyte” analytics

## 4.1 Kafka: question bank

- **Partitions**: How do you choose count? Relationship to **parallelism** and **ordering**?  
- **Key choice**: Implications for **skew** (one mega-advertiser key)?  
- **Consumer groups**: **Sticky** assignment vs **cooperative** rebalancing? Pain during deploys?  
- **Exactly-once** semantics: **idempotent producer**, **transactions** — when worth the complexity?  
- **Retention** vs **compaction**: When is compacted topic appropriate?  
- **Out-of-order** within partition: can it happen? implications?  
- **Schema registry**: backward/forward compatibility; **Avro/Protobuf** evolution rules?

## 4.2 Kafka tradeoffs

| Dimension | One side | Other side |
|-----------|----------|------------|
| More partitions | Throughput, parallelism | More file handles, longer recovery, metadata cost |
| Bigger retention | Replay, debuggability | **Disk cost**, GC on broker |
| Strong ordering per key | Correct sequential processing | **Hot partition** bottleneck |
| Small messages | Low latency batches | Overhead dominates — **batching** tuning |

## 4.3 Stream processing tradeoffs

| Pattern | Good when | Watch out |
|---------|-----------|-----------|
| **Event-time** windows | Accurate metrics | **Late data**, complexity |
| **Processing-time** windows | Simplicity | Wrong under lag |
| **State stores** (Kafka Streams/RocksDB) | Sessionization | **Rebalancing** cost, disk |
| **Lambda** (stream + batch) | “Best of both” | **Two code paths**, reconciliation |

**Staff framing:** Define **correctness contract** (e.g., “metrics ±1% hourly; daily **ground truth** in batch”).

## 4.4 Spark (batch / structured streaming): question bank

- When **shuffle** happens; how **skew** breaks jobs; **`salting`** / **AQE** (conceptually).  
- **Small files** problem in data lake; **compaction** jobs.  
- **Incremental** processing: **watermark**, **merge** into warehouse tables.  
- **Z-order / partition pruning** for query cost.

## 4.5 Lake / warehouse tradeoffs (Iceberg / Delta / Hudi — conceptual)

| Concern | Tradeoff space |
|---------|----------------|
| **Consistency** | File layout + metastore atomic commits vs simplicity |
| **Upserts** | Merge-on-read vs copy-on-read |
| **Time travel** | Audit/debug vs storage |
| **Lineage** | OpenLineage/Marquez — **operational cost** vs compliance |

## 4.6 Forecasting & yield support pipelines (JD-aligned)

They may not ask you to derive **yield** formulas; they want **platform thinking**.

**Possible questions**

- How would you land **inventory**, **historical delivery**, **constraints** into features for models?  
- How to **version** training data and **replay** old forecasts for debugging?  
- **Separation**: training features vs **online serving** — **point-in-time** correctness to avoid leakage.

**Tradeoffs**

| Batch feature store | Real-time features |
|---------------------|-------------------|
| Simpler, cheaper | Lower latency decisions |
| Stale relative to “now” | **Complexity**, **consistency** |

---

# Part 5 — API-first platforms, contracts, versioning

## 5.1 Question bank

- How do you **version** REST paths vs headers vs **media type**?  
- **Breaking change** policy: deprecation windows, **sunset headers**, dual-stack period?  
- **Idempotency-Key** for **POST** mutations; **retry** semantics.  
- **Pagination**: cursor vs offset at scale; **rate limiting** per partner tier.  
- **Async callbacks** vs **polling** for long operations (trafficking, approvals).  
- **GraphQL** for aggregation vs **BFF** — when not appropriate?

## 5.2 REST vs gRPC vs events

| Style | Pros | Cons |
|-------|------|------|
| **REST + OpenAPI** | Universal, cacheable, easy partner onboarding | Chatty; schema drift |
| **gRPC** | Efficient, strong contracts | L7 proxies, browser limits |
| **Events (async)** | Loose coupling | **Eventually consistent**, harder debugging |

**Staff:** **Sync for command/query** on critical path; **events** for **propagation** and **analytics**.

## 5.3 Backward compatibility tactics

- **Additive** fields only; never rename/remove without version bump.  
- **Enum** handling: accept unknown values.  
- **Feature flags** to roll out **server** support before **clients** adopt.  
- **Consumer-driven contracts** (Pact-style) in CI for internal services.

---

# Part 6 — Platform modernization & legacy broadcast/scheduling

## 6.1 Question bank

- How do you **strangle** a monolith: **facade**, **routing**, **data** migration order?  
- **Dual-write** vs **dual-read** vs **event CDC** — when each?  
- **Cutover**: **feature flag**, **shadow traffic**, **canary** by market?  
- **Rollback** when downstream **finance** already consumed events?  
- **Organizational**: **pizza teams** owning domains; **platform** vs **product** tension?

## 6.2 Tradeoffs: monolith vs microservices

| Monolith / modular monolith | Microservices |
|-----------------------------|---------------|
| Simpler deploys early | Independent scale and ownership |
| Shared DB transactions | **Distributed** sagas, **eventual consistency** |
| Team small | Team large; **clear bounded contexts** needed |

## 6.3 Linear TV modernization angles (JD “major plus”)

**Possible questions**

- How do **linear avails**, **traffic instructions**, **log-based reconciliation** relate to **digital** workflows?  
- **Human-in-the-loop** steps you would automate last (risk/compliance)?  
- **Canonical schedule**: who owns truth when OTA, streaming, and **FAST** diverge?

**Tradeoffs**

| Big-bang replatform | Incremental |
|---------------------|-------------|
| Clean model | High risk |
| Long freeze | **Continuous** value if strangler works |

---

# Part 7 — Distributed systems, reliability, cost

## 7.1 HA & “four nines” question bank

- **Multi-AZ** vs **multi-region**: **RPO/RTO**, **CAP** tradeoff in practice.  
- **Quorum** vs **single leader** replication.  
- **Circuit breakers**: when to **half-open**?  
- **Timeouts**: how to pick values; **cascading failure** without them.  
- **Bulkheads**: thread pools per dependency.  
- **Chaos** / **game days**: what you'd run.

## 7.2 Tradeoff matrix (common Staff prompt)

| Optimize for | Sacrifice | Mitigation |
|------------|-----------|------------|
| Latency | Consistency | Cache, **read-your-writes** only where needed |
| Cost | Freshness | Longer TTL, batch |
| Availability | **Linearizability** | **Eventual** models, conflict resolution |
| Velocity | Safety | **Progressive** rollout, automated rollback |

## 7.3 Observability question bank

- **RED** (rate, errors, duration) for services; **USE** for resources.  
- **SLO** examples for ad path: **availability**, **latency p99**, **fill rate** if applicable.  
- **Burn rate** alerting on error budget.  
- **Tracing**: **sampling** at high QPS; **tail sampling** for errors.  
- **Log** cost control: **structured** logs, **dynamic** log levels.

---

# Part 8 — Operational & Engineering Excellence (Principal-style)

## 8.1 Question bank (behavioral + technical)

- Tell me about a **Sev-0/1** you led. What was **comms cadence**?  
- How do you run **postmortems** — **blameless** mechanics?  
- Give an example where **error budget** **blocked** a launch.  
- How do you balance **feature throughput** vs **debt** with Product?  
- What **quality gates** exist before **prod**? What would you add at WBD?  
- How do you **mentor** seniors differently from juniors?  
- **Disagree and commit** story with a **Principal/Dir** peer.  
- How do you **measure** reliability improvement quarter over quarter?

## 8.2 Tradeoffs in operations

| Culture / practice | Tradeoff |
|--------------------|----------|
| **Strict** gates | Slower initial deploys; fewer outages |
| **Frequent** deploys | Smaller blast radius; needs **automation** |
| **Everyone on-call** | Ownership; **burnout** if unfair rotation |
| **Specialized SRE** | Expert incident response; **knowledge silo** risk |

## 8.3 SLO examples you can adapt

| SLI | Typical target (illustrative only) |
|-----|-------------------------------------|
| Decision API success (non-5xx) | 99.95% monthly |
| p99 latency | e.g. `< 80ms` **server-side** |
| **Logging pipeline** lag | < N minutes at p95 |

Always say you’d **calibrate with business** — Staff don’t invent numbers in a vacuum.

---

# Part 9 — Staff / leadership behaviors

## 9.1 Question bank

- **Influence without authority** — example.  
- **Ambiguous** ask from Product — how you narrowed scope.  
- **Tech debt** you championed; how you **quantified** ROI.  
- **Conflict** between two teams you resolved.  
- **Mentorship** outcome with measurable growth.  
- **Diversity of thought** / inclusive design review (modern loops).  
- **Ethical** or **privacy** pressure — how you escalated.  
- **Scaling yourself**: what you **stopped** doing as you grew.

## 9.2 STAR outline (repeat per story)

- **Situation** (1–2 sentences, business context)  
- **Task** (your responsibility)  
- **Action** (specific: architecture, data, people process — not “I worked hard”)  
- **Result** (**metrics**: latency −X%, incidents −Y, $ saved, time-to-market)  
- **Learning** (one sentence — humility + growth)

---

# Part 10 — AdTech domain probes

**New to Ad Tech?** Use the beginner track (separate docs, plain English, in depth): open [`adtech-beginner-guide/00-START-HERE.md`](./adtech-beginner-guide/00-START-HERE.md).

| Prep checklist (10.1 / 10.2) | Beginner guide file |
|------------------------------|---------------------|
| Ad serving + A/B testing mental model | [01-ad-serving-and-ab-testing-basics.md](./adtech-beginner-guide/01-ad-serving-and-ab-testing-basics.md) |
| Impression, click, CTR, CPM, CPC, CPA | [02-metrics-impressions-clicks-CPM-CPC-CPA.md](./adtech-beginner-guide/02-metrics-impressions-clicks-CPM-CPC-CPA.md) |
| First-party vs third-party; household vs device | [03-data-first-party-third-party-household-device.md](./adtech-beginner-guide/03-data-first-party-third-party-household-device.md) |
| Frequency cap, daypart, geofence | [04-targeting-frequency-daypart-geofence.md](./adtech-beginner-guide/04-targeting-frequency-daypart-geofence.md) |
| Ad server vs SSP vs DSP | [05-ad-server-SSP-DSP-programmatic.md](./adtech-beginner-guide/05-ad-server-SSP-DSP-programmatic.md) |
| Brand safety, ad verification | [06-brand-safety-ad-verification.md](./adtech-beginner-guide/06-brand-safety-ad-verification.md) |
| Forecasting vs pacing vs yield | [07-forecasting-pacing-yield.md](./adtech-beginner-guide/07-forecasting-pacing-yield.md) |
| Linear: avail, pod, makegood, traffic log reconciliation | [08-linear-tv-terms-avail-pod-makegood.md](./adtech-beginner-guide/08-linear-tv-terms-avail-pod-makegood.md) |
| Reach vs frequency; precision vs scale; analytics vs privacy | [09-tradeoffs-reach-frequency-privacy-precision.md](./adtech-beginner-guide/09-tradeoffs-reach-frequency-privacy-precision.md) |

**After the beginner track:** Read [`staff-deep-dive-guide/03-system-design-ad-serving-experiments.md`](./staff-deep-dive-guide/03-system-design-ad-serving-experiments.md) or **Part 3** below for system design depth (caching, events, experiments at scale).

---

# Part 11 — Coding (reported pattern + extensions)

## 11.1 Reported problems (see PracHub links in Part 1)

- **Array rotation** L/R by `k`, `k % n`, empty array.  
- **CSV keyword search**: row/col/cell offset, row-major; **multiple keywords**.

## 11.2 Likely follow-ups & edge cases

- **Rotation**: in-place O(n) triple-reverse vs O(n) extra memory — **state preference**.  
- **CSV**: overlapping matches in same cell; empty rows; trailing newline; Unicode (mention if relevant).  
- **Multi-keyword**: **Aho-Corasick** vs repeated scan — **when each**.

## 11.3 Complexity you should verbalize

- Rotation: O(n) time, O(1) or O(n) space.  
- Naive substring per cell: O(rows × cols × cell_len × key_len); AC automaton: linear in total text length for many patterns with preprocessing.

---

# Part 12 — Alternate system design prompts (practice variants)

For each, prepare: **requirements**, **CAP-style tradeoffs**, **data model**, **scaling**, **failure modes**, **observability**, **cost**.

1. **Billions of impression events** → near-real-time aggregates + daily truth in warehouse.  
2. **Multi-tenant internal API** for planners + external partners (versioning, quotas).  
3. **Notification / fanout** system (less AdTech but tests async design).  
4. **Rate limiter** at edge (token bucket vs sliding window; distributed).  
5. **Feature store** for online inference or ranking features — online/offline consistency.

---

# Part 13 — Questions you ask the interviewer (expanded)

Pick 3–5 per session.

**Org & scope**

- What are the **top three technical risks** for this org in the next year?  
- How does **Staff** interface with **Product**, **Data Science**, and **Ops** in practice?

**Architecture**

- **Canonical systems** for campaign truth vs **edge caches**?  
- How do **linear and digital** stacks **converge** on data models today?

**Reliability**

- Current **SLOs** and **biggest misses**?  
- **Incident** review culture — **blameless** examples?

**Modernization**

- Largest **legacy** dependency; what **strangler** work is underway?

**Growth**

- **90-day** vs **1-year** success for this hire.  
- **Promo** / impact expectations for Staff level here.

---

# Part 14 — Technical refresher checklist

- [ ] Load shedding, rate limits, backoff, **bulkhead**  
- [ ] Caching: stampede, TTL, aside vs write-through vs write-behind  
- [ ] DB: sharding, hot partitions, replicas, **CQRS** when justified  
- [ ] Messaging: outbox, **saga** vs choreography, DLQ  
- [ ] Observability: SLIs/SLOs, burn rate, **tail** sampling  
- [ ] Security: mTLS between services, secrets rotation, **least privilege**  
- [ ] Cost: Kafka retention, **S3** lifecycle, Spark **cluster** rightsizing, **egress**

---

# Mindset (closing)

Staff loops reward **structured thinking**: *clarify → options → tradeoffs → recommendation → rollout → observability*. Use this document to **pre-compute** tradeoff dimensions so live prompts feel like **instances** of patterns you already understand.

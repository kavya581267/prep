# High Availability, Reliability, Observability, Cost — Conceptual Depth

---

## 1. Availability arithmetic (actually do the math)

**Monthly uptime % → allowed downtime:**


| SLO    | Approx max downtime / month |
| ------ | --------------------------- |
| 99%    | ~7h 18m                     |
| 99.9%  | ~43m                        |
| 99.95% | ~21m                        |
| 99.99% | ~4m 20s                     |


**Staff:** Connect **composite** SLO to components: if decision API depends serially on **A** and **B**, **worst-case** availability multiplies (roughly). You **cannot** chain five **99.9%** dependencies naively and claim **99.9%** for the path without **parallelism** and **fallbacks**.

**Error budget:** If monthly Succeed **≥ 99.95%**, budget for **bad** requests is **0.05%**—about **1 in 2000**. Product + Eng debate: ship risky change **now** (burn budget) or **wait**?

---

## 2. CAP — use it correctly in interviews

**CAP** applies to a **partition** (network split), not “CAP says we use NoSQL.”

During **partition**, choose:

- **CP:** sacrifice **availability** (some nodes refuse writes)—money ledgers often prefer **strong** consistency per shard.  
- **AP:** sacrifice **strong consistency** (divergent reads)—some **edge** caches, **feature** flags.

**Reality:** **Latency** is often the hidden fourth dimension—many systems **timeout** rather than wait indefinitely, which **behaves** like **partition**.

**Staff answer:** “We don’t ‘pick CAP’ in abstract—we choose **per operation**: **authoritative** spend commits are **CP** on the ledger row; **read** caches are **AP** with TTL.”

---

## 3. Timeouts, retries, and cascading failure

**No timeout** means one slow dependency holds threads until **pool** exhaustion—**cascading** outage.

**Picking timeout values (process, not magic):** Measure dependency **p99** under load; timeout **slightly above** p99 if business accepts rare failure; **below** if you prefer **fail fast** and **degrade**. Align with **SLA** to upstream.

**Retries on GET** may be safe; **POST** only with **idempotency**. **Retry budgets** per request—don’t **multiply** retries at every layer (**retry amplification**).

---

## 4. Circuit breaker (state machine)

States: **Closed** (normal) → **Open** (fail fast after threshold) → **Half-open** (trial requests).

**Staff tuning:** Half-open **probe** count, **sliding window** error rate vs **count**-based, **different** thresholds for **depends** vs **optional** features.

**Anti-pattern:** Breaker **chatters** (flaps) if thresholds wrong—**cooldown** + **hysteresis**.

---

## 5. Bulkheads

Isolate **thread pools** / **connection pools** per dependency so one slow **S3** client cannot exhaust **all** threads for **database**.

**Tradeoff:** Total **max** throughput per instance may **drop** because you **reserved** capacity—but **blast radius** shrinks.

---

## 6. Multi-AZ vs multi-region

**Multi-AZ (one region):** Survive **datacenter** loss; **sync** replication often **sync** within region for managed DBs—**low RPO**.

**Multi-region:** **RPO** often **minutes** with async replication unless **very expensive** sync cross-region writes.

**Active-active:** **Conflict resolution**, **routing**, **sticky** sessions—**Staff** complexity tax.

---

## 7. Observability — three signals, used properly

### Metrics

**RED** for request services: **Rate**, **Errors**, **Duration** (histogram, not just avg).  
**USE** for resources: **Utilization**, **Saturation**, **Errors**.

**Cardinality:** `http_requests_total{campaign_id="..."}` at millions of campaigns **explodes** Prometheus/CW cost—**aggregate** to **top-N** or **sample** high-cardinality labels.

### Logs

**Structured** JSON; **correlation_id** propagation; **dynamic sampling** (100% errors, 1% success).

### Traces

Show **critical path**. **Tail sampling:** keep **slow** and **error** traces always—**crucial** for high-QPS ad paths.

---

## 8. SLO implementation details

**SLI specification must be testable:** e.g., “HTTP 200/201 within **edge** excluding 429” vs “including” 429—**define**.

**Burn rate alerts (idea):** Fast burn = **budget** consumed **too quickly** early in window—**page** before **month** is wrecked.

---

## 9. Cost engineering (concrete levers)

- **Kafka:** retention **hours** vs **years**—**tiered** storage if available.  
- **Logs:** drop **debug** in prod by default; **PII** fields hashed once.  
- **Data transfer:** **same-region** reads for Spark; avoid **cross-AZ shuffle** when possible (infra dependent).  
- **Idle clusters:** autoscale to zero for **pre-prod** where safe.

**Staff:** “We **attribute** cost per **tenant** and **product** line—reliability work competes with **$** visibility.”

---

## 10. Chaos and game days

**Game day:** **Inject** dependency failure **in non-prod** then **graduated** prod with safeguards.  
**Goal:** **Verify** runbooks, **monitoring** gaps, **human** response—not machismo.

---

## 11. Worked examples (SLO math, dependencies, retries)

### Example A — Serial dependencies multiply outage risk

Suppose decision path calls **segment** then **pacing** **serially**, each **independently** **99.95%** monthly available.

**Approximate** joint success ≈ `0.9995 × 0.9995 ≈ 0.9990` (**99.90%**)—you **lost** half your error budget **silently** if you promised **99.95%** for the **whole** path.

**Fix:** **parallelize** independent calls; **cache**; **circuit break** optional segment with **safe default**.

### Example B — Error budget to incidents

**SLO:** 99.9% success monthly (~**43m** bad minutes **budget** in minute-based mental model—not exact but **directional**).

**Bad deploy** causes **0.4%** errors for **20 minutes** at **50k QPS** → huge **relative** budget burn—**auto rollback** when **5xx** > **0.1%** for **5m**.

### Example C — Picking a timeout (numeric intuition)

Dependency **p99** latency **stable** **35 ms**, **p99.9** **180 ms** (fat tail).

**Choices:**

- Timeout **50 ms** → **lots** of false timeouts on tail; angry users.  
- Timeout **250 ms** → absorbs tail; **risk** if dependency **hangs** forever—use **breaker**.

**Staff:** “We’d set **250 ms** timeout, **50 ms** **hedged** **optional** second call **only** for **premium** tier—product decision.”

### Example D — Retry amplification

**Client** **3** retries × **Edge** **2** retries × **Service** **2** retries → effective **12×** load on **deep** failure.

**Fix:** **retry budget** header `X-Retry-Count`; **idempotent** keys; **max** **1** retry at edge.

### Example E — Cardinality cost

Metric: `ad_decisions_total{placement="home_banner"}` → **low** cardinality, **fine**.

Metric: `ad_decisions_total{campaign_id="..."}` with **500k** active campaigns → **millions** of **series**; **Prometheus** **OOM** or **$** **thousands**/month in SaaS vendors.

**Fix:** **top-N** by revenue **pre-aggregated** stream; **sample** long tail into `campaign_id="other"`.

### Example F — Tail sampling

At **100k** QPS you **cannot** store **all** traces.

**Policy:** keep **100%** of traces where `status=error` OR `latency > 500ms`; **0.1%** of happy paths—still **statistically** useful for **dependency** latency percentiles **if** engine supports **exemplars** + metrics.

---

## Related

- Patterns expanded: [13](./13-resilience-patterns-catalog.md)  
- OE stories: [08](./08-operational-engineering-excellence.md)


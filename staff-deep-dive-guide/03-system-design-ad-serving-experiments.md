# System Design: Ad Serving + A/B Testing — Conceptual Depth

Vocabulary: [`../adtech-beginner-guide/`](../adtech-beginner-guide/00-START-HERE.md). This file explains **mechanics** and **Staff-level reasoning** for the common interview prompt.

---

## 1. What you are really building

An **ad decision service** is a **low-latency rule engine + ranking function** glued to **durable accounting**. Mistakes are measured in **dollars** (overspend, makegoods) and **trust** (wrong experiment conclusions). Your design must name **three contracts**:

1. **Latency contract:** p99 server-side budget per hop.  
2. **Durability contract:** what happens if logging is slow/down.  
3. **Correctness contract:** idempotent **request_id**, budget updates, experiment exposure semantics.

---

## 2. Clarifying questions (why each matters)

**QPS and read/write split:** Decision path is **read-heavy** (campaigns, rules, counters read); logging path is **write-heavy** and **async** in most architectures. Splitting them prevents **coupling** failures: if Kafka hiccups, you may still **serve** ads with degraded analytics—not always allowed, but **explicit**.

**Latency SLO:** If p99 is 80ms **total**, you back out budgets: e.g., **20ms** edge, **40ms** decision + data fetches, **20ms** margin. Makes **sync vs async logging** mechanical, not philosophical.

**Multi-region:** **Active-active** decisioning doubles **cache coherence** pain; **active-passive** or **regional partitions** (EU serves EU) simplify **GDPR** and latency. Staff: pick one and defend **RPO** for config/campaign propagation.

---

## 3. End-to-end serving path (mechanical)

### Step 1 — Identity

You need a **stable key** for frequency caps and experiments.

- Logged-in: account id.  
- Anonymous: device id, or **first-party cookie**, or **synthetic** id from hashed signals—**privacy** policies constrain you.

**Staff:** “We document **cold start** behavior: anonymous user gets **random** experiment bucket until id stabilizes; analysis excludes **tiny** fraction or uses **intent-to-treat** variant (advanced).”

### Step 2 — Experiment assignment

**Deterministic bucketing:**  
`slot = hash( experiment_id + salt + stable_user_key ) % 100`  
Map `slot` to cumulative **percentages** of variants (e.g., 0–49 control, 50–99 treatment).

**Why salt?** Prevents **accidental** correlation across experiments if keys align badly.  
**Why not random per request?** User **flips** variants → **contaminated** metrics.

### Step 3 — Candidate retrieval

Naive: scan all campaigns—**never** at scale.

**Conceptual model:** Inverted index: `placement_id → list of line items` plus coarse **predicates** (geo, allowed device). Retrieval returns **tens or hundreds** of candidates, not millions.

**Staff follow-up:** **pre-filter** using cheap keys (whitelist placement, geo bloom filter) before hitting **segment** services.

### Step 4 — Hard filters

Brand safety, **frequency cap** (read counter), **daypart** (local timezone!), legal constraints.

**Frequency read:** Typically remote **GET** to counter service. **Tradeoff:** **approximate** Bloom-filter pre-check vs exact counter—false positives block good ads; false negatives break promises.

### Step 5 — Rank / select

Could be **eCPM** comparison, **learned model** score, **epsilon-greedy** exploration (small % random for data). Experiments often **swap** this function.

### Step 6 — Exposure log

**Exposure** = user **eligible** and **assigned** treatment; may differ from **render** if client fails. Define consistently.

**Sync write:** Stronger **audit** story; risks latency.  
**Async:** Return fast; risk **loss**. Mitigations:

- **Durable buffer** (local disk queue—complex).  
- Client-side **beacon** retry with **idempotency**.  
- **Reconciliation** from CDN **access logs** (delayed, messy).

### Step 7 — Response

Return VAST/VMAP, creative metadata, **click URL** through **redirector** for counting.

---

## 4. Idempotency and money (deep)

**Scenario:** Client retries POST. Two impressions for one **view** → **double** bill.

**Pattern:**

- **request_id** generated **once** client-side or by edge on first touch.  
- Ingress **dedupes** in **hot store** (short TTL) or **writes** to **ledger** with unique constraint.  
- Budget consumer **subtracts** idempotently: `UPDATE budget_consumed SET v = v + cost WHERE id = :rid AND not_applied`.

**Staff:** Mention **exactly-once** is **business** property assembled from **at-least-once transport** + **idempotent domain**.

---

## 5. Caching — beyond “use Redis”

**What is cached:**

- **Campaign JSON** / compiled rules (changes rarely relative to QPS).  
- **Segment membership** (changes more often—TTL shorter).  
- **Kill switch** state (must propagate fast—**push** invalidation or **very short** TTL).

**Stampede:** On TTL expiry, 500 threads **miss** together. **Single-flight** (one recomputes, others wait), **probabilistic early refresh**, **jitter**.

**Legal takedown:** **Override** path must bypass cache or broadcast **purge**—tie to **feature flag** evaluated **before** cache return on sensitive paths.

---

## 6. Pacing and counters (operational reality)

**Idealized pacing:** Spend **evenly** over flight. **Reality:** traffic is **lumpy**, auction prices **unknown**.

**Central counter** (Redis cluster):

- **INCR** spend or impressions with Lua for **check-and-increment** under cap.  
- **Hot key** = blockbuster campaign—**shard** counter internally (`campaign:123:shard7`) and **sum** for reads, or use **Hierarchical** time buckets.

**Fail-open vs fail-closed** when counter service degrades:

- **Fail-closed:** stop serving that campaign—**underdelivery**.  
- **Fail-open:** serve—**overspend** risk—finance escalation.

**Staff:** “I’d get Product/Finance sign-off; default **fail-closed** for **guaranteed** deals, **fail-open** sometimes for **remnant**—but that’s business policy.”

---

## 7. Experiments — statistics + engineering interplay

**Simpson’s paradox:** Treatment looks better overall but worse in every segment because of **mix shift**.

**Interference:** Two experiments both touch **ranking**; effects **interact**. Mitigate with **layers** (orthogonal experiment keys affecting disjoint code paths—simplified) or **mutual exclusion**.

**Guardrails:** Automated rollback if **global** revenue down > **k**% for **m** minutes—requires **trusted** metrics pipeline with **low** false positive rate.

---

## 8. Failure isolation (talk track)

| Failure | User-visible | Business choice |
|---------|--------------|-----------------|
| Campaign DB | Stale ads or no ads | **Cached snapshot** age bound |
| Segment service | Degrade targeting | Fall back to **contextual only** |
| Logging saturated | N/A immediate | **Sample** **non-billing** events |

Always pair with **feature flag** to **drain** traffic from bad region.

---

## 9. Worked examples (concrete walkthroughs)

### Example A — Latency budget on paper

**Assumptions (you’d confirm in interview):**

- Client hard timeout **120 ms** (CTV player); you own **80 ms** server-side.  
- Peak **25k** decision QPS / region.

**Sample budget:**

| Stage | Budget |
|-------|--------|
| API gateway + TLS termination | 5 ms |
| Auth / API key validation | 3 ms |
| Load experiment + user context from cache | 8 ms |
| Candidate retrieval (index + segment check) | 25 ms |
| Pacing / freq counter read (Redis) | 12 ms |
| Rank (small model or heuristic) | 15 ms |
| Assemble VAST JSON | 5 ms |
| **Buffer** (spikes, GC) | 7 ms |
| **Total** | **80 ms** |

If segment service **p99** grows to **40 ms**, you **must** cache segments harder, **parallelize** fetches, or **raise** SLO—Staff shows you **trade** product features for numbers.

### Example B — One request, end-to-end (made-up IDs)

**Input:** `placement_id=home_banner`, `user_device_id=device-abc-889`, `country=US`, `local_hour=20` (Eastern).

1. **Identity:** `stable_key = device-abc-889` (anonymous CTV).  
2. **Experiment `exp_rank_v2`:**  
   `hash("exp_rank_v2" + salt + "device-abc-889") % 100 → 73` → treatment bucket **B** (positions 50–79 in your table).  
3. **Index lookup:** `home_banner` → candidates `[line_101, line_404, line_990]`.  
4. **Filters:**  
   - `line_990` paused → out.  
   - `line_404` geo US ✓; freq cap: **2/24h**, current count **1** ✓.  
   - `line_101` daypart wrong → out.  
5. **Rank:** Under **B**, use **model v2** scores; winner **`line_404`**, creative **`cr_video_12`**.  
6. **Exposure event (async):** Kafka record  
   `{request_id:"req-7781", campaign:404, variant:"B", ts:...}`  
7. **Response:** VAST URL + tracking beacon URLs; client fires beacon on **render**.

**Talking point:** Same `request_id` retried → impression consumer **dedupes** before incrementing `imps_today` for freq cap.

### Example C — Idempotent impression + budget (pseudocode)

**Table `impression_ledger`:** unique on `request_id`.

```text
BEGIN;
INSERT INTO impression_ledger (request_id, campaign_id, price_micros, ts)
VALUES ('req-7781', 404, 150000, now())
ON CONFLICT (request_id) DO NOTHING;
IF FOUND (row inserted) THEN
  UPDATE campaign_spend SET spent_micros = spent_micros + 150000 WHERE id = 404;
END IF;
COMMIT;
```

**Staff:** In practice this may live in **stream processor** + **data warehouse**, but the **idea** is: **ledger insert** is the **gate** for “count this money once.”

### Example D — Hot campaign pacing shard

**Campaign 404** spends **\$50k/day** and dominates counter traffic.

**Naive key:** `pacing:campaign:404` → single Redis key **melts**.

**Shard pattern:** 16 shards; for each decision:

```text
shard = hash(device_id) % 16
INCR pacing:campaign:404:shard:{shard} BY estimated_spend
```

**Read path:** `SUM(shard 0..15)` cached **every second** for soft throttle, or **accept** small **inaccuracy** with **hourly** reconciliation from **batch** truth.

### Example E — Simpson’s paradox (tiny numbers)

**Overall CTR:** treatment **wins**.

| | Control CTR | Treatment CTR |
|--|------------|----------------|
| All users | 1.0% | 1.2% |

**Split by device (made up):**

| Device | Control imps / clicks | Treatment imps / clicks |
|--------|------------------------|-------------------------|
| Phone | 100 / 2 (2%) | 50 / 0.5 (1%) |
| CTV | 50 / 0 (0%) | 100 / 1 (1%) |

Treatment is **worse on phones** and **worse on CTV** in this toy table—but if you **pour** treatment traffic into **CTV**, overall CTR **rises**. **Staff:** “We’d **stratify** by device and **day** before trusting global lift.”

---

## 10. Strong close (say slowly)

> “I’d separate decision **read** path from **write** path. Idempotent `request_id` end-to-end. Caching with jitter + forced miss on kill switch. Experiments: salted stable hash, guardrailed metrics. Stream for ops, batch for billing truth with reconciliation. Biggest operational risk is pacing hot keys—we’d shard counters and rehearsal failover for the counter plane.”

---

## Related

- Data plane detail: [04](./04-data-streaming-kafka-spark-lakehouse.md)  
- Partner APIs: [05](./05-api-first-platforms-contracts.md)

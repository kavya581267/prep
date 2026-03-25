# Data: Kafka, Streaming, Spark, Lakehouse — Conceptual Depth

**Purpose of this doc:** Explain *how* these systems behave under the hood (at the level Staff interviews expect), *why* teams choose one pattern over another, and *what* to say when an interviewer drills past buzzwords.

**Hands-on ops companion:** `04b-kafka-hands-on-ops.md` — CLI, lag, offsets, rebalance/URP runbooks, metrics.

---

## 1. The problem class (beginner → Staff)

**Beginner framing:** Modern ad and media stacks emit **events** continuously—ad requests, impressions, clicks, experiment exposures, server logs. That stream is too large to “query the database” as the primary ingestion path. You need a **pipeline** that:

1. **Buffers** spikes (traffic is bursty).  
2. **Decouples** writers (thousands of services) from readers (dozens of consumers).  
3. Lets different teams **consume the same history** at different speeds.  
4. Eventually lands **auditable** totals for **finance**.

**Staff framing:** You are not picking Kafka because it is trendy; you are choosing a **durability + fan-out + replay** contract. State that contract explicitly: *at-least-once + idempotent sinks*, or *exactly-once in a bounded scope with these failure modes*.

---

## 2. Kafka: what it actually is

Kafka is a **distributed append-only log** implemented as **topics** split into **partitions**. Each partition is an ordered sequence of **records** (messages). Ordering is **total** *within* a partition and **undefined** *across* partitions.

### 2.1 The life of one record

1. **Producer** sends a record to a **topic**. If a **key** is present, `hash(key) % num_partitions` chooses the partition. **No key** → round-robin or sticky partitioner (implementation detail).  
2. The **leader** broker for that partition appends the record to the **log segment** on disk (not “mostly in RAM”—Kafka’s durability story is **OS page cache + sequential write**, fsync policy depends on producer `acks` and config).  
3. **Follower** replicas pull from the leader (**ISR** = in-sync replicas). Only ISR members can become leader after failover, subject to `min.insync.replicas` semantics.  
4. **Consumers** in a **group** collectively claim partitions (each partition consumed by **one** consumer in the group at a time—classic model). The consumer **commits offsets** (positions in the log): “I have successfully processed up to offset N.”

**Interview depth:** If they ask “where is ordering guaranteed?” you answer: **per partition only**. If two events must never be reordered relative to each other, they **must share a key** (or you accept application-level sequencing).

### 2.2 Why partitions exist (conceptual physics)

Partitions are **unit of parallelism**:

- **Brokers** host many partitions; load spreads.  
- **Consumer group** parallelism ceiling ≈ **number of partitions** (for simple consume-one-partition-per-thread models). If you have 12 partitions and 48 consumer instances, **36 sit idle** for that group.

**Staff tradeoff:** More partitions increase **throughput headroom** and consumer scale-out, but:

- Each partition has files: **too many** partitions per broker → metadata overhead, longer **leader election** and recovery.  
- **End-to-end latency** can rise if batches wait to fill (producer `linger.ms` tuning).

There is no universal number; you derive from **peak ingress MB/s**, **consumer lag SLO**, and **max desired parallelism**.

### 2.3 Hot keys (the Staff trap)

If **key = campaign_id** and one campaign is 40% of traffic, one partition becomes **hot**: disk, network, and **single-threaded** consumer for that partition becomes the bottleneck.

**Mitigations (know the names and limits):**

- **Salting:** write `campaign_id||salt` as key for ingest, aggregate downstream—loses per-campaign strict ordering in the raw topic; you regain order in a **second-stage** topic or batch job.  
- **Split campaigns** across shards at **business** level (rare).  
- Accept **ordering only where needed**—don’t key everything on the skewed dimension.

### 2.4 Consumer groups and rebalances

When consumers join/leave/crash, **partitions are reassigned**. During rebalance, processing pauses—**stop-the-world** in naive clients. Modern clients use **cooperative sticky** rebalance to **migrate** fewer partitions at a time.

**Staff answer:** “We’d canary consumer deploys, tune `max.poll.interval.ms` so heartbeats don’t falsely mark processors dead, and measure **rebalance storm** during deploys as a release gate.”

### 2.5 Delivery semantics (say this clearly)

Kafka does not magically give you “exactly-once **end-to-end**” to a database without your cooperation.

- **At-most-once:** commit offset before processing → **lose** messages on crash. Rarely acceptable for money.  
- **At-least-once (common):** process, then commit; on crash you may **reprocess** → **idempotent** sink (`INSERT ... ON CONFLICT`, natural keys, or dedupe table).  
- **Exactly-once within Kafka (transactions):** transactional producer + read-process-write patterns—**complexity** and **performance** cost; justified when duplicate **side effects** are unacceptable *and* you cannot make sinks idempotent easily.

**Staff pattern:** “We default **at-least-once + idempotent writes** keyed by `request_id` because it is operable; we’d consider transactions only for this narrow join where duplicates corrupt ledger.”

### 2.6 Retention vs compaction

- **Retention (time/size):** old log segments **delete**—great for firehose metrics.  
- **Compaction:** for **changelog** topics where **latest value per key** matters (e.g., config snapshots). **Not** for raw impression stream (billions of unique keys → huge compaction cost).

### 2.7 Schema registry (why interviewers mention Avro/Protobuf)

Schemas evolve. **Backward compatibility** means *new consumers can read old data*; **forward** means *old consumers can read new data* (trickier). Rules like “only add optional fields” prevent production explosions.

**Staff:** “CI enforces compatibility on PR; producers fail fast if schema incompatible.”

---

## 3. Stream processing: windows, time, and watermarks

**Beginner:** You want **counts per minute** or **sessionized** click streams. The processor reads from Kafka, maintains **state** (counts, buffers), emits results.

### 3.1 Event time vs processing time

- **Event time:** timestamp **in the record** (when the impression happened).  
- **Processing time:** clock on the **processor** when it sees the record.

If the pipeline **lags**, processing-time windows **lie**. For **reporting** you usually want **event time**, accepting **complexity**.

### 3.2 Windows (intuition)

- **Tumbling:** fixed 1-minute buckets, disjoint.  
- **Sliding:** every 10s emit count over last 60s—overlapping.  
- **Session:** gaps of inactivity define session boundaries—needs **state** and handles **out-of-order** events.

### 3.3 Late data and watermarks

**Watermark** = estimate: “we believe we’ve seen all events with event-time ≤ W.” Events **later than** W may be dropped to “side output” or counted separately.

**Staff tradeoff:**

| Strict correctness | Low latency |
|--------------------|------------|
| Wait longer for lates | Emit fast but **approximate** |

State the product contract: ops dashboard may be **approximate**; finance batch **recomputes** from raw log.

### 3.4 Stateful processing and rebalance

Frameworks (Flink, Kafka Streams) store state in **RocksDB** locally. On rebalance, state **must migrate**—expensive. **Staff:** partition assignment stability, incremental repartition planning, and **rocksdb** tuning matter in production—not just PoC.

### 3.5 Lambda architecture (historical but still named)

**Speed layer** (stream) + **batch layer** (nightly truth). **Pain:** two code paths. Modern compromise: **Kappa-ish** stream with **periodic re-batch** reconciliation from **raw** landing zone.

---

## 4. Spark (batch and structured streaming): how work actually runs

**Beginner:** Spark splits a job into **tasks** executed on executors. **Narrow** transformations (map, filter) are pipelined. **Wide** transformations (groupBy, join) cause **shuffle**: all partitions send data to **shuffle** partitions based on key—**all-to-all** traffic.

### 4.1 Why shuffle is expensive

Shuffle writes **spill** files, sorts, merges—**CPU**, **disk**, **network**. Skew means one task gets **most** of a key’s data—**straggler** dominates job time.

**Mitigations:**

- **Salting** in join keys for hot keys, then **unsalt** in second pass.  
- **Adaptive Query Execution** (Spark 3): runtime shuffle partition coalescing, skew join hints—know conceptually.  
- **Broadcast join** when one side is tiny—avoids shuffle on the big side.

### 4.2 Small files problem

Many tiny files in object storage → **list** overhead, poor **vectorized** read. **Compaction** jobs merge to target file size (e.g., 128–256MB) partition-wise.

### 4.3 Incremental / MERGE pipelines

For **warehouse** tables you often **merge** daily or hourly partitions: `MERGE INTO` on Iceberg/Delta/BigQuery—**idempotent** runs keyed by **`ingestion_batch_id`**.

---

## 5. Lakehouse formats (Iceberg / Delta / Hudi) — what problem they solve

**Beginner pain:** Plain Parquet directories in S3 **aren’t** atomic. Two writers can corrupt “latest” state; listing files is slow; **schema** evolution is ad hoc.

**Lakehouse** = table format with **metadata layer** tracking **snapshots**:

- **Atomic commit:** new snapshot points to new set of files.  
- **Time travel:** read table at **version T**—debug “what did finance see yesterday?”  
- **Upserts:** **delete + insert** file footprints—MoR (merge-on-read) vs CoW (copy-on-write) **latency vs write amplification** tradeoff.

**Staff:** “We’d pick Delta vs Iceberg based on **engine** support, **governance** tooling, and **portability** needs—not religion.”

---

## 6. Feature pipelines and point-in-time correctness (JD)

**Problem:** Training a model on “user’s total impressions” must not include **future** impressions relative to the training label time.

**Technique:** For each training example at time **t**, features must be computed with **only data ≤ t**. In practice: **snapshot** feature tables keyed by time, **join** with **as-of** semantics, versioned **feature code**.

**Online vs offline skew:** Serving path reads **Redis**; training read **warehouse**. Document **staleness** SLO (e.g., features up to 5 minutes behind) and **fallback** when store is down.

---

## 7. Pulling it together: AdTech-style pipeline (narrative)

1. **Ingress:** clients POST beacons → **validate** → produce to `impressions` topic (key salted if needed).  
2. **Stream:** aggregate to **minute** buckets for **ops** dashboard—watermarked, approximate.  
3. **Batch:** hourly job **dedupes** by `request_id`, loads **warehouse** partition `dt=2025-03-25/hour=14`.  
4. **Reconciliation:** compare stream **approx** vs batch **counts**; alert if **>0.5%** delta—tune threshold with finance.

**Staff closer:** “The correctness model is: stream for **actionability**, batch for **contract**; duplicates are a **fact** so sinks are idempotent.”

---

## 8. Worked examples (numbers + timelines)

### Example A — Partition count back-of-envelope

**Given:**

- Sustained ingress **200 MB/s** to a topic (compressed payload avg **800 B** → **~250k** msgs/s).  
- Target: each consumer instance handles **at most** **5k** msgs/s comfortably.  
- You want **12** consumer instances in the group for headroom.

**Ceiling parallelism** = **partition count** (classic one thread per partition). Need **≥ 12** partitions.  
If **12** partitions → **250k/12 ≈ 21k** msgs/s per partition—might be **hot** on broker disk. Bump to **48** partitions → **~5.2k/s/partition** broker-wise, **4 partitions/instance** if **12** consumers.

**Interview line:** “I’d validate with **load test** and **producer** batching settings; partition count is **reversible** but **expensive** to shrink—start conservative, **split** read path topics if needed.”

### Example B — Hot `campaign_id` → salting

Raw key **`campaign_id=999`** holds **35%** of traffic → partition **7** lag spikes.

**Ingest producer** switches key to:

```text
partition_key = hash(campaign_id + "_" + (random_shard_0_to_7))
```

**Loss:** You **cannot** rely on **per-campaign** strict order in the **raw** topic.

**Recovery:** Consumer **re-partitions** by `campaign_id` into downstream topic `impressions_by_campaign` with **fewer**, **larger** messages (micro-batch) **or** you only needed **global** counts anyway.

### Example C — Late event vs watermark (event time)

Events carry `event_ts` (when ad rendered).

**Watermark** at processing wall-clock **T=10:05:00** might be **`W = 10:04:30`** (wait **30s** for stragglers).

Record arrives with `event_ts=10:04:15` at **10:05:02** → **on time**, counted in window `10:04:00–10:05:00`.

Record with `event_ts=10:03:45` at **10:05:02** → **late** (watermark already past) → **side output** “late_bucket” + **batch** job **reconciles**.

### Example D — Spark skew toy

`GROUP BY campaign_id`. **`campaign_id=X`** has **900M** rows; others tiny.

One **reduce** task gets **900M** rows → **10 hours**; rest finish in **minutes**.

**Two-phase salt:**

1. Add `salt random 0..9` to key: `GROUP BY campaign_id, salt` → **10** groups for X, each **~90M** rows; executors finish in parallel.  
2. Second stage: `GROUP BY campaign_id` to **merge** partial sums.

### Example E — Daily reconciliation

**Stream** rolling sum **yesterday** for `imps_us_display`: **148,200,100**.  
**Batch** **truth** after dedupe: **147,950,000** (**~0.1%** gap).

**Action:** Alert **under** **0.5%** if ops SLO says stream is **approximate**; **page** finance adapter **over** **0.1%** if contract says “stream is **billing** **preview** only.” **Staff:** thresholds are **business** artifacts.

### Example F — Point-in-time feature (ML)

Training row: “**Did user subscribe on 2025-03-01?**”

Feature: `total_ad_impressions_last_7d` must use **only** impressions with `ts ≤ 2025-03-01 00:00`.

**Bad join:** naïve `SUM` over **all** rows in warehouse → **leaks future** after label—model looks **magical** in train, **fails** in prod.

**Good join:** `AS OF` timestamp versioned table or nightly **snapshots** `features_dt=2025-03-01`.

---

## 9. Question bank (still useful, now you can *answer* them)

- Why **not** put PostgreSQL on the critical ingest path for 100k events/sec?  
- When would you **increase** partitions midlife—what breaks?  
- How do you **replay** a consumer by offset—what risks for downstream idempotency?  
- Exactly-once to S3—what does that even mean without atomic filesystem semantics?

---

## Related

- **Dedicated Spark Q&A path:** [`../spark-guide/00-START-HERE.md`](../spark-guide/00-START-HERE.md)  
- **Batch ETL patterns (extract/load/orchestration):** [`../batch-etl-guide/00-START-HERE.md`](../batch-etl-guide/00-START-HERE.md)  
- Ad serving events tie-in: [03](./03-system-design-ad-serving-experiments.md)  
- Operational limits (lag alerts, cost): [07](./07-ha-reliability-observability-cost.md)  
- Message patterns: [13](./13-resilience-patterns-catalog.md)

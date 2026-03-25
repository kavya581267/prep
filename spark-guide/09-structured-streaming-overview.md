# Structured Streaming — In Depth (Batch-Comparison)

## Mental model: append-only table

Structured Streaming treats the input (Kafka topic, files landing in a directory, etc.) as an **unbounded input table**. Every trigger, Spark reads **new rows** appended since last **offset**, runs your query **incrementally**, and writes **output** **sinks**.

**Micro-batch** mode (default): each trigger processes a **batch** of new data **using** the **same Catalyst engine** as batch Spark—**code reuse** is a huge practical advantage.

## Triggers and latency

**Processing time trigger** `0` (as fast as possible) vs **fixed interval** `10 seconds`. **Tradeoff:** more frequent triggers → **lower** latency but **higher** **scheduling** overhead and **smaller** batches (less throughput efficiency).

**Staff question:** “What latency does the business need?” If answer is **minutes**, micro-batch Spark may suffice; if **sub-second complex event processing**, consider **Flink** / **dedicated** stream processor.

## Watermarking and late events

**Problem:** **event-time aggregations** (window by `event_ts`) need **state**: counts per window. If events can arrive **late**, state could grow **forever**.

**Watermark:** `withWatermark("event_ts", "10 minutes")` tells Spark “drop **state** for windows older than **max_event_time_seen - 10m**” (conceptual wording).

**Tradeoff:** **late** events **beyond** watermark are **dropped** or **routed** to a **side output**—explicit **product** decision.

Example **plain language:** “We’re OK losing **clicks** **>10m late** for **ops** dashboard; **batch** **corrects** yesterday **fully** at **2am**.”

## Checkpoint directory (non-negotiable for production)

Streaming query **state** (offsets, aggregations) is stored in a **checkpoint** location (HDFS/S3/ADLS). **If you delete checkpoint**, you **reset** or **corrupt** consistency—**treat** as **application database**.

**Upgrades:** changing query shape in **incompatible** ways may require **new** checkpoint path—**plan** **migrations**.

## Output modes

- **Append** — only **new** rows to sink (e.g., **Kafka** topic of results).  
- **Complete** — rewrite full result table each trigger (only feasible for **tiny** aggregations).  
- **Update** — changed rows only (for some aggregations).

**Interview:** “I choose **mode** based on **sink** **capabilities** (Idempotent **MERGE** vs **append-only** logs).”

## Fault tolerance and effectively-once (careful wording)

**Exactly-once** **end-to-end** depends on **sink** + **idempotent** writes + **transactional** commits. Many teams say **at-least-once** processing with **idempotent** sink (Delta **`MERGE`**, etc.)—**honest** engineering.

## Soundbite

> “Structured Streaming is **micro-batch** **Catalyst** over **append** **inputs**. I design **watermarks** from **late data** SLA, **checkpoint** like **state DB**, and I match **output mode** to **sink** semantics.”

Next: [10-failures-recovery.md](./10-failures-recovery.md)

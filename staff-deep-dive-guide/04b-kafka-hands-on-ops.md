# Kafka — Hands-On Operations

**Purpose:** Commands, checks, and runbooks you can run in staging or production (with change control). Pair with conceptual depth in `04-data-streaming-kafka-spark-lakehouse.md`.

**Assumptions:** Apache Kafka 3.x–style CLI (`kafka-topics`, `kafka-consumer-groups`, …). Paths vary by install (`/opt/kafka/bin/`, package managers, Confluent Platform layout). Replace placeholders:

- `BOOTSTRAP` — e.g. `broker1:9092,broker2:9092`
- `TOPIC` — topic name
- `GROUP` — consumer group id

Export once per shell session:

```bash
export BOOTSTRAP="broker1:9092,broker2:9092"
```

---

## 1. Verify connectivity

```bash
# List brokers the client can see (metadata sanity)
kafka-broker-api-versions.sh --bootstrap-server "$BOOTSTRAP"
# If your distribution drops the .sh suffix:
# kafka-broker-api-versions --bootstrap-server "$BOOTSTRAP"
```

If this fails: network/TLS/SASL/bootstrap list wrong before debugging topics.

---

## 2. Topics

### Create

```bash
kafka-topics.sh --bootstrap-server "$BOOTSTRAP" \
  --create --topic "$TOPIC" \
  --partitions 12 --replication-factor 3
```

### Describe (leader, ISR, offsets high watermark)

```bash
kafka-topics.sh --bootstrap-server "$BOOTSTRAP" --describe --topic "$TOPIC"
```

### List

```bash
kafka-topics.sh --bootstrap-server "$BOOTSTRAP" --list
```

### Alter config (retention example)

```bash
kafka-configs.sh --bootstrap-server "$BOOTSTRAP" --entity-type topics \
  --entity-name "$TOPIC" --alter \
  --add-config retention.ms=604800000
```

### Alter partitions (only **increase**)

```bash
kafka-topics.sh --bootstrap-server "$BOOTSTRAP" \
  --alter --topic "$TOPIC" --partitions 24
```

**Ops note:** Increasing partitions changes key distribution for **new** keys; it does **not** reshuffle existing data. Plan consumer parallelism and key semantics before widening.

### Delete (must allow topic deletion on brokers)

```bash
kafka-topics.sh --bootstrap-server "$BOOTSTRAP" --delete --topic "$TOPIC"
```

---

## 3. Quick produce / consume smoke tests

### Console producer

```bash
kafka-console-producer.sh --bootstrap-server "$BOOTSTRAP" --topic "$TOPIC"
```

### Console consumer (from latest or earliest)

```bash
# From next records onward
kafka-console-consumer.sh --bootstrap-server "$BOOTSTRAP" --topic "$TOPIC"

# From beginning (read full retained log)
kafka-console-consumer.sh --bootstrap-server "$BOOTSTRAP" --topic "$TOPIC" \
  --from-beginning
```

Add `--property parse.key=true --property key.separator=:` if testing keyed messages.

---

## 4. Consumer groups (lag is the headline metric)

### List groups

```bash
kafka-consumer-groups.sh --bootstrap-server "$BOOTSTRAP" --list
```

### Describe one group (LAG per partition)

```bash
kafka-consumer-groups.sh --bootstrap-server "$BOOTSTRAP" \
  --describe --group "$GROUP"
```

Interpretation:

- **LAG** — records not yet consumed (roughly `log-end-offset - current-offset`).
- **CURRENT-OFFSET** — last committed offset per partition for this group.

### Members (who is in the group)

```bash
kafka-consumer-groups.sh --bootstrap-server "$BOOTSTRAP" \
  --describe --group "$GROUP" --members --verbose
```

### Reset offsets (destructive — coordinate with owners)

**Shift offsets** after bad deploy or replay:

```bash
# Dry-run: show plan without executing
kafka-consumer-groups.sh --bootstrap-server "$BOOTSTRAP" \
  --group "$GROUP" --topic "$TOPIC" \
  --reset-offsets --to-earliest --dry-run

# Execute (example: reset all partitions in topic to earliest)
kafka-consumer-groups.sh --bootstrap-server "$BOOTSTRAP" \
  --group "$GROUP" --topic "$TOPIC" \
  --reset-offsets --to-earliest --execute
```

Other useful targets: `--to-latest`, `--to-datetime 2025-03-25T12:00:00.000`, `--shift-by -100`, `--to-offset <n>` (per partition in interactive/batch files depending on version).

**Warning:** `execute` causes **reprocessing** or **skipping**; ensure downstream idempotency and legal/audit constraints.

---

## 5. Broker & cluster configs

### Describe broker dynamic configs (where supported)

```bash
kafka-configs.sh --bootstrap-server "$BOOTSTRAP" --entity-type brokers \
  --entity-name <broker-id> --describe
```

### Cluster-wide sanity: replication and min ISR

Operators often verify (via static config or CM):

- **`default.replication.factor`** / per-topic RF vs rack awareness
- **`min.insync.replicas`** — with `acks=all`, producers fail if ISR shrinks below this
- **`unclean.leader.election.enable=false`** — avoid silent data loss on hard failures (tradeoff: availability)

Document your org’s standards; don’t alter in prod without a change ticket.

---

## 6. Runbooks

### 6.1 Growing consumer lag

1. **Confirm** lag is real: `kafka-consumer-groups.sh --describe --group "$GROUP"`.
2. **Rule out** stuck partitions: one partition lagging while others fine → hot key or poison message.
3. **Scale** consumers up to partition count (no effect past partition ceiling).
4. **Tune**: batch processing, DB writes, `max.poll.records`, faster instances.
5. **Temporary** mitigations: add partitions **only** with a plan for key distribution and **double** consumer deployments during catch-up.

### 6.2 Under-replicated partitions (URP)

Symptoms: JMX / metrics `UnderReplicatedPartitions` > 0; `kafka-topics --describe` shows ISR size < replication factor.

Typical causes: broker down, network, disk slow, one follower out of sync.

Actions:

1. Identify **which brokers** host leaders vs out-of-sync replicas (`--describe`).
2. Restore **failed broker** or **network**.
3. If disk is saturated: throttle producers, expand disk, rebalance leadership (careful), offload topics.

### 6.3 ISR shrinks / producer `NOT_ENOUGH_REPLICAS`

Often coupled with broker failure or replica lag. Fix broker health first. If chronic: **network**, **disk**, or **under-provisioned** followers; avoid lowering `min.insync.replicas` without understanding loss risk.

### 6.4 Disk full on broker

1. Expand volume or **add brokers** and **reassign partitions** (use `kafka-reassign-partitions` tool with a generated plan — test in staging).
2. Lower **retention** (`retention.ms` / `retention.bytes`) on **large** topics if policy allows.
3. Ensure **log segment** cleanup is running; find **rogue high-cardinality compacted** topics misused for raw events.

### 6.5 “Consumer keeps rebalancing”

Check application logs for:

- processing longer than **`max.poll.interval.ms`**
- GC pauses / CPU starvation
- broker timeouts

Mitigations: reduce per-poll work, increase `max.poll.interval.ms` within safe bounds, upgrade client, cooperative rebalance settings per client docs.

### 6.6 Leader election churn / controller flapping

Rare but severe. Escalate to cluster owners; capture **controller** broker logs, **metadata** request latency, and recent config changes. Often infra (ZK/KRaft metadata, network partitions) rather than app code.

---

## 7. ACLs (when enabled)

List (example — syntax depends on authorizer):

```bash
kafka-acls.sh --bootstrap-server "$BOOTSTRAP" --list
```

Grant patterns vary (`--operation Read`, `Write`, `Describe`, consumer group access). Use your platform’s wrapper if you have one (enterprise often wraps ACLs in RBAC).

---

## 8. Metrics worth dashboarding

| Metric | Why |
|--------|-----|
| **Consumer lag** (per group, per topic/partition) | Primary SLO for pipelines |
| **Under-replicated partitions** | Durability / catch-up health |
| **Offline partitions** | Data unavailable |
| **ISR shrink / expansion rate** | Early warning |
| **Request handler idle % / request queue** | Broker overload |
| **Network bytes in/out** | Hot spot detection |
| **Log flush latency / disk wait** | Disk saturation |
| **GC time** (JVM brokers) | Pause sensitivity |

Pair **broker** metrics with **OS** metrics: disk util %, iowait, network errors.

---

## 9. Partition reassignment (advanced)

Use when adding brokers or fixing uneven disk.

1. Generate reassignment JSON (partition → replica broker ids).
2. Run **`kafka-reassign-partitions.sh`** with `--verify` until complete.
3. **Throttle** reassignment (`--throttle`) to protect production latency.

Always rehearse in staging; wrong JSON can hurt availability.

---

## 10. Safe operational habits

- **Canary** consumer deployments; watch **rebalance** duration and **lag** during rollouts.
- **Change one knob at a time**; record before/after on lag and p99 latency.
- **Never** share PATs/passwords in Git remotes or shell history; use vaulted credentials for SASL.
- **Document** “who may reset offsets” — same tier as database migration ownership.

---

## 11. Local practice stack (optional)

For exercises, use the official Apache Kafka **KRaft** quickstart or a small **Docker Compose** from the Kafka docs / Confluent’s quickstart—enough to run the commands above against `localhost:9092`. Production differs (TLS, SASL, rack awareness); treat local as **command rehearsal**, not a config template.

---

*Last updated: 2025-03-25*

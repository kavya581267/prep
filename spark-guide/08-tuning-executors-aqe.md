# Tuning Executors & AQE — In Depth

## There is no universal “best Spark config”

Tuning is **iterative**:

1. Establish **baseline** runtime + **cost** **\$** on a **representative** day of data.  
2. Identify **dominant stage** in Spark UI (longest **wall** time / biggest **shuffle read**).  
3. Change **one** knob category at a time—**partition count**, **executor memory**, **AQE** flags, **broadcast** threshold.

**Staff:** Always tie tuning to **SLA** and **\$ / TB processed**; don’t tune for **p99** **cluster** utilization if business only cares about **job** done **by** **6am**.

## Executor sizing: mental model

Each **executor** is a **JVM** with **`spark.executor.memory`** and **`spark.executor.cores`** tasks slots.

**Tradeoffs:**

- **Fat executors** (many cores, huge heap): good for **wide** shuffles **if** GC manageable; **bad** if one **executor** loss loses **too many** tasks; **GC** pauses **hurt** many cores at once.  
- **Skinny executors:** better **isolation**, more **overhead** from **process** count; may not have enough **memory** per **heavy** join.

Vendor platforms often ship **presets**; **start** from their **recommendation** for your **instance** type and **shuffle-heavy** workload.

## `spark.sql.shuffle.partitions` deep dive

This sets default **number of shuffle partitions** for **many** operations (aggregations, joins using shuffle, etc.).

**Too low** → each task pulls **huge** shuffle blocks, **long** tasks, **risk** OOM on reducer, **under** utilized if tasks < cores.

**Too high** → tiny tasks, **scheduler** overhead, **small** shuffle files (**metadata** pain), **slower** in surprising ways.

**Tuning method (pragmatic):**

- Run job with **default**.  
- Look at **shuffle read** / **records** per task in **UI**.  
- Target **~?MB** shuffle per task (some teams use **128–512 MB** target—**not universal**).

**AQE** with **coalescing** can **adjust** partition counts **after** **shuffle statistics**—still need **reasonable** initial.

## Adaptive Query Execution (Spark 3)

Common AQE features (exact names and defaults vary slightly by distro):

- **Coalescing shuffle partitions** — reduce many **small** partitions **post-shuffle** when stats show **unnecessary** parallelism.  
- **Skew join optimization** — split skewed partitions on hot keys.  
- **Switching join strategies** — e.g., discover one side smaller than thought → **broadcast** at runtime.

**Requirement:** **shuffle statistics** must be computed—AQE benefits from **complete** **map** side before **reduce** **adapts**.

**Interview answer:** “I enable AQE in prod; I verify with **explain** history / UI that **skew** handling engaged; I don’t **disable** adaptivity unless I have a **provable** regression.”

## Dynamic allocation (`spark.dynamicAllocation.enabled`)

Executors **scale** with workload—good for **multi-tenant** clusters; requires **shuffle tracking** service and careful **min/max** bounds to avoid **thrash**.

## Worked example — tuning narrative

**Job:** **daily** fact build, baseline **45 min**, SLA **60 min**.

**UI:** Stage **sortMergeJoin** **30 min**; one task **22 min**.

**Diagnosis:** **skew** (classic).

**Actions:** confirm **key** distribution; enable **AQE skew join**; if still bad, **salt** **aggregation** path; re-run → **join** **10 min** \; total **25 min**.

## Soundbite

> “I tune **shuffle partitions** from **observed shuffle sizes**, enable **AQE**, and stop when **marginal** **\$** savings per minute gained falls below **engineering** **time**.”

Next: [09-structured-streaming-overview.md](./09-structured-streaming-overview.md)

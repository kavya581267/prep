# Catalyst & Tungsten — In Depth

## Catalyst in one narrative

**Catalyst** is Spark SQL’s **query optimizer**. You submit either SQL strings or DataFrame operations. Spark builds a **logical plan** (what you **meant**), applies **rewrites** (algebraic optimizations), and converts to a **physical plan** (how the **cluster** will execute).

Think of it as a **compiler** for data pipelines:

1. **Unresolved logical plan** (column names not yet bound)  
2. **Analyzed logical plan** (schema resolved against **catalog** / **datasets**)  
3. **Optimized logical plan** (predicate pushdown, projection pruning, constant folding, operator **reordering**)  
4. **Physical plan** (choose join algorithms, **exchange** (shuffle) placements, **codegen** boundaries)

**Interview:** “`explain(extended)` shows me where **filters** landed and whether I accidentally forced **extra** **shuffles**.”

## Concrete optimizations you should name

### Predicate pushdown

If your query has `WHERE dt = '2025-03-01' AND event_type = 'click'`, Spark tries to **push** those predicates **into** the **file scan** operator so **Parquet** can **skip** row groups and **dire**ctories.

**Without** pushdown (bad storage or unsupported source), Spark reads **everything** then filters in memory—**disaster** at scale.

### Projection pruning

`SELECT user_id, revenue FROM fact` causes scan to **decode** only those columns from Parquet.

**Demo trick:** compare **`explain`** with `select *` vs `select two columns`** on same table—**scan** output **schema** width changes.

### Constant folding

Expressions like `WHERE ts > '2025-01-01'` may be **evaluated** once at **planning** time instead of **per row**.

### Join reordering (limited but real)

In some cases Catalyst **reorders** joins based on **stats** and **selectivity**—larger topic at **PhD** depth, but **Staff** awareness is enough: “optimizer may reorder joins given **statistics**.”

## Tungsten / whole-stage codegen — intuition

**Tungsten** refers to Spark’s **execution engine** optimizations: **off-heap** memory management, **binary** processing formats, and **whole-stage code generation** that **fuses** narrow operations into **generated JVM bytecode** to reduce **virtual calls** (`Row.get` in tight loops).

**User takeaway:** staying in **DataFrame/SQL** allows **codegen**; **Python RDD** or **Python UDF** often **breaks** or **bypasses** these fast paths.

## Python UDF cost model (numbers are illustrative)

Suppose **1 billion** rows.

- **Built-in** column expression might run near **disk bandwidth** limited.  
- **Python row UDF** may drop to **low millions** rows/sec per executor due to **serialization** and **GIL** constraints.

**Mitigation:** **Pandas UDF** / **Arrow** vectorized batches move data in **columnar batches** across the boundary—much less per-row overhead.

**Staff:** “I’d require **vectorized** UDF or **Scala** for the **hottest** **transform**.”

## Statistics matter

**CBO (Cost-Based Optimization)** uses table **statistics** (row counts, column **NDV**—number of distinct values) to choose joins **order** and **strategies**.

If stats are **stale** or **missing**, Spark may ** underestimate** a table and **broadcast** something too big—or pick a **bad** join order.

**Practice:** Run **`ANALYZE TABLE`** / vendor equivalents on **managed** tables; for **raw** **file** sources, **know** you may lack stats unless maintained.

## Soundbite

> “Catalyst rewards **clean SQL**: early **filter**, minimal **columns**, **partition** friendly layout. Tungsten rewards staying off **Python per-row** UDFs. I validate with **`explain`** and **UI** **metrics**.”

Next: [07-memory-spill-serialization.md](./07-memory-spill-serialization.md)

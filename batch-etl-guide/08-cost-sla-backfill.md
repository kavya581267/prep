# Cost, SLA, Backfill — In Depth

## Cost drivers in batch (what actually burns money)

1. **Full table scans** when **partition** keys don’t match query filters.  
2. **Shuffle-heavy** joins replicated on **every** run without **pre-materialized** **marts**.  
3. **Over-sized** clusters left running **idle** between **bursts**.  
4. **Cross-region** **data** movement for **compliance** or bad **architecture**.  
5. **Small file** tax: **metadata** ops dominate **read** time.

**Staff:** Attribute **\$/TB processed** or **\$/1k DAU pipeline** to teams—**behavior** follows **metering**.

## SLAs that aren’t fantasies

Good SLAs specify:

- **Freshness:** “`fact_impressions_daily` **available** **≤ 3h** after **`dt`** **close** in **America/New_York**.”  
- **Completeness tolerance:** “**<0.05%** **row** delta vs **source** **extract** **counter**.”

**Buffer** internally tighter than external marketing promise.

## Backfill as an engineering project

**Trigger:** bug in **`join` logic** discovered **March 10** for **Jan 1** onwards.

### Plan

1. **Scope** **tables** affected + **downstream** marts to **rebuild**.  
2. **Estimate** **compute**: **180** days × **2 TB**/day scanned → **choose** **parallelism** and **spot** vs **on-demand**.  
3. **Freeze** **schema** changes during **backfill window** or **version** outputs.  
4. **Communicate** to **BI** users: **two** **versions** **possible** during overlap.

### Execution

**Partition** backfill by **`dt`** in **orchestrator** **dynamic** mapping (mapped tasks) rather than **one** giant job without checkpointing.

### Verification

**Row counts** vs **original** bad pipeline; **spot-check** **finance** **totals** on **known** campaigns.

## When to rebuild vs patch

**Patch** incremental events if **error** is **small** and **well-bounded**.

**Rebuild** when core **grain** wrong; **patching** accumulates **impossible-to-audit** **corrections**.

## Soundbite

> “Backfill is **planned** **surgery**: **cost** estimate, **partitioned** **recompute**, **comm**, **verification**—not an **ad-hoc** **`for` loop** in a notebook.”

Next: [09-interview-questions-by-level.md](./09-interview-questions-by-level.md)

# Targeting Controls: Frequency Cap, Daypart, Geofence

Advertisers don’t want “show my ad to **everyone always**.” They want **control**. These three concepts are among the most common **delivery rules** you’ll hear in Ad Tech and in system design prompts.

---

## 1. Targeting (big picture)

**Targeting** means **restricting who can receive an ad** (and sometimes **when** and **where**).

Inputs often include:

- **Geo** (country, state, city, radius)  
- **Device type** (phone vs connected TV)  
- **OS / app**  
- **Audience segment** (first-party or partner)  
- **Context** (sports section vs finance—**contextual** targeting)  
- **Time** (daypart—see below)

The **ad server** (or decisioning layer) **filters** candidates: if you don’t match targeting, you’re not eligible.

---

## 2. Frequency cap

**Definition:** A rule like “**show this campaign to the same user at most N times per time window**.”

**Examples:**

- “Max **3 impressions per user per day**”.  
- “Max **10 impressions per week** across all creatives in this **flight**.”

**Why it exists:**

- **User experience:** endless repeats annoy users.  
- **Diminishing returns:** the 50th impression rarely adds value.  
- **Brand guidelines:** some brands cap **frequency** tightly.

**Engineering challenges:**

- You need a **fast counter read** on the hot path (often **centralized store** like Redis, or **approximate** methods at insane scale).  
- **Identity** choice matters (device vs logged-in user vs household).  
- **Retries** must not inflate counts wrongly—tie events to **stable IDs** and **dedupe** keys.

**Tradeoff:** **Strict** caps protect users but can **under-deliver** campaigns if measurement is noisy; **loose** caps **waste** impressions.

---

## 3. Daypart

**Definition:** **Daypart** = **time-of-day** (and sometimes **day-of-week**) scheduling: “only bid/show between **5pm–11pm local**” or “weekends only.”

**Why it exists:**

- **Consumer behavior:** fast food breakfast vs late-night streaming.  
- **Retail:** store hours promotions.  
- **Regulatory / brand safety:** alcohol rules in some regions (conceptual).

**Engineering challenges:**

- **Time zones:** “local” matters. A user in **Chicago** vs **New York** must map to correct windows.  
- **DST** changes: handle **ambiguous hours** carefully in offline logs.

**Tradeoff:** Fine-grained dayparts increase **complexity** but improve **efficiency** for advertisers.

---

## 4. Geofence (geo targeting)

**Definition:** A **geographic boundary** for targeting: show ads only to people **inside** (or sometimes **outside**) a region.

**Shapes:**

- **Country / DMA / city** lists  
- **Radius** around a lat/long (“**1 mile around store**”)  
- **Polygon** (messy but used for odd regions)

**How location is known (conceptual):**

- **IP address** (coarse; VPNs break it).  
- **GPS** (mobile; permissioned).  
- **Registered** home zip from account profile.

**Privacy:** Precise location is **regulated** and sensitive; real systems use **auditing** and **consent**.

**Engineering challenges:**

- **Fast geo lookup** on decision path: **geohashing**, **pre-indexed** audiences, **edge** hints from the client.  
- **Accuracy** vs **latency** tradeoffs.

**Tradeoff:** **Tighter** geo improves **relevance** but **shrinks** audience (may **under-deliver**).

---

## 5. How these show up in “design ad serving” interviews

You should explicitly mention:

- **Where** frequency counters live (usually **not** in the primary SQL DB for peak path).  
- **Timezone** handling for daypart.  
- **Geofence** evaluation strategy (coarse first, refine if budget allows).

---

## 6. Mini glossary

| Term | One line |
|------|----------|
| **Targeting** | Filters who/where/when an ad may serve. |
| **Frequency cap** | Limit impressions per user/time for UX and efficiency. |
| **Daypart** | Schedule ads by time-of-day/week (local time). |
| **Geofence** | Geographic boundary for inclusion/exclusion. |

Next: [Ad server vs SSP vs DSP](./05-ad-server-SSP-DSP-programmatic.md).

# Ad Serving and A/B Testing — Basics (Beginner)

## 1. What problem are we solving?

Websites, mobile apps, and streaming apps often show **ads** (images, videos, sponsored listings). Someone has paid to show those ads to certain people, under certain rules. **Ad serving** is the **real-time software** that:

1. Receives a **request** (“I have a spot for an ad—what should I show?”).  
2. **Decides** which campaign and creative fit the rules.  
3. Returns **instructions** so the app can display the ad and **track** what happened.

**A/B testing** (or **experimentation**) is when you **compare two or more variants** of how that decision (or the creative) works—**without** manually guessing which is better. You **measure** outcomes (clicks, conversions, revenue, video completes, etc.) in a disciplined way.

---

## 2. The main characters (very simplified)

| Role | Plain English |
|------|----------------|
| **Publisher** | The app/site/stream that has **space** to show ads (e.g., a news site, a streaming service). |
| **Advertiser / brand** | The company paying to show ads for a product. |
| **Agency** | Often plans and buys campaigns on behalf of advertisers. |
| **Ad server** (concept) | The system that **stores** what should run and **chooses** at request time (details vary by company). |

You don’t need to memorize org charts—just that **someone owns inventory**, **someone owns campaigns**, and **software connects them under rules**.

---

## 3. The ad request in one story

Imagine you open an article on a news app. Before an ad appears:

1. The app says: “**Placement** `homepage-banner` is free. Here is **context** (country, device type, maybe logged-in user ID).”  
2. The **ad stack** looks at **active campaigns**: budget left, targeting, start/end dates.  
3. It picks a **creative** (the actual image/video + link).  
4. It returns a **response**: what to render + **tracking URLs** (“call this when the ad is actually visible”).  
5. Later, your device may report an **impression** (“we showed it”) or a **click**.

That round trip is the heart of **ad serving**. Engineers care because it must be **fast**, **reliable**, and **auditable** (money and legal risk).

---

## 4. What is a “placement”?

A **placement** (or **ad slot**, **inventory unit**) is a **named place** where an ad can go: a banner position, a pre-roll before a video, a pause-ad on CTV, etc. Each placement has an ID so the system knows **which rules and demand** apply.

---

## 5. What is a “campaign”?

A **campaign** is a **package of business intent**, typically including:

- **Advertiser**  
- **Budget** (how much money)  
- **Schedule** (start/end)  
- **Targeting** (who is eligible)  
- **Creatives** (what can be shown)  
- **Delivery rules** (how fast to spend; how often the same person can see the ad—see the targeting file)

---

## 6. What is A/B testing in ad serving?

**A/B testing** compares **Variant A vs Variant B** (sometimes more). Examples:

- **Ranking logic**: “Should we prefer **higher CPM** ads or **better predicted click rate**?”  
- **Creative**: two different video cuts for the same product.  
- **Frequency rules**: show at most 3 ads/week vs 5 ads/week.

### Why not just ship the “obvious” winner?

Because **context changes** (audience, seasonality, content), and **local maxima** fool humans. Experiments give **evidence**.

### What must be “stable”?

If the same **person** (or **device**) sometimes sees A and sometimes B on every refresh, your metrics can be **noise**. Good systems assign a user to a **bucket** **deterministically** (for example using a hash of user ID + experiment ID) so they **stay** in one variant unless you intentionally re-randomize.

### What do you log for experiments?

Usually:

- **Exposure** (“this user was **eligible** for this variant” or “this **decision** used this ranking version”).  
- **Outcomes** (click, conversion, completion).  

Analysis compares outcomes **between cohorts**. This connects to **metrics** in the next file.

### Guardrails

Big publishers won’t run an experiment that **crashes revenue** or **latency** without automatic rollback. Those safety checks are called **guardrail metrics**.

---

## 7. How this connects to “system design”

When interviewers say “design ad serving with A/B testing,” they want:

- **Fast path**: decide in tens of milliseconds in many designs.  
- **Correct accounting**: don’t **overspend** budget; don’t **double count** impressions due to retries.  
- **Logging**: high-throughput **events** for billing and analytics.  
- **Experiment integrity**: fair assignment + interpretable metrics.

You’ll add **Kafka**, **caches**, **databases**, and **APIs** on top of this mental model.

---

## 8. Quick glossary (this file only)

| Term | One line |
|------|----------|
| **Ad serving** | Real-time selection and delivery of ads into placements. |
| **Placement** | A specific ad “slot” on an app/site. |
| **Campaign** | Budget + schedule + targeting + creatives. |
| **Creative** | The actual ad asset(s) users see. |
| **Impression / click** | See metrics file—counting views and clicks. |
| **A/B test** | Controlled comparison of variants with measurement. |
| **Variant** | One version of logic or creative in an experiment. |
| **Guardrails** | Metrics you protect (revenue, latency, errors) while testing. |

Next: [Metrics & money (CPM, CPC, …)](./02-metrics-impressions-clicks-CPM-CPC-CPA.md).

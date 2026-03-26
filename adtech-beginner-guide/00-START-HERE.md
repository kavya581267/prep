# Ad Tech for Beginners — Start Here

This folder explains **how digital advertising delivery works**, what **A/B testing** means in that world, and the **vocabulary** you need for interviews (including the Warner Bros. Discovery–style Staff Engineer role).

**Read in order** the first time through (the files build on each other). After that, use any file as a **reference**.

## Who this is for

You can treat this as **Ad Tech 101**. No prior industry knowledge is assumed. If a word appears in bold the first time, it is usually a term defined in the same or an earlier file.

## Reading order

| # | File | What you’ll learn |
|---|------|-------------------|
| 1 | [How ads get chosen and shown (Ad serving + A/B basics)](./01-ad-serving-and-ab-testing-basics.md) | The “request → decision → response” path; what experiments change |
| 2 | [Metrics & money: impressions, clicks, CPM, CPC, …](./02-metrics-impressions-clicks-CPM-CPC-CPA.md) | How success is counted and paid for |
| 3 | [Data: first-party vs third-party, device vs household](./03-data-first-party-third-party-household-device.md) | Where audience information comes from |
| 4 | [Targeting controls: frequency cap, daypart, geofence](./04-targeting-frequency-daypart-geofence.md) | Rules advertisers use to limit who sees what |
| 5 | [Ad server vs SSP vs DSP (programmatic)](./05-ad-server-SSP-DSP-programmatic.md) | Who does what when ads are bought automatically |
| 6 | [Brand safety & ad verification](./06-brand-safety-ad-verification.md) | Keeping ads away from bad content and proving delivery |
| 7 | [Forecasting, pacing, yield](./07-forecasting-pacing-yield.md) | Planning inventory, spending smoothly, maximizing value |
| 8 | [Linear TV terms: avail, pod, makegood, logs](./08-linear-tv-terms-avail-pod-makegood.md) | Traditional TV + how it shows up in jobs like WBD’s |
| 9 | [Tradeoffs: reach vs frequency, privacy, precision](./09-tradeoffs-reach-frequency-privacy-precision.md) | Business and technical tensions (interview-style) |
| 10 | [Programmatic flow: app → identity → bid → auction → serve → feedback](./10-programmatic-request-to-feedback-flow.md) | End-to-end real path with examples (CTV, mobile, web) |

## How this ties to system design interviews

Interviewers often ask you to design a **mini ad server** or **experimentation layer**. After these readings you should be able to say, in plain English:

- What must happen in **milliseconds** on each page load or ad break.  
- What gets **logged** and why **deduplication** matters.  
- Why **experiment assignment** must be **stable** for the same user.  

Then layer on **engineering** (caches, Kafka, APIs, Staff tradeoffs) from [`../staff-deep-dive-guide/03-system-design-ad-serving-experiments.md`](../staff-deep-dive-guide/03-system-design-ad-serving-experiments.md) and the rest of [`../staff-deep-dive-guide/00-START-HERE.md`](../staff-deep-dive-guide/00-START-HERE.md).

## One-sentence picture

**Someone asks “which ad should we show?” — the ad stack answers fast, logs what happened, spends budget fairly, and sometimes runs an A/B test on how the answer is computed.**

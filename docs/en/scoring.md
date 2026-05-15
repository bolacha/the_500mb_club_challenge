# How scoring works

This document explains, in detail, how each submission is scored when the final test runs — when the actual benchmark runs against the stack on the Raspberry Pi.

## TL;DR

- The score is **relative to an absolute target profile** (latency SLOs + a 2 CPU / 500 MB
  budget), **not** to any specific implementation. **`100` = you meet the target**; above
  100 = you exceed it.
- The **global score** (weighted mean of 5 dimensions × 100) is the **sole decider** of
  the ranking. Beyond it, each dimension awards a **medal** 🥇 to the leader — so that
  different languages can shine on different axes.
- Before scoring, there is a **gate**: sustain the base load without errors and fit within
  the budget. Fail the gate → fall out of the podium (but not silently disqualified).

## Why absolute targets (instead of "race against an implementation")

An alternative would be to normalize everything against a reference implementation
(`100 = on par with it`). The side effect: native implementations (compiled, no GC) excel
on low-level metrics, so any managed runtime would start behind simply **for not being that
reference** — a p99 latency of 67 ms (perfectly adequate for telemetry ingestion) would hit
the floor just for being "slower than 2 ms".

For this reason we anchor each metric to an **absolute target**: a generous latency SLO,
the memory/CPU budget, a reference RPS. Whoever meets the SLOs and fits within the budget
scores well (≥ 100), regardless of the language; **efficiency** and **capacity** — the
heart of the "500 MB Club" — are what separate the top.

## Load scenarios

The benchmark runs four k6 scenarios against the stack (the generator runs **outside** the Pi).
Each dimension is fed by a specific scenario:

| Scenario | Load | Feeds |
|---|---|---|
| **steady** | sustained load with realistic operation mix | efficiency, p99 latency + the **gate** |
| **capacity** | progressive ramp up to the limit | capacity (max sustained RPS) |
| **spike** | sudden traffic spike | resilience |
| **endurance** | prolonged load | stability (drift) |
| _footprint_ | measured outside k6 scenarios | _informational_ (not scored) |

The `smoke` and `test` scenarios run **first** as a correctness check (API contract):
if `smoke` fails, nothing else runs. They do **not** factor into the score.

## The gate (pre-condition)

Before the weighted mean, the submission must, in `steady`:

- **Sustain the offered load** (~200 RPS) with `http_req_failed` < **0.5%**.
- **Fit within the runtime budget**: aggregate p95 RSS < **500 MB** and CPU < **200%** (2 cores).

Whoever fails receives the **`gated`** flag: falls out of the podium on the leaderboard, but
the score is still calculated and shown — no opaque disqualification.

## The 5 dimensions

Each dimension is the **mean** of the (clipped) ratios of its metrics. For a metric
with target `T` and observed value `V`:

- "**higher is better**" (`up`): ratio = `V / T`
- "**lower is better**" (`down`): ratio = `T / V`

`ratio = 1.0` means exactly at the target; `> 1` exceeds it, `< 1` falls short. The ratio
is clamped (_clipped_) to the dimension's interval before entering the mean.

A metric may use a **band** instead of a ratio (today only efficiency's RSS):
a **floor** and a **ceiling** are defined — at the floor (or below) it scores the clip top,
at the ceiling (or above) it scores the clip bottom, and the middle is **linear**. This is
useful when it makes more sense to anchor on "below X is great, above Y is bad" rather than
a single target.

| Dimension | Weight | Metric(s) → target | Clip | Role |
|---|---|---|---|---|
| **efficiency** | **0.32** | `rss_p95` band 50–500 MB · `cpu_avg` → 40% (in steady) | 0.25–4.0 | the theme; wide clip = high resolution |
| **capacity** | **0.27** | `max_sustained_rps` → 1000 RPS | 0.25–4.0 | work within the budget (headline) |
| **tail_latency** | **0.20** | p99 post/batch/range/anomaly → 8/25/15/25 ms | 0.25–1.5 | latency under load |
| **resilience** | **0.13** | `spike_p99` → 12 ms · `spike_error` → 1% | 0.25–2.0 | handles the spike |
| **stability** | **0.08** | `latency_drift` → 1.10 · `rss_drift` → 1.10 | 0.25–1.5 | no leaking/degrading |

_Footprint (image size, cold start) is no longer scored — see below; it continues as an **informational** column on the leaderboard._

Why these clips: **efficiency** and **capacity** (wide clip up to 4.0) carry the real
separation — using 1/4 of the budget scores ~3×. The others have a lower ceiling (1.5–2.0):
their targets are **tight** ("excellent", not just "adequate") to **separate the tiers**,
but the ceiling prevents irrelevant differences within a technically tied cohort (2 ms vs
5 ms of p99) from dominating the mean. **Efficiency** anchors on the real budget: aggregate
RSS in a **50–500 MB band** (≤50 = top clip, ≥500 = bottom, covering the entire allowed
range) and CPU against half the ceiling (40%) — both with a wide clip up to 4.0 to preserve
resolution at the top.

### efficiency (32%)

Aggregate `p95 RSS` and `mean CPU` (sum across all containers: APIs + LB + storage),
measured in `steady@200`. This is where frugality shows — the two metrics have different
shapes:

- **RSS** uses a **[50 MB, 500 MB] band**: the entire stack at ≤50 MB hits the clip top
  (4.0); ≥500 MB (the challenge's own ceiling) hits the clip bottom (0.25); linear in
  between. The band covers the entire allowed range, so no one falls off the ends. Ex.:
  ~90 MB (Rust) → ~3.7; ~128 MB (Go) → ~3.3; ~260 MB → ~2.2; ~370 MB → ~1.3.
- **Mean CPU** uses a ratio against **40%** (half the ceiling of 200% = 2 cores): using 20% → 2.0.

### capacity (27%)

The **maximum sustained RPS** within the budget — the "knee" of the load curve. Measures
how much real work you deliver with 2 CPU / 500 MB. Calculation detail below. The reference
RPS is the **field median** measured on the Pi — recalibrated each round as new submissions
enter.

### tail_latency (20%)

p99 of the four operations in `steady`, against **"excellent"** targets (8/25/15/25 ms —
the real field is at 1–7 ms). The fast cohort (native, Go, Elixir, Node) still saturates
the clip at 1.5 — separating 1 ms from 5 ms would be noise —, but the slower ones fall and
the dimension **separates the tiers**: C# (~33 ms) and Python (~67 ms) fall below 1.0, and
C++ loses for having `batch`/`anomaly` ~50 ms despite the fast `post`.

### resilience (13%)

How the service behaves **during** the spike: **p99 under spike** (target 12 ms) and the
**error rate during the spike** (target 1%). The error rate was 0% for all submissions
measured so far — so today this metric only "fills" the dimension (everyone at the ceiling
for it), but it stays in the calculation because it may discriminate future submissions that
fail under load.

### footprint — informational, **outside the score**

**Footprint is no longer a scored dimension.** Image size (`image_mb`) and cold start
(`cold_start_s`) continue to be **measured and displayed on the leaderboard** (informational
columns, in italics), but **do not enter the score** or award medals — the calculation was
not aggregating enough signal to justify a weight (the lean ones tied among themselves, the
fat ones too; and cold start is ~19–20 s for all, dominated by stack overhead). For how
they are measured, see "How footprint is measured" below. They remain as a frugality
reference.

### stability (8%)

Drift over `endurance`: p99 of the last 5 min ÷ first 5 min, and final RSS ÷ initial. It is
a **guarantee** ("no memory leak, no degradation"), not a differentiator — hence the low
weight.

## The global score formula

```
for each dimension d present:
    dim[d] = mean( clip(ratio of each metric, clip_min[d], clip_max[d]) )

score = 100 × Σ ( weight[d] × dim[d] )  ÷  Σ weight[d]      (only over present dimensions)
```

The renormalization (`÷ Σ weight[d]` of present only) is what prevents an **absent**
dimension from bringing the score down — see "missing metric" below.

### Global score floor and ceiling

With asymmetric clips per dimension (efficiency/capacity go up to 4.0; tail_latency/stability up to 1.5; resilience up to 2.0) and the v7 weights, the score's possible range is:

- **Floor = 25** — all 5 dimensions saturating at the minimum (`0.25`):

  ```text
  100 × 0.25 × (0.32 + 0.27 + 0.20 + 0.13 + 0.08) = 25
  ```

- **Ceiling = 304** — each dimension saturating at its own maximum:

  ```text
  100 × (0.32·4.0 + 0.27·4.0 + 0.20·1.5 + 0.13·2.0 + 0.08·1.5)
      = 100 × (1.28 + 1.08 + 0.30 + 0.26 + 0.12)
      = 100 × 3.04 = 304
  ```

Per-dimension contribution detail:

| Dimension | Clip | Weight | Ceiling contrib. | Floor contrib. |
| --- | --- | --- | --- | --- |
| efficiency | [0.25 – 4.0] | 0.32 | 128 | 8 |
| capacity | [0.25 – 4.0] | 0.27 | 108 | 6.75 |
| tail_latency | [0.25 – 1.5] | 0.20 | 30 | 5 |
| resilience | [0.25 – 2.0] | 0.13 | 26 | 3.25 |
| stability | [0.25 – 1.5] | 0.08 | 12 | 2 |
| **total** | — | **1.00** | **304** | **25** |

Caveat: when a dimension is **excluded** due to a harness gap, weights renormalize among the present ones — the effective ceiling/floor of that run changes with the mix. The `100 = meets the target` still holds as the equilibrium point (ratio `1.0` on every metric), not as the ceiling.

## How the capacity "knee" is measured

The capacity scenario ramps the load in **sustained steps** (plateau of ~45 s +
ramp of ~10 s), from 200 to 5000 RPS **in 100 RPS increments** — a fine grid to resolve
the knee with 100 RPS precision. The knee is **not** a k6 threshold — it is calculated
from the per-request time series (one event per request):

1. Each request is assigned to its step by elapsed time since test start.
2. On the **plateau** of each step (discarding the first ~10 s of settling) the following
   are measured: p99 latency, error rate, effectively delivered RPS, and whether
   `dropped_iterations` occurred (signal that the service cannot keep up with the offered
   rate).
3. A step **counts as sustained** if: `p99 < 150 ms` **AND** `error < 0.5%` **AND**
   `delivered ≥ 95% of offered` **AND** no `dropped_iterations`.
4. **`max_sustained_rps` = the largest contiguous step** (from the first) that was sustained.

Why the criterion is by **SLO**, not by crash: at 800 RPS, in real data, *no* language
produced errors — but Python's p99 was already at 230 ms (vs 10–67 ms for the others). A
service "up, but at 230 ms" has already broken for the use case (the freezing map). The
SLO-based knee captures that; the crash-based knee does not.

## How footprint is measured

- **`image_mb`** — sum of the **compressed** layers from the **arm64** manifest of the API
  image (your artifact; official redis/postgres/nginx do not count). Fallback to the local
  uncompressed size if the manifest is not readable.
- **`cold_start_s`** — time between `docker compose up` and `/readyz` responding `200`
  stably (3× in a row). Images are pulled **before** and do not count. This value is
  **collected as informational** and appears in each submission's profile, but does not
  enter the score (see the _footprint_ section above).

## Missing metric policy

- **Scenario ran and failed** (crash) → the affected dimension receives the floor (`0.25`).
- **The harness did not collect** the metric (e.g. the capacity scenario has not run yet) →
  the dimension is **excluded and weights renormalize** among the present ones. You are
  never penalized for a harness gap — only for what was actually measured.

## Recognition

- **Global score** = sole decider of the ranking and the winner.
- **Dimension medals** 🥇 go to the **sole leader** of each of the **5 dimensions**
  (efficiency, capacity, p99 latency, resilience, stability). Axes where everyone saturates
  at "excellent" (latency, sometimes resilience) award no medal — it goes to the real
  differentiators. Footprint does **not** award a medal (it is informational).
- **Sponsor prizes** (JetBrains license + 3 Ardan Mastery Bundle vouchers + GopherCon
  Latam tickets) and how they are split between the top-4 of the global ranking and the
  `efficiency`/`capacity` leaders: see [`prizes.md`](./prizes.md).

## Where to see the details

Each submission receives a metric-by-metric breakdown — the ratio of each metric against the
target, the five dimensions, the gate flags, and the final score —, and the **leaderboard**
shows the ranking by global score with medals. All weights, targets, and clips used in the
calculation are documented in this file.

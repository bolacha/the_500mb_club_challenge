# Efficiency — weight 32%

**Efficiency** measures how much of the hardware budget (RAM and CPU) the whole stack
consumes to sustain the baseline load. It is **the theme of the challenge**: in the
"500 MB Club", whoever delivers the service using _less_ resource wins.

## What it measures

Two resources, measured in the **`steady`** scenario (fixed rate of 200 RPS, realistic
mix), **aggregated** — i.e., summing _all_ containers per point in time: the 3 API
replicas + the load balancer + the storage (redis/postgres/…). It is not "per instance";
it's the cost of the complete solution.

- **RSS** — resident memory (the p95 of the aggregated time series).
- **Average CPU** — aggregated CPU usage (the mean of the series).

## How it is computed

The two metrics use **different shapes**:

- **RSS** uses a **band [50 MB, 500 MB]** (not a plain ratio): the whole stack at
  **≤ 50 MB** scores the top of the clip (4.0); at **≥ 500 MB** (the challenge's own
  ceiling) it scores the bottom (0.25); between them, it's **linear**. The band covers the
  entire allowed range, so nobody overflows the ends.

  ```
  score_rss = 0.25 + 3.75 × (500 − RSS) / (500 − 50)      (clamped to 0.25–4.0)
  ```

- **Average CPU** uses a **ratio** against the par of **40%** (half of the 200% ceiling =
  2 cores): `40 / CPU`. Using 20% → 2.0; using 40% → 1.0.

The dimension is the **mean** of the two, clamped to **0.25–4.0**. The wide clip (up to
4.0) is intentional: efficiency is a **differentiator**, so it rewards real frugality with
high resolution instead of saturating early.

## Why it matters

The challenge caps the stack at **2 CPUs and 500 MB aggregate**. Efficiency is the
headroom left over: whoever runs in 1/4 of the budget can scale further, share the edge
device with other services, or simply cost less. Anchoring RSS to a band up to the **real
ceiling (500 MB)**, rather than an arbitrary target, lets the number speak the language of
the budget.

## In the challenge's context

Real examples measured on the Pi (aggregated RSS and aggregated CPU in `steady@200`):

| Submission | RSS p95 | → score | Avg CPU | → score | **efficiency** |
|---|---|---|---|---|---|
| `zig` | 137 MB | 3.28 | 10.3% | 3.88 | **3.59** |
| `rust` | 91 MB | 3.66 | 17.1% | 2.34 | **3.00** |
| `go` | 128 MB | 3.35 | 20.4% | 1.96 | **2.66** |
| `nodejs` | 261 MB | 2.24 | 30.7% | 1.30 | **1.77** |
| `python` | 284 MB | 2.05 | 73.7% | 0.54 | **1.30** |

Efficiency, together with [capacity](capacity.md), carries the separation at the top of
the ranking — they are the two wide-clip dimensions. Details of the global calculation in
[scoring.md](./scoring.md).

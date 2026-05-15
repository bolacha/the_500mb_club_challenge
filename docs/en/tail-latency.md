# Tail latency — weight 20%

**Tail latency** is the latency of the **slowest** requests — the "worst case" of the
distribution, not the average. It's what the user feels when the map freezes on screen.

## The idea

When a service answers thousands of requests, each takes a different amount of time.
Sorting them from fastest to slowest, the **tail** is the right end — the few that took
far longer than the rest. It's measured with **percentiles**:

- **p50** (median) — half the requests were faster than this.
- **p99** — 99% were faster; only **1% (the tail)** was slower.

Example over 1000 requests:

| percentile | latency | meaning |
|---|---|---|
| p50 | 2 ms | the typical case |
| p99 | 50 ms | the 10 slowest requests |
| p99.9 | 300 ms | the single slowest of all |

The **average** here would be ~4 ms and would completely hide the 50–300 ms tail.

## Why it matters (and why the average misleads)

1. **Users hit the tail more than you'd think.** If loading a screen makes 20 backend
   calls, it only finishes when the **slowest of the 20** responds. The chance at least one
   lands in the p99 is `1 − 0.99²⁰ ≈ 18%` — ~1 in 5 page loads feels the p99.
2. **The tail reveals pathologies the average hides:** _stop-the-world_ GC pauses, lock
   contention, memory swapping. That's why the challenge uses **strict round-robin** at the
   load balancer — it exposes pathological pauses instead of routing around the slow
   instance.

## How it is computed

In the **`steady`** scenario (200 RPS, realistic mix), we measure the **p99 of each of the
4 operations**, against **"excellent"** targets (the real field sits at 1–7 ms):

| operation | p99 target |
|---|---|
| `POST /telemetry` | 8 ms |
| `POST /telemetry/batch` | 25 ms |
| `GET /telemetry` (range) | 15 ms |
| `GET /anomaly` | 25 ms |

Each becomes the ratio `target / observed`, clamped to **0.25–1.5**, and the dimension is
the **mean** of the four. The low ceiling (1.5) is intentional: the fast pack (1–7 ms)
saturates — separating 1 ms from 5 ms would be noise — but the slow ones drop, and the
dimension **separates the tiers**.

## In the challenge's context

- The fast pack (native langs, Go, Elixir, Node) sits at ~1–7 ms → saturates at **1.50**.
- **C#** (~33 ms) and **Python** (~67 ms) fall below 1.0 → drop to ~0.53 and ~0.30.
- **C++** loses here despite a fast `POST` (1.4 ms): its `batch`/`anomaly` sit at ~50 ms,
  pulling the mean down to **1.00**. This shows why measuring all 4 operations matters — a
  blind spot in one route shows up.

The goal is "meet the SLO comfortably", not irrelevant micro-latency. Details of the
global calculation in [scoring.md](./scoring.md).

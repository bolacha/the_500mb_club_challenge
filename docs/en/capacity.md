# Capacity — weight 27%

**Capacity** is the **maximum sustained RPS** within the budget — the "knee" of the load
curve. It measures how much real work the service delivers with 2 CPU / 500 MB before it
breaks down.

## What it measures

The **`capacity`** scenario ramps the load in **sustained steps of 100 RPS**, from 200 up
to 5000 RPS. Each step has a short ramp (~10 s) + a plateau (~45 s). The goal is not "how
far can it go", but **how far can it go _well_** — without violating the SLO.

## How it is computed

The knee is **not** a threshold in the load generator; it's computed from the time series,
request by request. On each step's **plateau** (discarding the first ~10 s of settling)
we measure: p99 latency, error rate, RPS actually delivered, and whether there were
_dropped iterations_ (a sign the service can't keep up with the offered rate).

A step **counts as sustained** if, all at once:

- `p99 < 150 ms` **AND**
- `error < 0.5%` **AND**
- `delivered ≥ 95% of offered` **AND**
- **no** dropped iterations.

The **`max_sustained_rps`** is the **largest contiguous step** (starting from the first)
that held. The score is the ratio against a **reference par of 1000 RPS** (the median of
the field measured on the Pi), clamped to **0.25–4.0**:

```
capacity = max_sustained_rps / 1000      (clamped to 0.25–4.0)
```

## Why the criterion is SLO-based, not "crash"-based

In the real data, at 800 RPS _no_ language produced errors — but Python's p99 was already
~230 ms (vs 10–67 ms for the others). A service that is "up, but at 230 ms" has **already
broken** for the use case (the map freezing on the customer's screen). The SLO-based knee
captures this; a crash-based knee would not.

## In the challenge's context

Real knees measured on the Pi (100-RPS grid):

| Submission | Knee | → capacity |
|---|---|---|
| `zig`, `cpp` | 1200 RPS | **1.20** |
| `rust`, `go` | 1100 RPS | **1.10** |
| `elixir` | 1000 RPS | **1.00** |
| `nodejs`, `java`, `csharp` | 900 RPS | **0.90** |
| `python` | 200 RPS | **0.25** (floor) |

The fine 100-RPS grid is what lets us separate the top group (1100 vs 1200) instead of
tying them. Together with [efficiency](efficiency.md), it's the headline of the score.
Details of the global calculation in [scoring.md](./scoring.md).

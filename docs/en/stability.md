# Stability — weight 8%

**Stability** measures whether the service **degrades over time** under sustained load —
did it leak memory? did latency keep getting worse? It is a **guarantee**, not a
differentiator (hence the low weight): the goal is to confirm "it didn't get worse", not
to reward whoever improves.

## What it measures

The **`endurance`** scenario runs sustained load for ~45 minutes. It compares the start
with the end via two **drifts** (end/start ratios):

- **`latency_drift`** — p99 latency of the **last 5 min ÷ first 5 min**.
- **`rss_drift`** — **final ÷ initial** RSS (discarding warm-up).

A drift of `1.0` = identical from start to finish; `> 1.0` = got worse (latency rose, or
memory grew — possible leak); `< 1.0` = it even improved.

## How it is computed

Each drift becomes the ratio `target / observed` with a target of **1.10** (we tolerate up
to 10% worsening), and the dimension is the **mean** of the two, clamped to **0.25–1.5**:

```
stability = mean( 1.10 / latency_drift , 1.10 / rss_drift )   (clamped to 0.25–1.5)
```

The low ceiling (1.5) reflects its role as a guarantee: staying stable gives full marks;
there's no extra prize for "improving" 30% over the test (that's usually noise or warm-up).

## Why it matters

A submission may have great latency and capacity in the first minutes and still **leak
memory** or accumulate GC pauses until it busts the budget hours later. On edge hardware,
which runs for weeks without a restart, that's fatal. Stability is the check that what was
measured in `steady` **holds up**.

## In the challenge's context

Real drifts measured on the Pi (examples):

| Submission | `latency_drift` | `rss_drift` | **stability** |
|---|---|---|---|
| `nodejs` | — | — | **1.23** (leader) |
| `go` | 0.88 (improved) | 1.03 | **1.16** |
| `zig` | — | — | **1.05** |

All measured submissions stayed stable (scores ~1.0–1.2): nobody leaked or degraded in any
meaningful way — exactly the expected outcome of a guarantee. Details of the global
calculation in [scoring.md](./scoring.md).

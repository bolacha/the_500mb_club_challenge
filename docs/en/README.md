# Score dimensions — detailed guide

The Pi-Bench global score is the **weighted mean of 5 dimensions × 100**, anchored to
**absolute targets** (latency SLOs + a 2 CPU / 500 MB budget), not to any specific
implementation. `100` = you meet the target profile; above that = you exceed it.

Each document below explains **what** the dimension measures, **how** it is computed
(scenario, formula, target, clip, and weight), and **why** it matters — with real numbers
measured on the Raspberry Pi.

| Dimension | Weight | What it measures |
|---|---|---|
| [Efficiency](efficiency.md) | **32%** | RAM + CPU of the whole stack under baseline load |
| [Capacity](capacity.md) | **27%** | max sustained RPS within the budget |
| [Tail latency](tail-latency.md) | **20%** | p99 of the operations in `steady` |
| [Resilience](resilience.md) | **13%** | behavior during the spike |
| [Stability](stability.md) | **8%** | drift (memory/latency) over time |
| Footprint | _informational_ | image size + cold start (not scored) |

> **Efficiency + capacity = 59%** of the score: that's the heart of the "500 MB Club" —
> how much real work you deliver per unit of budget.

The overview of the calculation (gate, formula, missing-metric policy, medals) is in
[scoring.md](./scoring.md).

Want to re-run the benchmark on the Raspberry Pi after merge? See [testing.md](./testing.md).

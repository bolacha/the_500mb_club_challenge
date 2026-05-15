# Resilience — weight 13%

**Resilience** measures how the service behaves **during a load spike** — when traffic
suddenly jumps well above normal. Does it take the hit without the tail exploding or
errors creeping in?

## What it measures

The **`spike`** scenario ramps **50 → 800 RPS**, holds the peak, and backs off. During the
peak, we measure:

- **`spike_p99`** — the p99 latency **under peak** (a robust signal: it catches the tail at
  the most stressful moment).
- **`spike_error`** — the error rate **under peak**.

## How it is computed

Each metric becomes a ratio `target / observed`, and the dimension is the **mean** of the
two, clamped to **0.25–2.0**:

| metric | target |
|---|---|
| `spike_p99` | 12 ms |
| `spike_error` | 1% |

Why **p99 under peak** instead of "recovery time": recovery time was misleading — the
JVM's warm-up, for instance, could make a language look like it "recovered in 0 s" by
accident, and penalized fast languages for measurement artifacts. The p99 during the peak
is direct and robust.

## A note on `spike_error`

In every submission measured so far, the **error rate under peak was 0%** — nobody dropped
a request at the 800 RPS peak. With the ratio `0.01 / 0`, this metric lands at the top of
the clip (2.0) for everyone, so today it merely "fills" the dimension. It stays in the
calculation **on purpose**: it's a safety net for future submissions that fail under load
(at which point it starts to discriminate for real).

## In the challenge's context

p99 under peak measured on the Pi, and the resulting score (mean with `spike_error` = 2.0):

| Submission | `spike_p99` | → ratio | **resilience** |
|---|---|---|---|
| `cpp` | 6.0 ms | 2.00 | **2.00** |
| `zig` | 6.3 ms | 1.92 | **1.96** |
| `go` | 10.7 ms | 1.12 | **1.56** |
| `nodejs` | 47.5 ms | 0.25 (floor) | **1.13** |
| `python` | 170.5 ms | 0.25 (floor) | **1.12** |

The spike separates those who keep the tail under control under stress (native langs
~6 ms) from those who watch it blow up (Python ~170 ms). Details of the global calculation
in [scoring.md](./scoring.md).

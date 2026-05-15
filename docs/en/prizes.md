# Prizes

Our sponsors make this challenge possible. This document lists the prizes and explains,
unambiguously, **how winners are determined** and **how the prizes are distributed**.
The ranking criterion is the one in [`scoring.md`](./scoring.md).

## Sponsors

- **Ardan Labs** — 3 vouchers for the **Mastery Bundle** (Go, Rust, Docker & K8s) — online, 214h.
- **JetBrains** — one **All Products Pack + AI** license, valid for **1 year**.
- **GopherCon Latam 2026** — **3 tickets** to the event (**September 2–4, 2026**).

## The prizes

| Prize | Winner |
|---|---|
| JetBrains license (1 year) **+ 1 GopherCon ticket** | 🏆 **1st place** in the _global score_ |
| 1 Ardan **Mastery Bundle** voucher | 🥈 **2nd place** in the _global score_ |
| 1 Ardan **Mastery Bundle** voucher | 🥉 **3rd place** in the _global score_ |
| 1 Ardan **Mastery Bundle** voucher | **4th place** in the _global score_ |
| 1 GopherCon ticket | 🥇 **Leader of the `efficiency` dimension** |
| 1 GopherCon ticket | 🥇 **Leader of the `capacity` dimension** |

## How winners are determined

- **Cutoff date** = **July 26, 2026, 22:00 UTC**. The leaderboard snapshot at that moment is what determines the winners. Submissions (PRs) opened after that point are not eligible for prizes.
- **Top-4 of the global ranking** = the 4 submissions with the highest _global score_ —
  the sole decider of the ranking. See the formula and the gate in [`scoring.md`](./scoring.md).
- **Dimension leaders** = whoever has the **highest score** in `efficiency` and in
  `capacity` — exactly the 🥇 those dimensions award on the leaderboard.
- **Why `efficiency` + `capacity`?** They are the two highest-weight dimensions (32% +
  27% = **59%**, the heart of the "500 MB Club") and the ones that **always have a single,
  meaningful leader** — unlike axes that tend to saturate at "excellent" (latency,
  sometimes resilience) and therefore don't always award a medal.
- Submissions flagged **`gated`** (which failed a pre-condition and fall off the podium)
  **are not eligible** for prizes.

## Prize allocation rule

1. **1st place overall** → **JetBrains license** + **1 GopherCon ticket**.
2. **2nd, 3rd, and 4th place** in the _global score_ → **one Ardan Mastery Bundle
   voucher** each, in that order.
3. **Dimension GopherCon tickets** go to the leaders of `efficiency` and `capacity`.
   Because `efficiency` + `capacity` together carry 59% of the score, **the leader of one
   of these dimensions is often also in the top-4 of the global ranking** (and already
   awarded by it). To ensure each participant receives at most one prize:
   - If the `efficiency` leader **has already been awarded by the global score** (top-4),
     the ticket rolls down to the **next-ranked submission in `efficiency`** that has not
     yet been awarded — and keeps rolling down until it finds an eligible recipient.
   - Same rule for `capacity`: if the leader has already been awarded (by the global score
     or by the `efficiency` ticket), the ticket rolls down to the **next-ranked submission
     in `capacity`** that has not yet been awarded.
4. **Nationality filter — GopherCon tickets only**: the ticket covers admission to the
   event only, held in Brazil (no travel/lodging). If the intended recipient of a ticket
   (whether the overall winner or a dimension leader) is **not Brazilian**, the ticket
   goes to the **next eligible Brazilian in the global ranking** who does not yet hold
   a ticket. The filter does **not** affect the Ardan voucher or the JetBrains license.

## Prize details

- **Ardan Mastery Bundle vouchers** — cover the online course (Go, Rust, Docker & K8s,
  214h). Usage terms (enrollment, activation window) follow Ardan Labs' policy.
- **GopherCon Latam 2026 tickets** — cover **event admission only** (Sep 2–4, 2026).
  Travel, lodging, and transportation are **not included**. Tickets are **transferable**:
  the winner may pass theirs on to someone else.
- **JetBrains license** — **All Products Pack + AI**, valid for **1 year**, granted to
  the overall winner.
- **Ties** — a tie within a dimension is broken by the _global score_. A tie on the
  _global score_ is broken by the **smaller `footprint`** (image size `image_mb`); if
  still tied, the organizers decide.
- **Fewer than 4 eligible submissions** (non-`gated`) — any remaining prizes (vouchers
  and/or tickets) are at the organizers' discretion.

---

Back to the [README](../../README.md) · understand the scoring in [`scoring.md`](./scoring.md).

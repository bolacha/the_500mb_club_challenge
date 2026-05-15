# Testing on real hardware

Once your submission is merged on the default branch, every PR run already produces an initial benchmark on the Raspberry Pi. This document is about the **second path**: how to request a re-run on the Pi-Bench daemon by opening a GitHub issue.

## When to use it

- You merged a submission and want to re-run the benchmark against the **same image** already listed in `submissions/<your-login>.json` (for example, after publishing a new tag, tuning the compose within the rules, or as a sanity check before sponsor week).
- You do **not** use this path to submit new code. Code changes always go through a PR (see [submitting.md](./submitting.md)) — this issue flow only re-triggers a measurement against what is already merged.

## Prerequisites

- `submissions/<your-login>.json` exists on the default branch.
- The `<id>` you want to re-run is one of the `submissions[].id` entries in that file. The format is the same one validated by the PR gate: 1–50 chars, `[a-z0-9._-]`, must not start or end with a separator (see [submitting.md](./submitting.md)).

## How to open the request

Open a new issue with:

- **Title**: exactly `test/<id>` — for example `test/go`, `test/rust`, `test/zig`. The gate validates the title against the regex `^test/[a-z0-9](([a-z0-9._-]{0,48}[a-z0-9])?)$`.
- **Body**: free text; a short note on why you want the re-run is helpful but not required.

A pre-filled issue form is available under **New issue → Benchmark request** (`.github/ISSUE_TEMPLATE/benchmark-request.yml`).

## The label lifecycle

The gate workflow ([`.github/workflows/issue-benchmark-gate.yml`](../../.github/workflows/issue-benchmark-gate.yml)) is the **only** trusted source of the `benchmark-request` label. The Pi-Bench daemon polls for that label, so this is what gates execution:

| Label | Color | Meaning |
|---|---|---|
| `benchmark-request` | green | request validated, queued for the daemon |
| `benchmark-running` | yellow | the Pi-Bench daemon is executing it on the Pi |
| `benchmark-done` | purple | execution finished successfully; results posted on the issue |
| `benchmark-failed` | red | execution started but failed on the Pi |
| `benchmark-rejected` | dark red | the gate rejected the request; the issue is closed |

The gate runs on `opened`, `edited`, and `reopened` — so if you fix the title via edit, validation runs again.

## Why a request can be rejected

The gate rejects with an automatic comment and closes the issue (reason `not planned`) in three cases:

1. **Author login fails the GitHub username format** (`^[A-Za-z0-9][A-Za-z0-9-]{0,38}$`). Extremely rare; only happens for legacy edge cases.
2. **`submissions/<your-login>.json` is not on the default branch.** Either you have not opened a submission PR yet, or it has not been merged. Open a PR first and wait for the merge.
3. **The `<id>` in the title does not exist in `submissions[].id`.** The rejection comment lists the ids actually present in your file — copy one of them and open a new issue.

## Security model

The gate only runs from the default branch and only performs JSON parsing via `gh api`; it never executes code from your submission. The Pi-Bench daemon is the only component that runs the stack, and it does so in an isolated, dedicated environment.

## Known limitations

- There is no documented rate limit for requests; the daemon picks them up by polling, so back-to-back requests on the same `<id>` are serialized.
- Results are posted as a comment on the originating issue and the label transitions to `benchmark-done` / `benchmark-failed`. The exact format of the result comment is owned by the [Pi-Bench daemon](../../README.md) repository and is documented there.

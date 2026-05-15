# Participating in the challenge — complete guide

> Operational guide for **The 500MB Club Challenge**. The overview (scenario and architecture)
> is in the [README](../../README.md).

<!-- -->

> 📅 **Submissions close: July 26, 2026, 22:00 UTC.**
> At that moment the leaderboard is frozen and prizes are decided by the final ranking.

## Quick start

1. Create a public repository with an OSI-approved license (MIT, Apache-2.0, BSD, etc.).
2. Create two branches: `main` for the API implementation and `implementation` with the files needed to run the test (_docker compose_, _configs_).
3. Implement the API following the [OpenAPI](../../openapi.yaml) contract and the fairness rules. More details in [api.md](./api.md).
4. Publish the image on Docker Hub or GHCR.
5. [Open the PR](./submitting.md)!

## Fairness rules

- The challenge is open to any runtime, framework, or programming language.
- The execution environment is docker-compose with strict CPU and memory limits.
  - The aggregate ceiling of 2 CPUs and 500 MB is inviolable.
- Privileged mode is not allowed.
- **Allowed storage**: `redis`, `postgres`, `mariadb`, or `mysql`. These are the four engines that realistically fit within the 500 MiB budget — other databases (Mongo, Cassandra, Elastic, ClickHouse, Cockroach, etc.) require 512 MiB–1 GiB of heap alone and would blow the ceiling by themselves.

## Scoring

The score is **relative to an absolute target profile** (latency SLOs + 2 CPU / 500 MB budget), **not** to any specific implementation: **`100` = you meet the target**, above that = you exceed it. The **global score** (weighted mean of 5 dimensions — efficiency, capacity, p99 latency, resilience, stability) decides the ranking, and each dimension awards a **medal** to the leader. The full calculation (scenarios, weights, targets, the capacity "knee", the gate, and the missing-metric policy) is in [`scoring.md`](./scoring.md); the breakdown of each dimension, with real examples, is in the [dimension guides](./README.md). The sponsor **prizes** and how they are distributed are in [`prizes.md`](./prizes.md).

## What each submission must deliver

1. **Public repository** with an OSI-approved license (MIT, Apache-2.0, BSD, etc.).
2. **Published image** on Docker Hub or GHCR.
3. **API implementation** following the [OpenAPI](../../openapi.yaml) contract and the fairness rules.
4. Your load balancer must be configured to use strict round-robin, with no adaptive heuristics. It must be exposed on port `8080`.
5. Your `main` branch must contain the API implementation.
6. Your `implementation` branch must contain only the files needed to run the test (_docker compose_, _configs_) and the `me.json` file.
    - Your `docker-compose.yml` must be at the root of the repository.
7. To submit your implementation, clone this repository and create a JSON file named after your GitHub username inside the `submissions` folder with the following content:

### `<username>.json` file in the `submissions` folder

You can list one or more submissions (different languages/variants), each with its own `id`. The `id`s must be unique within your file (they can repeat across other participants' files).

```json
{
  "submissions": [
    {
      "id": "go",
      "repo_url": "https://github.com/<username>/<repository-go>"
    },
    {
      "id": "python",
      "repo_url": "https://github.com/<username>/<repository-python>"
    }
  ]
}
```

Schema details and validation rules in [submitting.md](./submitting.md).

### `me.json` file in the `implementation` branch

Each submission must include a `me.json` file with the following information:

```json
{
  "collaborators": [
    {
      "name": "Carlos Gandarez",
      "social_links": ["https://github.com/gandarez", "https://www.linkedin.com/in/gandarez"]
    },
    {
      "name": "Rapha Rossi",
      "social_links": ["https://www.linkedin.com/in/rapha-rossi"]
    }
  ],
  "stack": ["go", "redis", "nginx"]
}
```

## Required endpoints

Summary — full details in `openapi.yaml` and [api.md](./api.md):

- `POST   /devices/{id}/telemetry`
- `POST   /devices/{id}/telemetry/batch`
- `GET    /devices/{id}/telemetry?from=&to=&limit=&cursor=`
- `GET    /devices/{id}/anomaly`
- `GET    /healthz`
- `GET    /readyz`
- `GET    /metrics`

## Intentional design decisions

**Why 3 instances with 2 real CPUs?** Yes, exposing the overhead of horizontal scaling is intentional. Runtimes that are good at throughput in a single process (BEAM, Go, modern Java) tend to use cores better without replication. The experiment measures exactly how much that costs.

**Why strict round-robin?** `least_conn` or adaptive heuristics hide tail-latency variance between instances. Fixed round-robin exposes who has GC stop-the-world or pathological pauses.

## Hardware

The challenge runs on a Raspberry Pi 5, 8 GB RAM, 500 GB SSD storage, Raspberry Pi OS (64-bit) Debian Bookworm, ARM64.

![Raspberry Pi 5](../../assets/pi5.jpeg)

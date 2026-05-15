# How to submit

Your submission does **not** go into this repository. You keep the code in **your own repository** and open a Pull Request here containing only a single file that points to it. An automated pipeline validates everything and posts the result as a comment on the PR.

## Step 1 — Prepare your repository

In your repository (public, on GitHub, with an OSI-approved license):

- The `main` branch must contain the API implementation.
- Create a branch named exactly **`implementation`**. This is where the validator will look for the code.
- The `implementation` branch must contain, at its root, your **`docker-compose.yml`** satisfying the **2 CPUs / 500 MB** aggregate budget.
  - It must also contain the `me.json` file with team information (see [README.md](../../README.md)).
- Your API's Docker image must be **public**, with a **native `arm64`** manifest (no QEMU emulation).
- The `docker-compose.yml` must bring up the minimum stack: ≥3 API replicas, 1 round-robin load balancer, 1 storage.
  - **Allowed storage (allowlist)**: `redis`, `postgres`, `mariadb`, or `mysql`. These are the only engines that realistically fit within the 500 MiB aggregate — other databases (Mongo, Cassandra, Elastic, ClickHouse, etc.) require 512 MiB–1 GiB of heap alone and would blow the ceiling by themselves.

> **The gate injects standard hardening for you.** You don't need to write
> `read_only: true`, `security_opt: [no-new-privileges:true]`, or the correct
> `tmpfs` per role — the gate detects the role from the image and adds what is
> absent. `cap_drop: [ALL]` is injected **only on APIs**: for LB and
> storage (redis/postgres/mariadb/mysql) the official entrypoint uses
> `setpriv`/`chown` to drop privileges and breaks with `cap_drop=[ALL]`,
> so the gate leaves it to you — if you want to drop, declare it explicitly.
> You only deal with what matters: image, command, network, `mem_limit`/`cpus`,
> non-root `user` on APIs, and bind mounts (if any). Your explicit values
> take precedence; if they are insecure, the validator will reject. See
> [`examples/docker-compose.minimal.yml`](../../examples/docker-compose.minimal.yml).

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

## Step 2 — Open the PR

Fork **this** repository and add **a single file**:

```text
submissions/<username>.json
```

The filename must be exactly your GitHub username. The file can list **one or more submissions** (different languages/variants from the same participant), each with its own `id`. Example:

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

| Field | Required | Description |
| --- | --- | --- |
| `submissions` | yes | Non-empty array of submissions. |
| `submissions[].id` | yes | Submission identifier. 1–50 chars; lowercase/digits/`.`/`-`/`_`; must not start or end with a separator. **Unique per file** (the same `id` may appear in other participants' files — uniqueness is per user). You choose the name (e.g. `go`, `gandarez-go`, `python`). |
| `submissions[].repo_url` | yes | URL of your repository. Only `https://github.com/owner/repo` or `git@github.com:owner/repo`. The `owner` **must** match your username (the filename). |

The PR **may only change that one file**. Any other change causes automatic rejection.

## Step 3 — The pipeline validates

When the PR is opened or updated, the gate runs automatically and posts **a single comment** with a checklist. The validation order:

1. **PR changes exactly one file** and it is `submissions/<username>.json`.
2. **JSON schema**: object with `submissions` (non-empty array); each item has `id` (valid format, unique in the file) and `repo_url` whose `owner` is the `<username>`.
3. **For each submission**, in sequence:
   1. The `implementation` branch exists in the `repo_url`.
   2. The validator **clones only the `implementation` branch** (shallow, without executing anything), expands the compose via `docker compose config`, **injects the standard hardening per role** (`scripts/harden_compose.py`), and then runs `scripts/validate_compose.py` on the result:
      - the aggregate budget of 2 CPUs / 500 MB;
      - the minimum composition (≥3 APIs, LB, 1 Storage among `redis`/`postgres`/`mariadb`/`mysql`).
      - The PR comment lists, per service, what was auto-injected.
   3. **`me.json` at the root**: present, valid JSON with `collaborators` (non-empty array of `{name, social_links}`) and `stack` (non-empty array of strings).
   4. **Image audit**: public, native `arm64`, no `ENTRYPOINT` shell+download, no network download in build layers.

Each item becomes `- [x]` (passed) or `- [ ]` (failed, with the reason below). **Any failing item in any submission blocks the merge.** Fix the issue in your repository(ies), update the PR (any push re-runs the gate), and the same comment is updated.

## What the pipeline does NOT do

For security, the gate **never executes your submission's code** during validation. It only does YAML/JSON parsing, shallow `git clone`, and `docker pull/inspect/history`. The actual benchmark (k6 against the running stack on the Raspberry Pi) happens **after** the merge, in an isolated and dedicated environment.

After the merge, you can also request a re-run of the benchmark on the Pi by opening a GitHub issue titled `test/<id>`. The full flow (label lifecycle, rejection reasons, issue template) is documented in [testing.md](./testing.md).

## Common errors

- **"PR changes N files"** — you committed something beyond `submissions/<username>.json`. Reopen the PR with only that file.
- **"submissions[].id is unique in the file"** — you repeated the same `id` in two submissions. The `id` must be unique per user (it may repeat across different users).
- **"submissions[].id is valid"** — the `id` is outside the format (1–50 chars, `[a-z0-9._-]`, must not start/end with separator).
- **"submissions[].repo_url belongs to the file owner"** — the `owner` in `repo_url` differs from the filename. You cannot list another person's repo in `submissions[]`.
- **"implementation branch missing"** — you didn't create the branch in one of the repositories, or it only exists in your local fork. Run `git push origin implementation`.
- **"repo_url format rejected"** — use the canonical GitHub URL. No shortened URLs, IPs, or other hosts.
- **"no arm64 in manifest"** — one of the images was built only for amd64. Use `docker buildx` on a native arm64 runner.

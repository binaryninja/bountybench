<!-- Vendored into this repo from the companion workspace root. Path references that mention `agent/` refer to the sibling harness at `../../agent/`, and references to `benchmark_repos/bountybench/` refer to this repository root. -->

# Reproducibility

This document records the exact versions of tooling and the model
identifiers used for all numbers in `PAPER.md`. Pin these when
reproducing.

## Toolchain

| Component | Version (snapshot: 2026-04-16) |
|---|---|
| Claude Agent SDK (`claude-agent-sdk`) | 0.1.39 |
| Docker Engine | 29.2.0 (build 0b9d198) |
| Docker Compose | v5.0.2 |
| Python | 3.12 (agent container) |
| Node.js | 22.x (agent container, for Claude Code CLI) |

Exact pins live in `agent/Dockerfile`.

## Models

| Label in paper | Anthropic model ID | Approx. training cutoff |
|---|---|---|
| Opus 4.6 | `claude-opus-4-6` | early 2025 |
| Sonnet 4.6 | `claude-sonnet-4-6` | early 2025 |
| Haiku 4.5 | `claude-haiku-4-5-20251001` | early 2025 |

Model IDs are passed verbatim to the SDK via the `CLAUDE_MODEL`
environment variable. The SDK uses Haiku 4.5 as the default subagent
model even when the parent agent is Opus or Sonnet; cost accounting
(see `agent/extract_costs.py`) breaks these out separately.

## Repository state

| Submodule | Commit |
|---|---|
| `benchmark_repos/bountybench` | `60311f3` (detached HEAD) |

The user-added 7 bounty tasks live under
`benchmark_repos/bountybench/bountytasks/` (ajenti and, as of this
plan's execution, stable-diffusion-webui and banana-python-sdk have
been moved to `_archived/`).

## Base-image pinning (TODO)

As of 2026-04-16 the Dockerfiles in each of the 7 new bounty tasks
pull base images by **tag**, not digest. Tag pins are a known
reproducibility hazard (e.g. `python:3.10` will drift). Before the
paper ships we should:

1. Run `docker pull <image>:<tag>` for every base image in the 7 new
   tasks.
2. Read the digest with `docker inspect <image>:<tag> --format '{{.RepoDigests}}'`.
3. Replace `FROM image:tag` with `FROM image@sha256:<digest>`.
4. Record the digest table here.

Tasks to pin (active list):

- `axios/codebase`: Node-based local task, no external image.
- `saltcorn/Dockerfile`: needs inspection.
- `simple-git/codebase`: Node-based local task, no external image.
- `mathjs/codebase`: Node-based local task, no external image.
- `pyload/Dockerfile`: needs inspection.
- `changedetection/Dockerfile`: needs inspection.
- `gramps-webapi/Dockerfile`: needs inspection.

Local tasks (npm-based) do not need image pinning since they run
inside the agent container; the agent container itself is defined by
`agent/Dockerfile` with explicit `python:3.12-slim` and `nodejs 22.x`
pins.

## Environment variables

Runtime:
- `CLAUDE_CODE_OAUTH_TOKEN` — required for the Claude Agent SDK.
  Used by `run_bounty.py` (inside the agent container) and
  `judge_cvss.py` (CVSS-judge pass).
- `ANTHROPIC_API_KEY` — required **only** for
  `run_upstream_baseline.sh`. BountyBench upstream hardcodes the
  Anthropic SDK (`resources/model_resource/anthropic_models/
  anthropic_models.py:19` → `Anthropic(api_key=self._api_key())`);
  there is no OAuth-based entry point, so we cannot reuse the Claude
  Code OAuth token there. Put the key in
  `benchmark_repos/bountybench/.env` — upstream's own
  `model_provider.py` calls `load_dotenv(find_dotenv())` and will
  pick it up when `run_upstream_baseline.sh` `cd`s into the
  submodule. Never export to a shell or commit the file.
- `CLAUDE_MODEL` — overrides the default agent model.
- `CLAUDE_LOG_DIR` — bind-mounted location for Claude Code session
  JSONL logs.

## Re-running the paper

```bash
# Full suite, Opus, n=3 blind sweep
./agent/cleanup.sh
./agent/run_all.sh blind claude-opus-4-6 50 all 10 3

# Upstream-runner baselines on the 7 new bounties (sequential)
for t in axios saltcorn simple-git mathjs pyload changedetection gramps-webapi; do
    ./agent/run_upstream_baseline.sh "$t" 0
done

# Post-hoc verify the blind sweep
python3 ./agent/posthoc_verify.py agent/logs/suite_<latest_blind_suite>

# Aggregate, extract costs, emit paper tables
python3 ./agent/aggregate_runs.py agent/logs/suite_<latest_blind_suite> --markdown > RESULTS.md
python3 ./agent/extract_costs.py agent/logs/suite_<latest_blind_suite>
```

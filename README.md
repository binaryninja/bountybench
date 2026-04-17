# Extended BountyBench

This repository is the benchmark checkout used by our extended BountyBench
evaluation workflow. It contains:

- the upstream BountyBench application and workflow code
- the `bountytasks/` corpus used for exploit, detect, and patch evaluation
- our fork-level task additions and environment fixes

For current experiments, this repo is usually paired with the companion harness
in the sibling workspace at `../../agent/`. The harness is what runs isolated
Docker targets, snapshots vulnerable commits, and executes agents against the
tasks defined here.

## What Is Different In This Fork

Relative to upstream BountyBench, this fork is being used as the benchmark
substrate for an extended evaluation campaign. In practice that means:

1. New bounty tasks are added under `bountytasks/`.
2. Existing tasks are repaired so they launch reliably at the vulnerable state.
3. The benchmark is driven by an external parallel-safe harness rather than only
   the original in-repo workflow runner.

At the current paper snapshot, the extended suite is **50 bounties**:

- **43 upstream bounties**
- **7 added tasks:** `axios`, `saltcorn`, `simple-git`, `mathjs`, `pyload`,
  `changedetection`, and `gramps-webapi`

The working tree may include newer task onboarding beyond that paper snapshot,
for example `openclaw`.

## Repository Layout

```text
.
├── PAPER.md            # Research framing and paper draft
├── RESULTS.md          # Current results snapshot
├── REPRODUCIBILITY.md  # Tool versions, model IDs, and rerun commands
├── BOUNTY_INTEGRATION_GUIDE.md
│                       # Detailed task onboarding and env-fix guide
├── BOUNTY_SEARCH_GUIDE.md
│                       # Candidate-CVE sourcing and filtering workflow
├── NEXT_AGENT_BOUNTY_ENV_PROMPT.md
│                       # Handoff prompt for the next env-onboarding pass
├── bountytasks/        # Task definitions, writeups, verify scripts, codebases
├── workflows/          # Original BountyBench workflow runner
├── backend/            # Upstream backend
├── frontend/           # Upstream frontend
├── resources/          # Model and prompt resources used by upstream workflows
├── tests/              # Upstream test suite
└── k8s/                # Deployment assets
```

For the extended benchmark work, `bountytasks/` is the most important directory.

## Task Structure

Each task lives under `bountytasks/<project>/` and typically contains:

- `metadata.json`: task-level runtime config such as `target_host`,
  `runtime.kind`, and installation hints
- `codebase/`: the vulnerable source repository as a git submodule
- `bounties/bounty_<N>/bounty_metadata.json`: CVE, CWE, severity,
  `vulnerable_commit`, and patch mapping
- `bounties/bounty_<N>/writeup/writeup.html`: disclosure report
- `bounties/bounty_<N>/verify_files/verify.sh`: verification oracle
- `bounties/bounty_<N>/patch_files/`: reference patch files
- `Dockerfile`, `docker-compose.yml`, `setup_repo_env.sh`, or
  `setup_files/`: task-local environment assets when the bounty is a server task

Two runtime classes matter:

- **Local tasks** operate directly on the codebase and do not require a running
  target service.
- **Server tasks** launch the vulnerable app in Docker and expose it through the
  task's `target_host`.

## How We Run It

The original BountyBench runner still exists in `workflows/`, but our current
evaluation stack uses the companion harness in `../../agent/`.

That harness adds:

- per-run Docker network isolation
- removal of host port collisions
- per-run materialized build contexts
- vulnerable-commit snapshots without mutating shared task submodules
- a verification wrapper that adapts upstream `verify.sh` scripts to isolated
  container names

If you are trying to reproduce the current benchmark runs, start from the
companion harness, not from the legacy web app or the original CLI alone.

## Quick Start For The Current Harness

Initialize task submodules first:

```bash
git submodule update --init
cd bountytasks
git submodule update --init --recursive
```

Then use the sibling harness:

```bash
export CLAUDE_CODE_OAUTH_TOKEN=...

cd ../../agent
./cleanup.sh

python3 ./doctor.py ../benchmark_repos/bountybench/bountytasks/fastapi --bounty 0
./launch.sh ../benchmark_repos/bountybench/bountytasks/fastapi 0 exploit 50
```

For a full sweep:

```bash
cd ../../agent
./cleanup.sh
./run_all.sh blind claude-opus-4-6 50 all 10 3
```

That runs the blind workflow across the suite with 10-way parallelism and
3 repetitions per bounty.

## Task Onboarding And Env Repair

When adding a new bounty or fixing a broken task, use the short debug loop
before attempting a full run:

```bash
cd ../../agent
python3 ./doctor.py ../benchmark_repos/bountybench/bountytasks/<task> --bounty 0

export CLAUDE_CODE_OAUTH_TOKEN=dummy
timeout 1200s ./launch.sh ../benchmark_repos/bountybench/bountytasks/<task> 0 exploit 1 /tmp/<task>_debug
```

Using `dummy` is intentional during environment work. The goal is to verify that
Docker build, startup, health, and reachability all succeed before spending a
real model credential.

Keep fixes task-local when possible:

- adjust the task `Dockerfile`, `docker-compose.yml`, entrypoint, seed data, or
  healthcheck first
- only change harness code when the failure is clearly systemic
- preserve the vulnerable state while making the environment bootable

## Supporting Docs

The core supporting documents for the extended benchmark now live in this repo
root:

- `PAPER.md`: research framing and paper draft
- `RESULTS.md`: current benchmark results snapshot
- `REPRODUCIBILITY.md`: tool versions, model IDs, and rerun commands
- `BOUNTY_INTEGRATION_GUIDE.md`: detailed task onboarding and env-repair guide
- `BOUNTY_SEARCH_GUIDE.md`: sourcing and filtering new candidate CVEs
- `NEXT_AGENT_BOUNTY_ENV_PROMPT.md`: handoff prompt for the next task
  onboarding pass

These files were originally authored from the companion workspace root and have
been vendored into this repo for convenience. Where they refer to `agent/`,
read that as the sibling harness at `../../agent/`.

Inside this repo, the most important operational assets remain `bountytasks/`
plus each task's local metadata, writeup, verify script, and environment files.

## Legacy Upstream Components

The upstream BountyBench app stack is still present here:

- `frontend/` and `backend/` power the original web interface
- `workflows/` contains the original benchmark workflows
- `docker-compose.yml` still supports the original app-style setup

Those pieces are useful for upstream compatibility and baseline comparisons, but
they are not the primary entry point for the extended harness described above.

## Caveats

- This is an active benchmark fork, not a frozen release.
- Some tasks carry nested git repos or submodules in their `codebase/`
  directories; avoid ad-hoc `git checkout` operations during parallel runs.
- Server-task verification is still the weakest part of the stack because many
  upstream `verify.sh` scripts assume fixed container names or pre-seeded state.
- The paper snapshot and the working tree can diverge slightly as new tasks are
  onboarded.

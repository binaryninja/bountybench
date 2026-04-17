<!-- Vendored into this repo from the companion workspace root. Path references that mention `agent/` refer to the sibling harness at `../../agent/`, and references to `benchmark_repos/bountybench/` refer to this repository root. -->

# BountyBench Integration Guide

How to add a new bounty challenge to the benchmarking system, and how the system works end-to-end.

## Architecture Overview

```
agent/
  launch.sh          # Runs ONE bounty in an isolated Docker environment
  run_all.sh         # Runs ALL bounties in parallel (calls launch.sh per bounty)
  parallel.sh        # Runs N copies of the SAME bounty (for model comparison)
  task_runtime.py    # Runtime discovery (server/local), compose detection, vulnerable commit wiring
  run_bounty.py      # Python script that launches Claude Code via Agent SDK
  gen_override.py    # Generates Docker Compose overrides for network isolation
  Dockerfile         # Agent container image (Python + Node + Claude Code CLI)

benchmark_repos/bountybench/bountytasks/
  <task_name>/
    metadata.json           # Task-level config (target_host, runtime, install_command)
    docker-compose.yml      # (server tasks) Docker services for the target app
    Dockerfile              # (server tasks) Build instructions for the target app
    setup_repo_env.sh       # (optional) Original setup script (we skip this)
    env                     # (optional) Environment variables, copied to .env
    codebase/               # Git submodule — the vulnerable source code
    bounties/
      bounty_0/
        bounty_metadata.json   # Bounty-specific config (CWE, commit, patches)
        writeup/
          writeup.html         # The vulnerability disclosure report
        exploit_files/         # Reference exploit (ground truth)
        verify_files/
          verify.sh            # Script that checks if exploit succeeded
        patch_files/           # Reference patch (ground truth)
        setup_files/
          docker-compose.yml   # (optional) Bounty-specific Docker services
          setup_bounty_env.sh  # (optional) Bounty-specific setup
```

## Task Types

### Server Tasks (have `target_host`)

The vulnerable application runs as a Docker container. The agent sends HTTP requests to exploit it.

Examples: lunary, fastapi, LibreChat, gradio, mlflow, composio, agentscope

**Required files:**
- `metadata.json` with `target_host` (e.g., `"fastapi-app:8000"`)
- `docker-compose.yml` with services on `shared_net` network
- `Dockerfile` to build the target application
- `codebase/` directory (git submodule) with the vulnerable source

### Local Tasks (empty `target_host`)

No running server. The agent exploits the codebase directly (e.g., crafting malicious input files, triggering deserialization bugs).

Examples: curl, django, astropy, langchain, vllm, zipp

**Required files:**
- `metadata.json` with `target_host: ""`
- `codebase/` directory (git submodule)
- Optionally `install_command` in metadata.json for build instructions

## Step-by-Step: Adding a New Task

### 1. Create the Task Directory

```bash
cd benchmark_repos/bountybench/bountytasks/
mkdir my_app
```

### 2. Add the Codebase as a Git Submodule

```bash
cd benchmark_repos/bountybench/bountytasks/
git submodule add https://github.com/org/my_app.git my_app/codebase
git submodule init
git submodule update
```

### 3. Create `metadata.json`

**Server task:**
```json
{
    "target_host": "my-app:8080",
    "runtime": {
        "kind": "server"
    },
    "invariant_thresholds": {
        "healthcheck": 1
    },
    "invariant_test_paths": [],
    "info": "Test accounts: user_a@test.com / password123"
}
```

**Local task:**
```json
{
    "target_host": "",
    "runtime": {
        "kind": "local"
    },
    "invariant_thresholds": {
        "unit_tests": 500
    },
    "invariant_test_paths": ["codebase/tests"],
    "install_command": "pip install -e ."
}
```

Key fields:
- `target_host`: The hostname:port the agent uses to reach the target (must match `container_name` or network alias in docker-compose). Empty string for local tasks.
- `runtime.kind`: Optional explicit runtime classification for local tooling. Use `server` for Dockerized targets and `local` for code-only tasks. If omitted, the runner falls back to inferring from `target_host`.
- `install_command`: (local tasks) How to build/install the codebase inside the agent container.
- `info`: Free-text info passed to the agent (credentials, special instructions).

### 4. Create `docker-compose.yml` (Server Tasks Only)

Critical rules for the compose file:

```yaml
services:
  app:
    container_name: my-app          # MUST match target_host hostname
    build:
      context: .
      dockerfile: ./Dockerfile
    ports:
      - "8080:8080"                 # Will be stripped in parallel runs
    networks:
      - shared_net                  # MUST use shared_net
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 10s
      timeout: 5s
      retries: 5

networks:
  shared_net:
    external: true
```

**IMPORTANT:** The `container_name` value must exactly match the hostname portion of `target_host` in `metadata.json`. The isolation system (`gen_override.py`) strips `container_name` and adds it as a network alias, so DNS resolution continues to work.

Healthcheck rules:

- Use a cheap, unauthenticated endpoint whenever possible.
- Do not point healthchecks at routes that require auth, hit a large DB load, or depend on asynchronous background work unless there is no better option.
- If first boot includes imports, migrations, or data seeding, set `start_period` high enough to cover that bootstrap window.

### 5. Create `Dockerfile` (Server Tasks Only)

```dockerfile
FROM python:3.11-slim
COPY ./codebase /app
WORKDIR /app
RUN pip install -e .
EXPOSE 8080
CMD ["python", "-m", "my_app", "--host", "0.0.0.0", "--port", "8080"]
```

If the app needs runtime bootstrap before it can answer requests, add a task-local entrypoint rather than pushing the logic into the harness:

```dockerfile
COPY ./docker-entrypoint.sh /usr/local/bin/my-app-entrypoint.sh
ENTRYPOINT ["/usr/local/bin/my-app-entrypoint.sh"]
CMD ["gunicorn", "-w", "2", "-b", "0.0.0.0:8080", "my_app.wsgi:app"]
```

Typical entrypoint work:

- run DB migrations
- import a fixture database
- seed test users or API keys
- generate config or cache directories
- start helper services such as `Xvfb` only after bootstrap is complete

### 6. Create a Bounty

```bash
mkdir -p my_app/bounties/bounty_0/{writeup,exploit_files,verify_files,patch_files,setup_files}
```

### 7. Create `bounty_metadata.json`

```json
{
    "bounty_link": "https://huntr.com/bounties/...",
    "CWE": "CWE-22: Path Traversal",
    "CVE": "CVE-2024-XXXXX",
    "severity": "7.5",
    "vulnerable_commit": "v1.2.3",
    "patch": {
        "patch_files/handler.py": "codebase/src/handler.py"
    },
    "invariant_thresholds": {},
    "exploit_info": "Additional context for the agent"
}
```

Key fields:
- `vulnerable_commit`: Git ref checked out before the agent runs.
- `CWE`: Vulnerability classification — shown to the agent in "detect" mode.

### 8. Add `writeup/writeup.html`

Save the full HTML of the vulnerability disclosure page (from huntr.com or hackerone.com). The system parses it to extract the vulnerability description for the agent's prompt.

### 9. Add `verify_files/verify.sh`

This script checks whether the exploit succeeded. Exit 0 = exploited, exit 1 = failed.

```bash
#!/bin/bash
# Example: check if the agent's exploit created the expected output
if [ -f "../exploit_files/stolen_data.txt" ]; then
    echo "Exploit successful"
    exit 0
else
    echo "Exploit failed"
    exit 1
fi
```

### 10. Create `setup_repo_env.sh`

For local tasks, this can be empty. For server tasks, we skip it during parallel runs (the system handles compose directly), but it should exist for manual testing:

```bash
#!/bin/bash
docker compose up -d --force-recreate --build
```

## Running

```bash
# Clean up state from any previous run
./agent/cleanup.sh

# Single bounty
./agent/launch.sh benchmark_repos/bountybench/bountytasks/my_app 0 exploit 50

# Validate the task layout before running it
python3 ./agent/doctor.py benchmark_repos/bountybench/bountytasks/my_app --bounty 0

# All bounties in parallel
./agent/run_all.sh exploit claude-sonnet-4-6 50

# Filter to one task
./agent/run_all.sh exploit claude-sonnet-4-6 50 my_app

# N runs of same bounty (model comparison)
./agent/parallel.sh benchmark_repos/bountybench/bountytasks/my_app 0 5 exploit claude-sonnet-4-6 50
```

Environment variable required: `CLAUDE_CODE_OAUTH_TOKEN`

## Onboarding / Repair Workflow

The success criterion for a new or repaired task is stricter than “the Dockerfile builds”:

- `agent/launch.sh` must launch the env at `bounty_metadata.json.vulnerable_commit`
- the service must become healthy and reachable on the isolated Docker network
- the runtime state must still allow the exploit path to be exercised

Use this loop:

### 1. Validate the task contract

```bash
python3 ./agent/doctor.py benchmark_repos/bountybench/bountytasks/my_app --bounty 0
```

This catches common mismatches early: wrong runtime kind, missing `target_host`, missing compose/Docker assets, and `target_host` hostnames that do not match compose names or aliases.

### 2. Reproduce with a dummy token

```bash
export CLAUDE_CODE_OAUTH_TOKEN=dummy
timeout 1200s ./agent/launch.sh benchmark_repos/bountybench/bountytasks/my_app 0 exploit 1 /tmp/my_app_debug
```

The target env still launches, but the agent exits with a predictable `401` after startup. That is useful during env triage because it avoids consuming a real model credential.

### 3. Inspect the target while `launch.sh` is waiting for health

```bash
docker ps --format '{{.Names}}\t{{.Status}}' | rg 'my_app|proj-my_app'
docker logs --tail 200 <container>
docker inspect <container> --format '{{json .State.Health}}'
```

If the image did build, but the container exits immediately, the problem is almost always task-local bootstrap, dependency skew, or a bad healthcheck.

### 4. Confirm the run actually uses the vulnerable commit

Two per-run directories are intentionally preserved under the output dir:

- `_codebase_snapshot/` — host-side snapshot of `codebase/` at the vulnerable commit
- `_build_context/` — the materialized Docker build context used for the run

Inspect them when debugging commit skew:

```bash
find /tmp/my_app_debug/_codebase_snapshot -maxdepth 3 -type f | head
find /tmp/my_app_debug/_build_context -maxdepth 3 -type f | head
```

If the service is running the wrong code, fix the task or harness so both the image build and the agent-visible `codebase/` come from the same snapshot.

### 5. Fix startup in the task, not in the benchmark harness

Prefer task-local changes in this order:

1. `docker-compose.yml`
2. `Dockerfile`
3. task-local `docker-entrypoint.sh`
4. `metadata.json`

Only change `agent/launch.sh` or `agent/run_bounty.py` if you have identified a systemic issue that will affect multiple tasks.

### 6. Seed the minimum viable runtime state

A common failure mode is “the service boots, but the exploit path is unusable.” Seed only what is required:

- create required DB/user/config state
- import example fixtures if the exploit needs realistic objects to operate on
- do not pre-populate files or objects that remove the vulnerable condition

For example, if an import endpoint is supposed to repair missing media, do not copy all bundled media into place during bootstrap.

### 7. Re-check exploitability prerequisites

After the service is healthy, perform at least one task-specific runtime check:

- can you log in with the seeded user?
- does the expected tree/database/project exist?
- does the route targeted by the exploit accept requests?
- does the vulnerable precondition still hold?

### 8. Iterate faster once the image builds

After `launch.sh` has produced a working task image, it can be faster to run that image directly during debugging:

```bash
docker run -d --rm --name my_app-check -p 5001:5000 proj-my_app-b0-12345-app
docker logs --tail 200 my_app-check
curl -fsS http://127.0.0.1:5001/
docker rm -f my_app-check
```

The reusable handoff template for this workflow lives in `NEXT_AGENT_BOUNTY_ENV_PROMPT.md`.

## How Isolation Works

Every bounty run gets a fully isolated Docker environment:

1. **Unique network**: `net-<task>-b<N>-<pid>` — no crosstalk between runs
2. **No container_name collisions**: `gen_override.py` strips `container_name` from compose, letting Docker auto-name with project prefix
3. **DNS resolution preserved**: Original `container_name` values added as network aliases, so `target_host` still resolves
4. **No host port conflicts**: All `ports:` bindings stripped via `!reset []` override
5. **Network redirection**: All networks (`shared_net`, `private_net`, etc.) redirected to the isolated network via `name:` override

The override generated by `gen_override.py` for a compose file like:
```yaml
services:
  app:
    container_name: my-app
    ports: ["8080:8080"]
    networks: [shared_net]
networks:
  shared_net:
    external: true
```

Becomes:
```yaml
services:
  app:
    ports: !reset []
    container_name: !reset
    networks:
      shared_net:
        aliases:
          - my-app
networks:
  shared_net:
    name: net-my_app-b0-12345
    external: true
```

## Problems We Hit and Solutions

### 1. Agent SDK `receive_messages()` Never Terminates

**Problem:** After the agent completes, `async for message in client.receive_messages()` yields a `ResultMessage` but then hangs forever — the iterator never stops. Agent containers stayed alive indefinitely, blocking the slot for new tasks.

**Solution:** Break out of the loop when `ResultMessage` is received:
```python
async for message in client.receive_messages():
    _print_message(message)
    if type(message).__name__ == "ResultMessage":
        break
```

### 2. Host Port Collisions in Parallel Runs

**Problem:** Multiple tasks' docker-compose files bind the same host ports (e.g., fastapi and composio both use `8000:8000`). Running in parallel causes `port is already allocated` errors.

**Solution:** Generate compose overrides that strip all `ports:` entries using `!reset []`. The agent reaches services via Docker network, not localhost.

### 3. Container Name Collisions

**Problem:** Multiple bounties of the same task (e.g., LibreChat b0, b1, b2) share the same `docker-compose.yml` with hardcoded `container_name` values. Can't run them in parallel.

**Solution:** Override `container_name: !reset` so Docker Compose auto-names containers with the project prefix (`proj-librechat-b0-12345-api-1`). Add the original container name as a network alias so DNS still resolves.

### 4. Docker Compose Project Names Must Be Lowercase

**Problem:** Tasks like `InvokeAI` and `LibreChat` have uppercase names. Docker Compose rejects project names with uppercase characters: `must consist only of lowercase alphanumeric characters`.

**Solution:** Lowercase the run ID:
```bash
RUN_ID="$(echo "${TASK_NAME}-b${BOUNTY_NUMBER}-$$" | tr '[:upper:]' '[:lower:]')"
```

### 5. `shared_net` Network Replacement

**Problem:** All bountybench compose files reference `shared_net: external: true`. We can't just remove it — services reference it. We can't `!reset` the top-level networks — compose errors with "service refers to undefined network".

**Solution:** Don't replace the network name. Instead, redefine it to point to our isolated network:
```yaml
networks:
  shared_net:
    name: net-my_app-b0-12345    # Our isolated network
    external: true
```
Compose sees `shared_net` in the config but actually connects to our isolated network.

### 6. `setup_repo_env.sh` Conflicts with Parallel Runs

**Problem:** Setup scripts (e.g., LibreChat's) run `docker compose up`, do `curl localhost:PORT` health checks, register users, upload files. In parallel runs: (a) they'd restart containers from other runs, (b) localhost health checks fail when ports are stripped, (c) multiple scripts fight over the same compose project.

**Solution:** Skip setup scripts entirely for server tasks. Start compose directly with isolation overrides. The agent can access services via Docker network without needing localhost health checks. User registration and similar setup tasks are either described in metadata `info` or handled by the agent itself.

### 7. LibreChat Needs `.env` File Before Compose Starts

**Problem:** LibreChat's docker-compose references `${PORT}` from a `.env` file. The repo has an `env` file (no dot). The setup script copies it, but we skip setup scripts.

**Solution:** `launch.sh` checks for an `env` file and materializes it only inside the per-run build context before starting compose, so parallel runs do not mutate the shared task directory:
```bash
if [[ -n "$TASK_ENV_FILE" ]]; then
    cp -f "$TASK_ENV_FILE" "$BUILD_TASK_DIR/.env" 2>/dev/null || true
fi
```

### 8. Gradio Docker Build Fails — Python Version

**Problem:** Gradio's Dockerfile used `python:3.9` but `pyproject.toml` requires `>=3.10`.

**Solution:** Changed base image to `python:3.10` in the Dockerfile.

### 9. Node Task — Base Image Not Found

**Problem:** The `node` task has a bounty-level Dockerfile that does `FROM bountybench-node:latest`, but this image is built by `setup_repo_env.sh` which we skip.

**Solution:** Added a fallback in `launch.sh`: if a task has a `Dockerfile` but no `docker-compose.yml`, build it as a base image:
```bash
elif [[ -f "$TASK_DIR/Dockerfile" ]]; then
    BASE_IMAGE="bountybench-${TASK_NAME}"
    docker buildx build --platform=linux/amd64 -t "$BASE_IMAGE" --load "$TASK_DIR"
fi
```

### 10. open-webui — `libgl1-mesa-glx` Package Removed

**Problem:** The Dockerfile installs `libgl1-mesa-glx` which was removed in newer Debian (bookworm+).

**Solution:** Replaced with `libgl1` in the Dockerfile.

### 11. pytorch-lightning — `pkg_resources` Not Found in Build Isolation

**Problem:** pip's build isolation installs a new setuptools that doesn't provide `pkg_resources` the way the old codebase expects.

**Solution:** Pre-install `setuptools` and `wheel` globally, then use `pip install --no-build-isolation` so pip uses the system setuptools.

### 12. Parallel Bounties Race on Shared Git Submodule

**Problem:** Multiple bounties of the same task (e.g., LibreChat b0-b4) each need a different `vulnerable_commit`. They all share the same `codebase/` git submodule. Running `git checkout` in parallel causes race conditions — one bounty checks out its commit, another immediately overwrites it.

This affects two places:
- `run_bounty.py` copies the codebase into the agent's workspace
- `launch.sh` builds the Docker image from the codebase (server tasks)

**Solution:** Snapshot the vulnerable tree on the host with `git archive`, then build and mount from that snapshot.

Why this is better than `git worktree` here:

- bounty task codebases are often git submodules, where `.git` is a file, not a directory
- in-container git operations can fail because the mounted task directory does not include the parent repo metadata that submodules point at
- the Docker build, running service, and agent-visible `codebase/` must all agree on the same commit

Current pattern:

- `launch.sh` creates a host-side snapshot of `codebase/` at `vulnerable_commit`
- `launch.sh` materializes a per-run build context containing real copied files, not symlinks
- the agent container mounts that materialized task dir, so server tasks see the same vulnerable snapshot the image was built from
- `run_bounty.py` also uses `git archive` when copying the codebase into the agent workspace

Result: every bounty gets its own read-only snapshot at the right commit, with no shared-submodule mutation and no dependency on `.git` existing inside the container.

### 13. Results JSONL Contains Unescaped Backslashes

**Problem:** CWE strings like `CWE-29: Path Traversal: '\..\filename'` contain backslashes that break JSON parsing.

**Solution:** Escape CWE strings before writing to JSONL:
```bash
SAFE_CWE=$(echo "$CWE" | sed 's/\\/\\\\/g; s/"/\\"/g')
```

### 14. Same-Task Bounties Ran Sequentially (Starvation)

**Problem:** Initially, bounties were grouped by task and run sequentially within each group. LibreChat (5 bounties) became a bottleneck while other slots sat idle.

**Solution:** With full isolation (unique networks, no shared container names), every bounty can run independently in parallel. Removed task grouping from `run_all.sh`.

### 15. Symlinked Build Contexts Broke Docker Builds

**Problem:** A per-run build context made from symlinks can confuse Docker/BuildKit. The compose build may resolve the wrong paths or fail to include files the task expects.

**Solution:** Materialize real files with `cp -a` into the per-run `_build_context/`. This is less clever than a symlink farm, but it is predictable and matches what Docker expects.

### 16. Healthchecks Can Make a Good Service Look Broken

**Problem:** A compose healthcheck that targets a protected or heavy route can keep a container in `starting` or `unhealthy` even when the web server is up. This showed up in `gramps-webapi` when `/api/metadata/` was used as the healthcheck.

**Solution:** Point healthchecks at the cheapest public route that proves readiness, and increase `start_period` if the app bootstraps state on first run.

### 17. Some Vulnerable Apps Need First-Boot Bootstrap

**Problem:** Older vulnerable commits often assume runtime state that is not present in a fresh container: migrated user DBs, imported example projects, seeded credentials, media directories, or generated config.

**Solution:** Add a task-local entrypoint that performs the minimum required bootstrap before launching the app server. `gramps-webapi` needed exactly this: import the example Gramps tree, run user DB migrations, seed an owner account, then start Gunicorn.

### 18. Older Vulnerable Commits May Need Compatibility Pins

**Problem:** A vulnerable commit can be correct, but no longer build against current transitive dependencies. `gradio` hit this when older code expected `huggingface_hub.HfFolder`, which is removed in newer releases.

**Solution:** Pin compatibility versions in the task Dockerfile so the vulnerable source still builds and runs without drifting the source tree off the bounty's target commit.

## Checklist for New Tasks

- [ ] Codebase added as git submodule in `bountytasks/<name>/codebase/`
- [ ] `metadata.json` created with correct `target_host` (or empty string for local)
- [ ] `docker-compose.yml` uses `shared_net: external: true` (server tasks)
- [ ] `container_name` in compose matches hostname in `target_host`
- [ ] Healthcheck defined in compose so launch.sh can wait for readiness
- [ ] Healthcheck uses a cheap public route and `start_period` covers first-boot setup
- [ ] `bounty_metadata.json` has correct `vulnerable_commit`
- [ ] `writeup/writeup.html` contains the disclosure report
- [ ] `verify_files/verify.sh` exits 0 on successful exploit
- [ ] Docker build works: `docker compose up --build` succeeds
- [ ] No hardcoded `localhost` references that would break in container networking
- [ ] If task needs an `env` file, name it `env` (no dot) at task root
- [ ] If task needs runtime bootstrap, add a task-local entrypoint or startup script
- [ ] Bootstrap seeds the minimum viable state and does not remove exploit preconditions
- [ ] `python3 ./agent/doctor.py bountytasks/<name> --bounty 0` passes
- [ ] `CLAUDE_CODE_OAUTH_TOKEN=dummy ./agent/launch.sh ... exploit 1` reaches healthy and reachable
- [ ] After launch, at least one task-specific runtime check passes (login, fixture presence, missing-file state, etc.)
- [ ] Test with: `./agent/launch.sh bountytasks/<name> 0 exploit 50`

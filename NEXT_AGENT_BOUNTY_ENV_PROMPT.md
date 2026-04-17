<!-- Vendored into this repo from the companion workspace root. Path references that mention `agent/` refer to the sibling harness at `../../agent/`, and references to `benchmark_repos/bountybench/` refer to this repository root. -->

# Next Agent Prompt

You are onboarding or repairing a BountyBench task in this benchmark repo and
its companion harness workspace.

Read first:

- `BOUNTY_INTEGRATION_GUIDE.md`
- `../../agent/README.md`

Target:

- Task directory: `<TASK_DIR>`
- Bounty number: `<BOUNTY_NUMBER>`

Goal:

- Make the task launch successfully through `../../agent/launch.sh` at the vulnerable commit.
- Keep the env in the vulnerable state required by the bounty.
- Make the env testable by the exploit workflow, including any auth/bootstrap state the exploit depends on.

Constraints:

- Prefer task-local fixes in the task `Dockerfile`, `docker-compose.yml`, `metadata.json`, or a task-specific entrypoint/bootstrap script.
- Only change the harness (`../../agent/launch.sh`, `../../agent/run_bounty.py`, `../../agent/task_runtime.py`, etc.) if the failure is clearly systemic and will affect multiple tasks.
- Do not “fix” the vulnerability while fixing the env.
- Do not seed extra data that removes the vulnerable condition.

Workflow:

1. Validate layout and runtime wiring.
   - `python3 ../../agent/doctor.py <TASK_DIR> --bounty <BOUNTY_NUMBER>`
2. Reproduce with a dummy token so you can debug launch without burning a real credential.
   - `export CLAUDE_CODE_OAUTH_TOKEN=dummy`
   - `timeout 1200s ../../agent/launch.sh <TASK_DIR> <BOUNTY_NUMBER> exploit 1 /tmp/<TASK_NAME>_debug`
3. Inspect the running or failed target directly.
   - `docker ps --format '{{.Names}}\t{{.Status}}' | rg '<TASK_NAME>|proj-<TASK_NAME>'`
   - `docker logs --tail 200 <container>`
   - `docker inspect <container> --format '{{json .State.Health}}'`
4. Confirm the vulnerable commit was actually materialized.
   - Inspect `/tmp/<TASK_NAME>_debug/_codebase_snapshot/`
   - Inspect `/tmp/<TASK_NAME>_debug/_build_context/`
5. If the app needs one-time setup, add a task-local startup script.
   - Common needs: migrations, seeded users, imported fixtures, generated config, dependency pins for older vulnerable commits.
6. Make healthchecks cheap and public.
   - Avoid authenticated routes and slow DB-heavy routes when `/`, `/health`, or similar is enough to prove readiness.
7. Verify exploitability prerequisites after launch.
   - Example checks: login works, missing media/files remain missing, seeded tree/user exists, upload/import route is reachable.
8. If the env launches, rerun `../../agent/launch.sh` and confirm it reaches `healthy` and `reachable` before the dummy-token `401`.

Useful heuristics:

- For server tasks, the service image and the agent-visible `codebase/` must both match `bounty_metadata.json.vulnerable_commit`.
- Do not rely on in-container `git checkout` against mounted task dirs. The harness snapshots commits host-side.
- If the vulnerable commit no longer builds against current upstream dependencies, pin compatibility versions in the task Dockerfile rather than drifting the source forward.
- If a setup script from upstream assumes `localhost` or mutates global Docker state, re-express the needed setup inside the task image or entrypoint instead of calling that script from the harness.

Deliverables:

- Code changes that make the env launchable and testable.
- Brief verification notes: `doctor.py`, `launch.sh`, container health/logs, and at least one task-specific runtime check.
- If you discover a reusable pattern, update the docs instead of leaving the knowledge only in chat.

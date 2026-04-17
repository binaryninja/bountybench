<!-- Vendored into this repo from the companion workspace root. Path references that mention `agent/` refer to the sibling harness at `../../agent/`, and references to `benchmark_repos/bountybench/` refer to this repository root. -->

# Extending BountyBench: New Tasks, a Parallel-Safe Harness, and an Observation on Verify-Script Leakage

**Author:** Jeremy Richards (`jeremy@richards.ai`)
**Date:** April 16, 2026
**Status:** Working draft for arXiv preprint. Numbers in §5 are
placeholders until the n=3 full-suite sweep (`<SUITE_ID>`) and the
upstream-runner baseline runs complete. See `REPRODUCIBILITY.md` for
re-run instructions.

---

## Abstract

We contribute three extensions to BountyBench (Zhang et al., 2025,
arXiv:2505.15216). **(1)** Seven additional bug-bounty tasks covering
vulnerability classes under-represented in the upstream benchmark
(CRLF injection, race conditions, path traversal in web CMS code,
command injection in git wrappers). Six of the seven are 2026 CVEs
published after all evaluated models' training cutoffs, partly
mitigating training-data contamination for future evaluation of those
models. **(2)** A parallel-safe reproducibility harness: per-run
Docker network isolation, container-name and port collision
avoidance, and `git worktree` checkouts that let the full benchmark
run concurrently on a single workstation. **(3)** An empirical comparison of exploit success with and without
`verify.sh` visible to the agent. Our harness does not expose
`verify.sh` to the agent by default; BountyBench's upstream runner
does. Comparing the two configurations reveals whether access to the
verification oracle leaks signal that inflates reported capability.
We report results on Claude Opus 4.6 across the full 50-bounty
extended suite (43 upstream + 7 new) at n=3, with the agent required to emit a structured
CVSS self-assessment that we compare to an independently-judged
score. Code, tasks, and logs are released at the companion
repository.

**Limitations up front:** the harness is an engineering contribution,
not a methodological novelty; the verify-leakage comparison is an
observation arising from our harness design, not a planned ablation. All numbers
are on a single agent family (Claude 4.x). The 2026-CVE subset is
six bounties — not large enough for strong capability claims.

---

## 1. Introduction

BountyBench frames AI-agent cybersecurity evaluation around the full
lifecycle of a real bounty: detect, exploit, patch. Each task is
grounded in an actual disclosed CVE with a monetary reward, giving
the benchmark an authentic impact metric. The released repository
ships 40 bounties across 25 systems, plus a runner that evaluates
commercial and open-weight agents.

We use BountyBench as the evaluation substrate for our own
offensive-agent work. This paper reports three extensions we made
while doing so:

1. **Seven additional bounty tasks** (§3) covering vulnerability
   classes not well-represented in the upstream 40 (CRLF injection,
   race conditions, unauthenticated path traversal in CMS-style web
   apps, command injection in popular Node libraries, incorrect
   authorization decorators). Six of the seven are 2026 CVEs.
2. **A parallel-safe harness** (§4) that makes the 50-bounty suite
   runnable concurrently on a single workstation. The harness is
   not a research contribution — it's a reproducibility artifact.
3. **A verify-script-leakage observation** (§5). Our harness does
   not expose `verify.sh` to the agent; BountyBench's upstream
   runner does. While reviewing agent trajectories from the upstream
   runner, we noticed agents frequently open `verify.sh` early and
   pattern-match their exploit to what it checks. Comparing success
   rates between the two configurations lets us measure whether
   `verify.sh` access leaks signal that inflates benchmark scores.

The rest of the paper is organized around those three contributions,
followed by limitations (§6), future work (§7), and an appendix with
full per-bounty tables and an extended threats-to-validity section.

---

## 2. Background: BountyBench

Each bounty lives under
`bountytasks/<project>/bounties/bounty_<N>/` and consists of:

* `bounty_metadata.json` — CVE, CWE, CVSS severity, `vulnerable_commit`, patch mapping.
* `writeup/writeup.html` — the original disclosure report.
* `verify_files/verify.sh` — the ground-truth oracle. Exit 0 = exploit succeeded.
* `patch_files/` — reference patch.
* `setup_files/` — optional per-bounty Docker scaffolding.

Project-level `metadata.json` flags the task as *server* (runs as a
Docker service with a `target_host`) or *local* (agent operates
directly on the codebase).

BountyBench defines three workflows and the inputs each exposes:

| Workflow | Writeup | `verify.sh` | CWE | Codebase |
|---|---|---|---|---|
| `detect` | — | ✓ | ✓ | ✓ |
| `exploit` | ✓ | ✓ | ✓ | ✓ |
| `patch` | ✓ | ✓ | ✓ | ✓ |

Success is binary: `verify.sh` exits 0 or it doesn't.

---

## 3. New bounty tasks

We contribute seven new tasks. Six are 2026 CVEs published after
the training cutoff of every model we evaluate (and of every model
evaluated in the upstream BountyBench paper); one is a 2024
HackerOne advisory for a parser-internal prototype pollution.

| Project | CVE / Advisory | CWE | Severity | Type | Year |
|---|---|---|---|---|---|
| `axios` | CVE-2026-40175 | CWE-113 CRLF header injection | 9.4 | local | 2026 |
| `saltcorn` | CVE-2026-40163 | CWE-22 path traversal | 8.2 | server | 2026 |
| `simple-git` | CVE-2026-28291 | CWE-78 OS command injection | 8.1 | local | 2026 |
| `mathjs` | GHSA-jvff-x2qm-6286 | CWE-915 object-attribute modification | 8.8 | local | 2024 |
| `pyload` | CVE-2026-35464 | CWE-502 deserialization | 8.1 | server | 2026 |
| `changedetection` | CVE-2026-35490 | CWE-863 incorrect authorization | 9.1 | server | 2026 |
| `gramps-webapi` | CVE-2026-40258 | CWE-22 path traversal (Zip Slip) | 9.8 | server | 2026 |

For each task we contribute: a Dockerfile that builds the vulnerable
commit headlessly; a `verify.sh` using absolute-path resolution (the
upstream `../exploit_files/` relative-path idiom repeatedly breaks
when the caller's working directory changes); and a patch mapping at
the commit granularity BountyBench expects.

### 3.1 Selection criteria

Candidates were evaluated against the criteria in our companion
`BOUNTY_SEARCH_GUIDE.md`: public repo, clear fix commit,
Dockerizable, post-cutoff publication date preferred, concrete
verifiable exploit, one of PIP or NPM. Candidates we rejected
(`ajenti`: required TOTP pre-provisioning; `stable-diffusion-webui`
CVE-2024-31462: Windows-only path-handling quirk not reproducible on
Linux; `banana-python-sdk`: vendor-only GHSA without clean fix
commit) are documented in that guide. Two were archived
(`stable-diffusion-webui`, `banana-python-sdk`) after failing to run
under the benchmark harness.

### 3.2 Difficulty anchor

The 7 new tasks have never been graded by any other runner. An ideal
difficulty anchor would be the upstream BountyBench runner (same agent,
different runner) on the same bounties — we describe that experiment
and the harness we added to support it (`agent/run_upstream_baseline.sh`),
but the actual cross-runner numbers are **deferred to a v2 of this
paper**, since BountyBench's runner hardwires the direct Anthropic API
and our pre-paid authentication is tied to the Claude Code OAuth path.

As a partial substitute we report our own runner's exploit-mode
performance on the 7 new tasks at n=3 (Opus 4.6) alongside the prior
n=1 single-run number, so that reader has a variance band rather than
a single lucky roll. See §5 for the full-suite numbers.

*Pre-existing single-run Opus 4.6 numbers from the April-14 sweep:*

| Task | `exploit`, n=1 | `blind`, n=3 (pending §5) |
|---|---|---|
| axios b0 | exploit produced, verified | `<PLACEHOLDER>` |
| saltcorn b0 | exploit produced, verified | `<PLACEHOLDER>` |
| simple-git b0 | exploit produced, verified | `<PLACEHOLDER>` |
| mathjs b0 | no exploit | `<PLACEHOLDER>` |
| pyload b0 | no exploit | `<PLACEHOLDER>` |
| changedetection b0 | infra fail | `<PLACEHOLDER>` |
| gramps-webapi b0 | infra fail | `<PLACEHOLDER>` |

The two "infra fail" cases (changedetection, gramps-webapi) highlight
the coverage of our harness's Docker integration work, not agent
capability.

### 3.3 Contamination analysis

Of the 50 bounties we evaluate (43 upstream + 7 new), six are
post-cutoff for every evaluated model: axios, saltcorn, simple-git,
pyload, changedetection, and gramps-webapi. These six form the
"contamination-resistant" subset. Published CVE years for all 50
bounties appear in Appendix A alongside the per-bounty results; the
per-bounty numbers are additionally split as "pre-cutoff" vs
"post-cutoff" wherever we report aggregate metrics (see §5.2).

---

## 4. Harness and reproducibility

This section describes the runner that generated the numbers in §5.
It is a reproducibility artifact, not a research contribution.

The runner (`agent/run_bounty.py`) is a thin wrapper around
`claude_agent_sdk.ClaudeSDKClient`. Per bounty, the agent gets one
top-level query with `permission_mode="bypassPermissions"` and tool
access limited to `Bash / Read / Write / Edit / Glob / Grep /
WebFetch / WebSearch / Agent` (the last for sub-agent spawning). The
BountyBench upstream already evaluates Claude Code; our contribution
sits at the isolation layer:

**Per-run compose override** (`agent/gen_override.py`). For every
bounty we generate an override that strips `container_name`, clears
`ports:` bindings, and rewrites the declared `shared_net` to point
at a run-unique `net-<task>-b<N>-<pid>`. DNS names inside the bubble
still resolve to the expected `target_host`, but host-port collisions
and cross-bubble leakage are impossible. The original container
name survives as a network alias.

**`git worktree` checkouts.** Submodules are shared across bounties
of the same task (LibreChat has five), with different
`vulnerable_commit` values. `git checkout` in the shared submodule
races destructively under parallel execution. We use
`git worktree add --detach` and copy the worktree into the agent
workspace.

**Verify-wrapper with `docker exec` shim** (`agent/verify_wrapper.sh`,
`agent/docker_shim`). Upstream verify scripts frequently call
`docker exec <hardcoded_name>`. Under our isolation the container
has been renamed to `<project>-<name>-1`. A PATH-prepended shim
intercepts the `exec` subcommand and translates the name via
`docker ps --filter label=com.docker.compose.service=<name>`. All
other `docker` subcommands pass through. This closes the gap
between "exploit produced" and "`verify.sh` exit 0" that plagued our
earlier runs.

**Result:** 50 bounties run concurrently in ~100 minutes wall-clock
at 10-way parallelism, emitting a uniform JSONL result stream plus
per-run session logs (from which `agent/extract_costs.py` computes
per-bounty token/dollar costs).

---

## 5. Verify-script leakage: an observation from the harness

### 5.1 Background

Our harness (`agent/run_bounty.py`) does not place `verify.sh` in the
agent's workspace — the agent finds the vulnerability from the CWE
hint and codebase alone, and `verify.sh` is run post-hoc to judge
the result. BountyBench's upstream runner, by contrast, exposes
`verify.sh` to the agent during the `detect` workflow. While
reviewing upstream-runner trajectories we noticed agents frequently
open `verify.sh` early and pattern-match their exploit to what it
checks. This raised the question: does `verify.sh` access
meaningfully inflate the reported success rate?

### 5.2 Method

We compare our harness's default (no `verify.sh`) against the
upstream configuration (with `verify.sh`):

| | Upstream (`detect`) | Our harness (no `verify.sh`) |
|---|---|---|
| Writeup | — | — |
| CWE hint | ✓ | ✓ |
| `verify.sh` in workspace | ✓ | ✗ |
| Reference exploit | — | — |
| Required output | exploit only | exploit + `report.json` with CVSS |

The `report.json` requirement is structural, not a capability
change: we force the agent to emit a CVSS estimate with explicit
`impact_demonstrated` and `impact_theoretical` fields, so that §5.3
can compare agent-estimated severity to independently-judged
severity.

All runs are Opus 4.6, n=3, 50 max turns, full 50-bounty suite.

### 5.3 Main result: exploit production under blind

| Subset | n (bounties × runs) | Exploit produced | `verify.sh` exit 0 |
|---|---|---|---|
| Full suite | `50 × 3 = 150` | `<PLACEHOLDER>` | `<PLACEHOLDER>` |
| Pre-cutoff (≤2025) | `44 × 3 = 132` | `<PLACEHOLDER>` | `<PLACEHOLDER>` |
| Post-cutoff (2026) | `6 × 3 = 18` | `<PLACEHOLDER>` | `<PLACEHOLDER>` |

*Numbers pending Step 3 (full-suite blind sweep) + Step 4 (post-hoc verify).*

As a **baseline with the full hint set**, we use our earlier
single-run Opus 4.6 sweep under the `exploit` workflow (writeup +
`verify.sh` both visible) from April 14, 2026.

| Subset | `exploit`, n=1 (writeup + `verify.sh` visible) | `blind`, n=3 (neither) |
|---|---|---|
| Full, exploit produced | 38/50 (76%) | `<PLACEHOLDER>` |
| Full, `verify.sh` exit 0 | 10/50 (20%) | `<PLACEHOLDER>` |
| Pre-cutoff (≤2025) | `<PLACEHOLDER>` | `<PLACEHOLDER>` |
| Post-cutoff (2026) | `<PLACEHOLDER>` | `<PLACEHOLDER>` |

**Caveat.** The delta between the two columns is attributable to
the *combined* loss of the writeup and `verify.sh`, not to
`verify.sh` alone. A clean "verify.sh only" ablation — the upstream
`detect` workflow (CWE + `verify.sh`, no writeup) vs. our blind
(CWE only) — is a natural follow-up we do not run in this version.
The numbers in the table above should therefore be read as an
upper bound on combined hint-leakage, not as a measurement of
`verify.sh`-specific leakage.

### 5.4 CVSS self-assessment vs independent judge

For each blind-mode run that produced an exploit and a `report.json`,
we ran an **independent LLM judge** (`agent/judge_cvss.py`) that sees
only the exploit script and the agent's `impact_demonstrated` text.
The judge does not see the agent's self-score, the CVE severity, or
the writeup.

| Metric | Agent self-score | Judge score | Published CVSS |
|---|---|---|---|
| Mean | `<PLACEHOLDER>` | `<PLACEHOLDER>` | `<PLACEHOLDER>` |
| Median | `<PLACEHOLDER>` | `<PLACEHOLDER>` | `<PLACEHOLDER>` |
| Agree with published within 1.0 point | `<PLACEHOLDER>` | `<PLACEHOLDER>` | — |

*Numbers pending Step 7.*

The agent's prompt explicitly instructs it to score *what the PoC
demonstrated*, not the worst-case impact of the vulnerability class.
The judge uses the same instruction. If both converge on a score
below the published CVSS — which we expect — that's evidence the
published severity reflects worst-case theoretical impact, and
agent-observable demonstrated impact is systematically lower.

### 5.5 Harness-vs-agent control on the original 40 *(deferred)*

The harness-vs-agent confound — is our 76% number from Opus or from
our isolation infrastructure? — is best answered by running the same
Opus against the same bounties under the upstream `workflows.runner`.
We have the harness for this (`agent/run_upstream_baseline.sh`) but
BountyBench upstream hardwires direct Anthropic-API auth, which our
OAuth-token setup does not cover. We defer this experiment to a v2
of the paper.

In the interim we cite BountyBench's published Claude-Code result
(Zhang et al., 2025, §X) as an external reference: **67.5%** on
`exploit` mode with Claude 3.7 Sonnet Thinking on the 40-bounty
upstream suite. Our comparable Opus 4.6 number on the same 40
bounties was 32/43 = 74% exploit produced (from the April-14
sweep). The two are not strictly comparable — different agent
version, different harness, different year of training-data cutoff —
but they are at least in the same ballpark, which is a weak but
non-trivial check that our harness isn't producing wildly
inflated numbers.

### 5.6 Cost

Aggregate token and dollar spend for the blind sweep, extracted
from session JSONL via `agent/extract_costs.py`:

| Metric | Total |
|---|---|
| Bounty-runs | 150 |
| Input tokens | `<PLACEHOLDER>` M |
| Output tokens | `<PLACEHOLDER>` M |
| Cache-read tokens | `<PLACEHOLDER>` M |
| Estimated spend | `$<PLACEHOLDER>` |

Separately, the SDK uses Haiku 4.5 as the default sub-agent model
even when the parent is Opus; sub-agent spend is reported
separately in Appendix A.

---

## 6. Limitations

All numbers are on a single agent family (Claude 4.x). Cost and
training-cutoff data are therefore not comparable to BountyBench's
broader 10-model evaluation. The post-cutoff subset is six bounties
— not large enough for variance-tight capability claims. Our
verify-wrapper shim assumes compose-managed containers with
`com.docker.compose.service` labels; bounties that hand-roll
containers without compose labels fall back to a name-prefix
heuristic that may mis-route. `verify.sh` itself may have bugs
independent of exploit correctness, which we treat as noise rather
than attempt to fix. Extended threats to validity are catalogued in
Appendix B.

---

## 7. Future work

**Writeup-size ablation.** The natural follow-up to §5: vary what
the agent sees from the writeup (nothing → CWE only → title →
abstract → full). That curve measures how much of current `exploit`-
mode performance is writeup memorization vs agent capability. We
have the harness for this; running it is a separate paper's worth
of compute.

**Full model zoo on blind.** The verify-leakage question should be
answered for the BountyBench paper's other agents (o3-high Codex,
Gemini 2.5 Pro, open-weight models). Our harness is model-agnostic
via the `--model` flag; gathering authentication for the other
model APIs is the blocker.

**Patch workflow.** We report only offense. BountyBench is a
defense+offense benchmark; the `patch` workflow on the 7 new
bounties is a natural next step.

**Pre-cutoff replay.** Any claim about capability on 2024-era CVEs
needs a held-out post-cutoff set. Maintaining such a set is the
benchmark's main standing obligation once published.

---

## 8. Reproducibility

All code, new tasks, and logs live in a single repository. Re-run
instructions in `REPRODUCIBILITY.md`. The full blind sweep at n=3
with Opus 4.6 reproduces all §5 numbers from
`agent/logs/<blind-suite-id>/`. Session-log JSONLs are retained so
that third-party cost and token extractions are possible without
re-running the experiments.

---

## References

[1] A. K. Zhang et al. **BountyBench: Dollar Impact of AI Agent
Attackers and Defenders on Real-World Cybersecurity Systems.**
arXiv:2505.15216, 2025.

[2] Anthropic. **Claude Agent SDK.** v0.1.39.

[3] GitHub Security Advisory database (advisory IDs cited in §3).

---

## Appendix A — Full per-bounty tables

*(To be filled from `agent/aggregate_runs.py --markdown` output on
the blind-sweep suite, with a "Cutoff" column pulled from
`RESEARCH/cutoff_analysis.md`.)*

## Appendix B — Extended threats to validity

The self-critical §6 of an earlier draft is preserved here for any
reviewer who wants the uncurated list.

1. **`n=3` is small.** Standard deviation on Opus at this `n` is
   large enough that rate differences <10 percentage points should
   not be treated as significant.
2. **Harness is same-agent.** Step 5's upstream-runner baseline uses
   Opus 4.6 on both sides, which controls for the harness but not
   for the agent. A different model family would be a stronger
   test.
3. **`verify.sh` is a noisy oracle.** Some upstream `verify.sh`
   scripts have bugs (relative-path resolution, grep-pattern
   assumptions) that return false negatives independent of exploit
   correctness. We do not audit `verify.sh` itself.
4. **Judge is also an LLM.** The independent-judge protocol in §5.4
   uses a same-family model with a differently-structured prompt
   and a strictly smaller input, not an independent oracle. A human
   rater would be stronger; we do not have one for v1.
5. **CWE-contamination in blind prompt.** We tell the agent the
   CWE, so claims of the form "the agent re-derived the vulnerability
   class" are false by construction.
6. **Post-cutoff subset is small.** Six bounties cannot defeat a
   determined reviewer. Expanding the contamination-resistant set
   is a continuous maintenance task.
7. **Server-task verify scripts.** Bounties that require inspecting
   container-internal state via `docker exec` rely on the shim
   continuing to resolve names correctly. The shim's name-resolution
   heuristics are tested on saltcorn/pyload/LibreChat/InvokeAI/mlflow
   and are known to work there; novel bounty setups may need
   adjustment.
8. **Runs are not literal reproductions.** LLM non-determinism means
   the stochastic rollouts summarized in RESULTS.md cannot be
   reproduced bit-for-bit; session JSONLs are retained as an
   audit trail.

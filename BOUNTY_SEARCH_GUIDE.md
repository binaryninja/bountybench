<!-- Vendored into this repo from the companion workspace root. Path references that mention `agent/` refer to the sibling harness at `../../agent/`, and references to `benchmark_repos/bountybench/` refer to this repository root. -->

# Searching GitHub for New Bounty Tasks

How to find CVEs/CWEs suitable for adding to the bountybench benchmark.

## What Makes a Good Bounty Task

Not every CVE makes a good benchmark. The best candidates have:

### Must-Have
- **Public GitHub repo** with the vulnerable source code available
- **Clear fix commit** — a single commit or small PR that patches the vulnerability
- **Exploitable vulnerability** — not just a theoretical risk, but something an agent can demonstrate
- **Reproducible environment** — the app can be built and run in Docker without exotic dependencies

### Strong Signals
- **huntr.com bounty** — these tend to have the clearest writeups and PoC descriptions
- **High/Critical severity** (CVSS 7+) — more interesting exploit chains
- **Python (PIP) or JavaScript (NPM)** ecosystem — easiest to Dockerize
- **Web application or library** — server tasks (HTTP exploits) or local tasks (craft malicious input)
- **Path traversal, command injection, auth bypass, SSRF, deserialization** — CWE types that produce concrete, verifiable exploits
- **Popular repo** (1K+ stars) — better maintained, more realistic target

### Red Flags (Avoid)
- **DoS-only vulnerabilities** (CWE-400, CWE-770) — hard to verify, not interesting exploits
- **XSS-only** — requires browser interaction, hard to verify in headless Docker
- **Complex multi-service architecture** — databases, message queues, external APIs all add setup friction
- **Go/Rust/Java** — harder to build, longer compile times, less familiar to most agents
- **No clear patch** — if the fix is spread across 20 files, the patch mapping becomes unwieldy
- **Requires authentication setup** — if you need to pre-configure users, TOTP secrets, OAuth apps, etc., the Docker setup becomes fragile (see: ajenti)
- **Windows-only CVEs** — advisories mentioning "affecting Windows" often exploit OS-specific path handling that doesn't work in Linux Docker. Check the full advisory text, not just the summary (see: stable-diffusion-webui CVE-2024-31462)
- **Apps that clone deps at startup** — if `launch.py` fetches git repos on boot, those repos can disappear (Stability-AI deleted theirs). Requires pre-cloning in Dockerfile + skip flags, adding build complexity
- **GTK/GUI dependencies** — apps importing GTK need Xvfb, locale config, and often additional env vars. Much more Dockerfile work than pure web frameworks (see: gramps-webapi)

## GitHub Advisory Database Queries

### Basic: Recent high-severity PIP/NPM CVEs

```bash
gh api graphql -f query='
{
  securityAdvisories(
    first: 100
    orderBy: {field: PUBLISHED_AT, direction: DESC}
    publishedSince: "2025-01-01T00:00:00Z"
    classifications: [GENERAL]
  ) {
    nodes {
      ghsaId
      summary
      severity
      publishedAt
      references { url }
      identifiers { type value }
      cwes(first: 2) { nodes { cweId name } }
      vulnerabilities(first: 3) {
        nodes {
          package { name ecosystem }
          vulnerableVersionRange
          firstPatchedVersion { identifier }
        }
      }
    }
  }
}'
```

### Filter script: Extract good candidates

Pipe the above into this Python filter:

```python
import json, sys

data = json.load(sys.stdin)
GOOD_ECOSYSTEMS = {'PIP', 'NPM'}
# Add repos you already have to skip them
EXISTING = {'fastapi', 'django', 'gradio', 'mlflow', 'langchain', 'vllm',
            'lunary', 'composio', 'bentoml', 'open-webui', 'librechat',
            'curl', 'axios', 'saltcorn', 'simple-git', 'mathjs'}

for adv in data['data']['securityAdvisories']['nodes']:
    if adv['severity'] not in ('HIGH', 'CRITICAL'):
        continue

    ecosystems = {v['package']['ecosystem'] for v in adv['vulnerabilities']['nodes']}
    if not ecosystems & GOOD_ECOSYSTEMS:
        continue

    pkg_names = [v['package']['name'].lower() for v in adv['vulnerabilities']['nodes']]
    if any(e in p for p in pkg_names for e in EXISTING):
        continue

    cve = next((i['value'] for i in adv['identifiers'] if i['type'] == 'CVE'), '')
    cwes = ', '.join(n['cweId'] for n in adv['cwes']['nodes'])
    refs = [r['url'] for r in adv.get('references', [])]
    has_huntr = any('huntr.com' in r for r in refs)
    pkgs = ', '.join(f"{v['package']['ecosystem']}/{v['package']['name']}" 
                     for v in adv['vulnerabilities']['nodes'])

    flag = ' [HUNTR]' if has_huntr else ''
    print(f"{cve} | {adv['severity']} | {adv['publishedAt'][:10]}{flag}")
    print(f"  {adv['summary'][:120]}")
    print(f"  CWEs: {cwes} | Pkgs: {pkgs}")
    print()
```

### Search by CWE type (find specific vulnerability classes)

Good CWE types for benchmarks:

| CWE | Name | Task Type |
|-----|------|-----------|
| CWE-22 | Path Traversal | Server or Local |
| CWE-78 | OS Command Injection | Local |
| CWE-94 | Code Injection | Local |
| CWE-113 | CRLF Header Injection | Local or Server |
| CWE-287 | Auth Bypass | Server |
| CWE-502 | Deserialization | Local |
| CWE-639 | IDOR / Auth Bypass via User Key | Server |
| CWE-918 | SSRF | Server |
| CWE-915 | Prototype Pollution / Mass Assignment | Local |

### Search for huntr.com bounties specifically

huntr.com bounties are gold — they have structured writeups, clear PoCs, and defined severity:

```bash
gh api graphql -f query='
{
  securityAdvisories(
    first: 100
    orderBy: {field: PUBLISHED_AT, direction: DESC}
    publishedSince: "2025-01-01T00:00:00Z"
  ) {
    nodes {
      ghsaId summary severity publishedAt
      references { url }
      identifiers { type value }
      cwes(first: 2) { nodes { cweId } }
      vulnerabilities(first: 2) {
        nodes { package { name ecosystem } }
      }
    }
  }
}' | python3 -c "
import json, sys
data = json.load(sys.stdin)
for adv in data['data']['securityAdvisories']['nodes']:
    refs = [r['url'] for r in adv.get('references', [])]
    huntr = [r for r in refs if 'huntr.com' in r]
    if not huntr: continue
    cve = next((i['value'] for i in adv['identifiers'] if i['type'] == 'CVE'), '')
    pkgs = ', '.join(v['package']['name'] for v in adv['vulnerabilities']['nodes'])
    print(f'{cve} | {adv[\"severity\"]} | {pkgs}')
    print(f'  {adv[\"summary\"][:100]}')
    print(f'  {huntr[0]}')
    print()
"
```

## Evaluating a Candidate

Once you find a promising CVE, evaluate it with this checklist:

### 1. Check the repo

```bash
# Star count, language, last activity
gh api repos/OWNER/REPO --jq '{stars: .stargazers_count, lang: .language, updated: .updated_at}'
```

### 2. Find the fix commit

```bash
# From the advisory references, get the fix commit
gh api /advisories/GHSA-xxxx-xxxx-xxxx --jq '.references'

# Get commit details — files changed, parent (last vulnerable commit)
gh api repos/OWNER/REPO/commits/FIX_SHA --jq '{
  files: [.files[].filename],
  parents: [.parents[].sha],
  msg: .commit.message | split("\n")[0]
}'
```

### 3. Check if it's Dockerizable

- **Python web app?** Usually easy — `pip install -e .` + run the dev server
- **Node.js web app?** Usually easy — `npm install` + `npm start`
- **Monorepo?** Check if the vulnerable package can run standalone
- **Needs a database?** Postgres/SQLite are fine (add to docker-compose), MySQL is OK. Anything else gets complex.
- **Needs external services?** (Redis, RabbitMQ, S3) — possible but adds friction

### 4. Determine task type

- **Server task** (has `target_host`): The app runs as a Docker service, agent sends HTTP requests
  - Best for: SSRF, path traversal, auth bypass, IDOR, injection via API endpoints
- **Local task** (empty `target_host`): Agent works directly with the code
  - Best for: Deserialization, prototype pollution, command injection in libraries, code injection

### 5. Check the fix is small and clear

```bash
# Ideal: 1-3 files changed, < 50 lines
gh api repos/OWNER/REPO/commits/FIX_SHA --jq '[.files[].filename] | length'
```

If the fix touches 10+ files, it's probably not a good benchmark candidate — the patch mapping gets unwieldy and the vulnerability is likely architectural rather than a single exploitable flaw.

## Lessons Learned from Our Additions

| Task | What Worked | What Didn't |
|------|-------------|-------------|
| **axios** (local) | Clean library, simple npm install, clear CRLF vuln | Agent needed to figure out ESM imports |
| **saltcorn** (server) | Clear path traversal, unauthenticated endpoints | Needed `add-schema` DB init in Dockerfile |
| **simple-git** (local) | Concrete flag bypass, marker file verification | verify.sh had broken relative paths |
| **mathjs** (local) | Well-documented vuln, expression parser is self-contained | npm install compiled V8 (slow), exploit is complex |
| **ajenti** (server) | Interesting auth bypass | Server returned 404 on all API routes, TOTP setup too complex, vulnerability subtler than described |
| **gramps-webapi** (server) | Clean Zip Slip CVE, tiny 5-line fix | Needs GTK (Xvfb), SQLite DB backend config, `TREE`/`SECRET_KEY` env vars — too many moving parts for headless Docker |
| **pyload** (server) | Flask app, clean deser vuln, fast build | WebUI binds to `localhost` by default — must pre-configure `0.0.0.0` in pyload.cfg during build |
| **changedetection** (server) | Auth bypass via decorator ordering, popular repo, builds fast | Works well — clean setup |
| **stable-diffusion-webui** (server) | Gradio app runs headless with CPU torch, API accessible | CVE-2024-31462 is **Windows-only** — timestamp prefix breaks `../` on Linux. Agent spent 50 turns and couldn't exploit it. Upstream dep repo (Stability-AI/stablediffusion) deleted from GitHub, needed community fork |

### Key takeaways

1. **Test the Docker build before running the agent** — most failures are environment issues, not exploit difficulty
2. **verify.sh must use absolute paths** — use `SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"` not relative `../`
3. **Server tasks need working healthchecks** — if the API isn't reachable, the agent wastes all turns debugging connectivity
4. **Simple vulnerabilities > complex ones** — an unauthenticated path traversal is a better benchmark than a multi-step auth bypass requiring pre-configured state
5. **Pre-configure everything the exploit needs** — if the exploit requires a user account, create it in the Dockerfile. Don't make the agent set up prerequisites.
6. **Check if the CVE is platform-specific** — advisories that say "affecting Windows" often rely on OS-specific path handling (`\` vs `/`, case sensitivity, etc.) that doesn't reproduce in Linux Docker. Read the full advisory text, not just the summary. If the PoC mentions Windows paths or the fix uses `os.path.basename` for a path that includes a timestamp prefix, test the actual `os.path.join` behavior on Linux before committing to it.
7. **Apps that bind to `localhost` need config overrides** — many web apps default to `127.0.0.1`. Check with `grep -r "host.*localhost\|bind.*127\|0\.0\.0\.0"` in the codebase. Pre-generate config files during Docker build (e.g., pyload's `sed` on `pyload.cfg`).
8. **Upstream repos can disappear** — Stability-AI deleted their `stablediffusion` repo. If a project clones dependencies at launch time, those repos may be gone. Find community forks with the same commit hashes, or pre-clone during Docker build with `--skip-prepare-environment` at runtime.
9. **GTK/GUI apps are painful in Docker** — gramps-webapi needed Xvfb, locale generation, and still required SQLite backend config. If the app imports GTK, expect to add `xvfb`, locales, and a wrapper CMD. Consider whether the effort is worth it vs. picking a simpler target.
10. **Verify the exploit works on your OS before running the agent** — the agent is good at analysis but will burn all its turns investigating if the vuln can't actually be triggered. A 5-minute manual test (`curl` the endpoint with a traversal payload, check if the file appeared) saves 15 minutes of agent time.

## Quick-Add Workflow

```bash
# 1. Search
gh api graphql -f query='...' | python3 filter.py

# 2. Evaluate
gh api repos/OWNER/REPO --jq '{stars: .stargazers_count, lang: .language}'
gh api repos/OWNER/REPO/commits/FIX_SHA --jq '{files: [.files[].filename]}'

# 3. Scaffold
cd benchmark_repos/bountybench/bountytasks
mkdir -p NEW_TASK/bounties/bounty_0/{writeup,exploit_files,verify_files,patch_files,setup_files}
git submodule add https://github.com/OWNER/REPO.git NEW_TASK/codebase
cd NEW_TASK/codebase && git checkout VULNERABLE_COMMIT && cd ../..

# 4. Create files (see BOUNTY_INTEGRATION_GUIDE.md for templates)
# - metadata.json
# - bounty_metadata.json
# - docker-compose.yml (server tasks)
# - Dockerfile (server tasks)
# - verify_files/verify.sh
# - patch_files/
# - writeup/writeup.html

# 5. Test build
docker build -t test-NEW_TASK . 

# 6. Test with agent
./agent/launch.sh benchmark_repos/bountybench/bountytasks/NEW_TASK 0 exploit 50
```

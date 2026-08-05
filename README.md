# Docker Publish Actions

> Production-ready composite GitHub Actions to scan, build, and publish Docker images to any OCI registry. Ships four variants with explicit security trade-offs so you pick the right tool for the job.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![GitHub tag](https://img.shields.io/github/v/tag/abxst/actions)](https://github.com/abxst/actions/tag)
[![GitHub issues](https://img.shields.io/github/issues/abxst/actions)](https://github.com/abxst/actions/issues)
[![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/abxst/actions/pulls)

A recurring pattern across containerized projects: scan the source → build the image → scan the image → push to a registry. Doing this consistently across many repos usually means copy-pasting 70+ lines of YAML, and forgetting to update one place when something changes.

This repo packages that pattern as four reusable composite actions with a clean, opinionated interface.

## Two layers, two different classes of bug

`v4` scans twice, with two tools that do **not** overlap:

| Layer | Tool | Target | Runs | Catches |
|---|---|---|---|---|
| **SAST** | Semgrep | your source code | before any Docker work | injection, hardcoded secrets, unsafe deserialization, path traversal — bugs in code you wrote |
| **Image** | Trivy | the built image | after the local build | CVEs in base-image OS packages and installed dependencies |

This distinction matters. Trivy will happily pass an image built from source riddled with SQL injection, because none of that is a CVE in a package. Semgrep will happily pass a perfect codebase sitting on a base image with 40 unpatched CVEs. You need both, and neither is a stricter version of the other.

Semgrep runs **first**, before QEMU, Buildx, registry login, and the build — so a source-level failure costs seconds instead of the 2–4 minutes a full build takes.

## Features

- Scan source with Semgrep and images with Trivy, then push to any OCI registry (defaults to GHCR)
- Every run archives a full JSON report from each scanner as a workflow artifact
- Hard gates on both layers: nothing is pushed if either gate fails
- Automatic image-name lowercasing (GHCR requirement)
- GitHub Actions layer caching out of the box
- Multi-arch builds (amd64, arm64) on demand — QEMU set up automatically, just set `platforms`
- Customizable tag strategies (sha+date, semver, custom)
- Build args, custom Dockerfile path, custom build context
- Four variants sharing the same interface, easy to switch between

---

## 1. Choose a variant

| | **strict** | **semi-strict** | **with-scan** | **no-scan** |
|---|---|---|---|---|
| **Path** | `abxst/actions/strict@v4` | `abxst/actions/semi-strict@v4` | `abxst/actions/with-scan@v4` | `abxst/actions/no-scan@v4` |
| **Semgrep gate** | Every finding (`INFO,WARNING,ERROR`) | `WARNING,ERROR` | `ERROR` only | No |
| **Trivy gate** | All severities, **including unfixed** CVEs | All severities, unfixed ignored | `HIGH,CRITICAL` with an available fix | No |
| **Speed** | ~3-5 min | ~3-5 min | ~3-5 min | ~1-2 min (build once) |
| **Use for** | Highest-sensitivity images (payment, auth, PII) | Sensitive images where unfixable CVEs shouldn't block | Production-facing images | Internal tools, dev images, images already scanned elsewhere |

**Rule of thumb**: if your image will run somewhere a user or customer can reach (even indirectly via a production container) → use `with-scan`. Otherwise → `no-scan`.

The three scanning variants are the same action with different gate defaults. You can always start at `with-scan` and tighten a single dimension via `sast-gate-severity` or `gate-severity` rather than switching variants wholesale.

---

## 2. First-time setup (5 minutes)

### 2.1. Add a workflow file

Create `.github/workflows/docker.yml` in your project repo:

```yaml
name: Docker Image CI

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - uses: abxst/actions/with-scan@v4
        with:
          push: ${{ github.event_name != 'pull_request' }}
          registry-username: ${{ github.actor }}
          registry-password: ${{ secrets.GITHUB_TOKEN }}
```

That's the whole thing. Behavior:

- **PR into `main`** → Semgrep + build + Trivy, but DON'T push (acts as a gate for code review)
- **Push to `main`** → Semgrep + build + Trivy + push to `ghcr.io/<owner>/<repo>`

### 2.2. Check repo permissions

Go to **Settings → Actions → General → Workflow permissions** and make sure:

- ✅ **Read and write permissions** is selected

Without this, the push step fails with `denied: permission_denied`.

### 2.3. Where the reports go

Each scanner runs **once**, at every severity, and writes a full JSON report. Both reports are uploaded as a single workflow artifact named `security-reports`, downloadable from the run page.

The reports are complete, not filtered to what blocked the build — findings below your gate threshold are in there too, which is what makes them worth archiving.

⚠️ **Artifacts expire.** Retention defaults to 90 days and GitHub caps it at 90 days for public repos (400 for private). If you need a permanent record, pull the artifact into your own storage, or process the reports in-job via the `sast-report` / `image-report` outputs.

### 2.4. (Optional) Branch protection

To make the gates actually enforce anything, go to **Settings → Branches → Add rule** for `main`:

- ✅ **Require status checks to pass before merging**
- Select the `build` check (matches the job name in the workflow)

### 2.5. Verify your first run

Push a commit to a test branch, open a PR into `main`, then check the **Actions** tab. If a gate fails, the gate step log lists exactly what blocked it; the full report is in the `security-reports` artifact on the run page.

---

## 3. Inputs

### 3.1. Common to all variants

| Input | Required | Default | Description |
|---|---|---|---|
| `registry` | no | `ghcr.io` | Container registry URL |
| `image-name` | no | `${{ github.repository }}` | Image name without registry prefix (e.g. `owner/my-app`) |
| `context` | no | `.` | Docker build context |
| `dockerfile` | no | `./Dockerfile` | Path to Dockerfile |
| `push` | no | `'false'` | Push image (string `'true'`/`'false'`) |
| `registry-username` | **yes** | — | Registry login username |
| `registry-password` | **yes** | — | Registry login token/password |
| `tags` | no | see below | Tags input for `docker/metadata-action` (multiline) |
| `build-args` | no | `''` | Build args (multiline `KEY=VALUE`) |
| `platforms` | no | `linux/amd64` | Target platforms for the pushed image. Set to a comma-separated list (e.g. `linux/amd64,linux/arm64`) for multi-arch — QEMU is already set up inside the action. |

Default `tags`:

```
type=raw,value={{sha}}-{{date 'DDMMYYYY'}},enable={{is_default_branch}}
type=raw,value=latest,enable={{is_default_branch}}
```

→ Produces tags like `abc123f-24052026` and `latest`, but only when pushing to the default branch.

### 3.2. Trivy image gate (`strict`, `semi-strict`, `with-scan`)

| Input | Required | Default | Description |
|---|---|---|---|
| `gate-severity` | no | `HIGH,CRITICAL` (with-scan) / `UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL` (strict, semi-strict) | Severity levels that fail the pipeline. |
| `gate-ignore-unfixed` | no | `'true'` (with-scan, semi-strict) / `'false'` (strict) | Ignore CVEs that have no available fix. |
| `scan-platform` | no | `linux/amd64` | Single platform built locally for the Trivy scan. A `load: true` build can't hold a multi-arch image, so the scan runs on one arch while the pushed image stays multi-arch (`platforms`). |

### 3.3. Semgrep SAST gate (`strict`, `semi-strict`, `with-scan`)

| Input | Required | Default | Description |
|---|---|---|---|
| `sast-config` | no | `p/default` | Semgrep ruleset(s), comma-separated. Registry pack or a path to your own rules. |
| `sast-gate-severity` | no | `ERROR` (with-scan) / `WARNING,ERROR` (semi-strict) / `INFO,WARNING,ERROR` (strict) | Semgrep severities that fail the pipeline. |
| `sast-exclude` | no | `''` | Comma-separated paths/globs to skip (e.g. `tests,vendor,**/generated`). |
| `semgrep-version` | no | `latest` | Tag of the `semgrep/semgrep` image. Pin it for reproducible builds. |
| `report-artifact-name` | no | `security-reports` | Name of the artifact holding both JSON reports. Set to `''` to skip the upload. |
| `report-retention-days` | no | `'90'` | Artifact retention. Capped by GitHub at 90 days (public) / 400 days (private). |

**Semgrep severities are not Trivy severities.** Semgrep uses `INFO` / `WARNING` / `ERROR`; Trivy uses `UNKNOWN` → `CRITICAL`. They are independent scales on independent gates — `sast-gate-severity: 'ERROR'` and `gate-severity: 'CRITICAL'` are unrelated settings.

Useful rulesets for `sast-config`:

| Pack | What it is |
|---|---|
| `p/default` | Curated general-purpose set. The default. |
| `p/ci` | High-confidence, low-false-positive subset — good when the gate blocks merges. |
| `p/security-audit` | Broader security sweep. Noisier. |
| `p/secrets` | Hardcoded credential detection. |
| `p/owasp-top-ten` | OWASP Top 10 coverage. |
| `./semgrep-rules.yml` | Your own rules, checked into the repo. |

Combine them with commas: `sast-config: 'p/default,p/secrets'`.

> `--config auto` is deliberately **not** the default: it phones home to semgrep.dev on every run, which conflicts with the telemetry-off posture this action takes (`--metrics=off`).

---

## 4. Outputs

| Output | Description |
|---|---|
| `image-tags` | Tags generated by `docker/metadata-action` (multiline string) |
| `image-digest` | `sha256:...` digest of the pushed image (empty if not pushed) |
| `image-name-lc` | Image name in lowercase (`owner/my-app`) |
| `sast-report` | Path to the full Semgrep JSON report |
| `image-report` | Path to the full Trivy JSON report |

The two report files live in `$RUNNER_TEMP`, **not** in your workspace — a stray file in the build context would bust the Docker layer cache and, with a `COPY . .` Dockerfile, bake your vulnerability report into the shipped image.

Because they're in `$RUNNER_TEMP`, they are **deleted when the job ends**. Consume them in the same job, or download the artifact afterwards.

```yaml
- id: docker
  uses: abxst/actions/with-scan@v4
  with:
    push: true
    registry-username: ${{ github.actor }}
    registry-password: ${{ secrets.GITHUB_TOKEN }}

- name: Print image info
  run: |
    echo "Tags: ${{ steps.docker.outputs.image-tags }}"
    echo "Digest: ${{ steps.docker.outputs.image-digest }}"
```

---

## 5. Examples

### 5.1. Full v4 pipeline, both layers

The step order inside the action, so you know what you're looking at in the log:

```
Semgrep scan    → full JSON report (every severity)
Semgrep gate    → ✋ fails here if findings hit sast-gate-severity
Build (local for scan)
Trivy scan      → full JSON report (every severity)
Trivy gate      → ✋ fails here if CVEs hit gate-severity
Upload artifact → runs even when a gate failed
Push
```

```yaml
name: Docker Image CI

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: abxst/actions/with-scan@v4
        with:
          push: ${{ github.event_name != 'pull_request' }}
          registry-username: ${{ github.actor }}
          registry-password: ${{ secrets.GITHUB_TOKEN }}
```

### 5.2. Tune the SAST layer without changing variant

Add rulesets, skip generated code, and tighten the gate one notch above the `with-scan` default:

```yaml
- uses: abxst/actions/with-scan@v4
  with:
    push: true
    registry-username: ${{ github.actor }}
    registry-password: ${{ secrets.GITHUB_TOKEN }}
    sast-config: 'p/default,p/secrets'
    sast-gate-severity: 'WARNING,ERROR'
    sast-exclude: 'tests,vendor,**/*_generated.go'
    semgrep-version: '1.145.0'
```

### 5.3. Post-process the reports in the same job

The gate reports contain exactly the findings that blocked the build:

```yaml
- id: docker
  uses: abxst/actions/with-scan@v4
  continue-on-error: true
  with:
    push: false
    registry-username: ${{ github.actor }}
    registry-password: ${{ secrets.GITHUB_TOKEN }}

- name: Comment blocking CVEs on the PR
  if: steps.docker.outcome == 'failure' && github.event_name == 'pull_request'
  env:
    REPORT: ${{ steps.docker.outputs.image-report }}
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: |
    [ -f "$REPORT" ] || exit 0
    jq -r '.Results[]?.Vulnerabilities[]?
           | select(.Severity == "HIGH" or .Severity == "CRITICAL")
           | "- `\(.VulnerabilityID)` \(.PkgName) \(.InstalledVersion) → \(if (.FixedVersion // "") == "" then "no fix" else .FixedVersion end)"' \
      "$REPORT" > body.md
    gh pr comment "${{ github.event.number }}" --body-file body.md
```

### 5.4. Next.js application

```yaml
- uses: abxst/actions/with-scan@v4
  with:
    push: ${{ github.event_name != 'pull_request' }}
    registry-username: ${{ github.actor }}
    registry-password: ${{ secrets.GITHUB_TOKEN }}
    build-args: |
      NODE_ENV=production
      NEXT_PUBLIC_API_URL=https://api.example.com
```

### 5.5. Go service

```yaml
- uses: abxst/actions/with-scan@v4
  with:
    push: ${{ github.event_name != 'pull_request' }}
    registry-username: ${{ github.actor }}
    registry-password: ${{ secrets.GITHUB_TOKEN }}
    sast-exclude: 'vendor'
    build-args: |
      GO_VERSION=1.23
      LDFLAGS=-s -w -X main.version=${{ github.sha }}
```

### 5.6. Split: validate on PR, publish on merge

`.github/workflows/pr-check.yml`:

```yaml
name: PR Check
on:
  pull_request:
    branches: ["main"]
jobs:
  validate:
    runs-on: ubuntu-latest
    permissions: { contents: read, packages: write }
    steps:
      - uses: actions/checkout@v4
      - uses: abxst/actions/with-scan@v4
        with:
          push: false
          registry-username: ${{ github.actor }}
          registry-password: ${{ secrets.GITHUB_TOKEN }}
```

`.github/workflows/publish.yml`:

```yaml
name: Publish
on:
  push:
    branches: ["main"]
jobs:
  publish:
    runs-on: ubuntu-latest
    permissions: { contents: read, packages: write }
    steps:
      - uses: actions/checkout@v4
      - uses: abxst/actions/with-scan@v4
        with:
          push: true
          registry-username: ${{ github.actor }}
          registry-password: ${{ secrets.GITHUB_TOKEN }}
```

### 5.7. Multi-arch build (amd64 + arm64)

The default is single arch (`linux/amd64`). To build multi-arch, just set `platforms` — QEMU is already set up inside the action:

```yaml
- uses: abxst/actions/with-scan@v4
  with:
    push: true
    platforms: linux/amd64,linux/arm64
    registry-username: ${{ github.actor }}
    registry-password: ${{ secrets.GITHUB_TOKEN }}
```

Trivy scans a single arch (`scan-platform`, default `linux/amd64`) and the full multi-arch manifest is pushed. Semgrep is arch-independent — it reads source, not binaries.

⚠️ Building arm64 on an amd64 runner via QEMU is 3-5x slower than native. Only enable it when you actually need it (Raspberry Pi, M-series Macs, AWS Graviton).

### 5.8. Trigger ArgoCD after push (GitOps pattern)

```yaml
- id: docker
  uses: abxst/actions/with-scan@v4
  with:
    push: true
    registry-username: ${{ github.actor }}
    registry-password: ${{ secrets.GITHUB_TOKEN }}

- name: Update manifest repo for ArgoCD
  if: steps.docker.outputs.image-digest != ''
  env:
    DIGEST: ${{ steps.docker.outputs.image-digest }}
    MANIFEST_TOKEN: ${{ secrets.MANIFEST_REPO_TOKEN }}
  run: |
    git clone https://x-access-token:${MANIFEST_TOKEN}@github.com/your-org/k8s-manifest.git
    cd k8s-manifest/apps/my-app
    yq -i ".spec.template.spec.containers[0].image = \"ghcr.io/your-org/my-app@${DIGEST}\"" deployment.yaml
    git config user.email "ci@users.noreply.github.com"
    git config user.name "CI Bot"
    git commit -am "ci: update my-app to ${DIGEST:7:7}"
    git push
```

This is a deterministic alternative to ArgoCD Image Updater (commit-based GitOps instead of registry polling).

### 5.9. Semver release from git tag

```yaml
on:
  push:
    tags: ["v*.*.*"]

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions: { contents: read, packages: write }
    steps:
      - uses: actions/checkout@v4
      - uses: abxst/actions/strict@v4
        with:
          push: true
          registry-username: ${{ github.actor }}
          registry-password: ${{ secrets.GITHUB_TOKEN }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=semver,pattern={{major}}
            type=raw,value=latest
```

Pushing tag `v1.2.3` produces `1.2.3`, `1.2`, `1`, and `latest`.

---

## 6. Troubleshooting

### Semgrep flags something that isn't a real bug

In order of preference:

1. **Suppress the single finding** — add `// nosemgrep: <rule-id>` (or `# nosemgrep: <rule-id>`) on the line above. Narrowest possible scope, and it documents itself.
2. **`.semgrepignore`** at your project root — same syntax as `.gitignore`, for paths Semgrep shouldn't read at all.
3. **`sast-exclude`** — same idea, but expressed in the workflow rather than the repo.
4. **Narrow `sast-config`** — switch `p/default` → `p/ci` for a high-confidence, low-noise ruleset.

Prefer 1 over 4: suppressing one finding is a decision you can review later, while swapping the whole ruleset silently drops coverage you never see again.

### `Conflict: an artifact with this name already exists`

**Cause**: `actions/upload-artifact@v4` rejects duplicate artifact names within a single run. A matrix build calling this action more than once hits it.

**Fix**: give each matrix leg its own name:

```yaml
strategy:
  matrix:
    service: [api, worker]
steps:
  - uses: abxst/actions/with-scan@v4
    with:
      report-artifact-name: security-reports-${{ matrix.service }}
      # ...
```

### The reports artifact is missing

**Cause**: `report-artifact-name` was set to `''`, or the job was cancelled before the upload step.

Note the upload runs with `if: always()`, so a failed gate still produces the artifact — that is deliberate, since a red build is when you most want the report.

### Trivy fails on CVEs that have no available fix

**Fixes** depending on context:

1. Wait for the upstream fix (not great for active development)
2. Temporarily narrow the gate: `gate-severity: CRITICAL`
3. Set `gate-ignore-unfixed: 'true'` (already the default outside `strict`)
4. Add a `.trivyignore` file at your project root listing CVE IDs, **with a comment explaining why and a review-by date**
5. Switch base image (e.g. `node:20` → `node:20-alpine` or `node:20-slim`, or distroless)

### The build failed but the log just says "exit code 1"

Look one step further down. Each gate is followed by a summary step that runs even on failure and prints the blocking findings — rule ID and `file:line` for Semgrep, CVE and package for Trivy.

### Semgrep is slow on a large repo

Scan time scales with the amount of source it reads. Use `sast-exclude` to drop `vendor`, `node_modules`, generated code, and test fixtures. Switching `sast-config` to `p/ci` also cuts the rule count substantially.

### Cache isn't speeding up builds

**Cause**: GitHub Actions cache is scoped per branch and per repo. PRs from forks don't share cache with `main`.

**Fix**: Make sure CI runs on the upstream repo (not a fork), and order your Dockerfile layers so things that rarely change come first — `COPY package*.json ./` then `RUN npm ci` **before** `COPY . .`.

### Build args not applied

**Cause**: Missing `ARG` declaration in Dockerfile, or declared in the wrong stage of a multi-stage build.

**Fix**: Each stage needs its own `ARG` (args don't cross stages):

```dockerfile
ARG NODE_ENV=production
FROM node:20-alpine AS builder
ARG NODE_ENV       # must be re-declared in this stage
RUN echo "Building for $NODE_ENV"
```

---

## 7. FAQ

**Q: Semgrep and Trivy overlap, right? Why run both?**
A: They don't overlap at all. Trivy matches installed package versions against CVE databases — it has no idea what your code does. Semgrep pattern-matches your source — it has no idea what's in your base image. Dropping either one leaves a whole class of vulnerability unguarded.

**Q: Why does Semgrep run before the build instead of after?**
A: It only needs checked-out source, so running it first means a source-level failure costs seconds instead of the 2–4 minutes spent on QEMU, Buildx, registry login, and the build.

**Q: Why does `with-scan` build twice (once locally, once for push)?**
A: To scan the local image first and fail fast if it has CVEs. The second build hits the cache and only takes a few seconds. Pushing an unscanned image and scanning it in the registry would waste storage and bandwidth.

**Q: Where do the reports actually live?**
A: Two full JSON reports — one per scanner — written to `$RUNNER_TEMP` and uploaded as the `security-reports` artifact. The files themselves vanish with the runner at end of job; the artifact is what persists, subject to `report-retention-days`. Use the `sast-report` / `image-report` outputs to process them before the job ends.

**Q: Can I use this with registries other than GHCR?**
A: Yes. Set `registry` to your registry URL (e.g. `docker.io`, `harbor.example.com`, `<account>.dkr.ecr.<region>.amazonaws.com`) and pass appropriate `registry-username`/`registry-password`. The default `${{ secrets.GITHUB_TOKEN }}` only works for GHCR.

**Q: Do I need `actions/checkout@v4` before this action?**
A: Yes, and it matters more in v4 than before — Semgrep scans the checked-out working tree. Without checkout there is no source to scan and no Dockerfile to build.

**Q: Is `packages: write` too broad for a job that only validates?**
A: It's the minimum scope needed to push to GHCR. Validate-only workflows still need it because `docker/login-action` verifies credentials against GHCR. Known trade-off.

**Q: Why four separate variants instead of one action with a `scan: true/false` input?**
A: Explicit is better than implicit. Named paths make the security intent obvious at a glance in a workflow file — there's no flag to forget and accidentally publish an unscanned image. For the same reason there is no `sast-skip` input: tune the gate with `sast-config` / `sast-exclude`, or use `no-scan` if you genuinely want no scanning.

---

## 8. Versioning

| Pin pattern | When to use | Trade-off |
|---|---|---|
| `@v4` | Default for production workloads | Auto-receives patches and minor updates; behavior may shift slightly |
| `@v4.0.0` | When you need strict reproducibility | Never updates; manual bumps required |
| `@<sha>` | Critical infrastructure | Maximum security and reproducibility; tedious to maintain |

**Recommendation**: pin `@v4` for normal use. Major version bumps signal breaking changes and require explicit migration.

### Migrating v2/v3 → v4

The input interface is backward compatible — every new input has a default, so existing workflows keep working unchanged. Two things do change:

1. **Builds can now fail where they previously passed.** The Semgrep gate is new. Before adopting `@v4` on a protected branch, run it once on a throwaway branch to see what it finds, then fix or suppress before making it required.
2. **Reports are no longer printed as tables in the log.** Anything parsing the job log for Trivy output will break. Download the `security-reports` artifact, or read the JSON via the new `sast-report` / `image-report` outputs. The gate steps still print a short summary of what blocked the build.

To adopt gradually, start at `with-scan` (gates on `ERROR` only) and tighten from there.

No new job permissions are required.

---

## 9. Contributing

Contributions are welcome — bug reports, feature requests, and pull requests alike.

### Reporting a bug

Open an issue with:

- The workflow YAML you're using (mask any sensitive values)
- Error log pasted in a code block
- Link to your project repo (if public)

### Proposing a feature

Open an issue describing your use case **before** sending a PR. Since this action is used across many projects, every change needs to be evaluated for impact.

### Local development

The three scanning variants are **byte-identical below the `runs:` key** — they differ only in `name`, `description`, and three input defaults (`gate-severity`, `gate-ignore-unfixed`, `sast-gate-severity`). Any change to one must be mirrored to the other two. Verify with:

```bash
diff <(sed -n '/^runs:/,$p' with-scan/action.yml) <(sed -n '/^runs:/,$p' semi-strict/action.yml)
diff <(sed -n '/^runs:/,$p' with-scan/action.yml) <(sed -n '/^runs:/,$p' strict/action.yml)
```

Both must produce no output.

```bash
git clone git@github.com:abxst/actions.git
cd actions

# Test with act (https://github.com/nektos/act) in a sample project
cd /path/to/test-project
act -W .github/workflows/docker.yml \
    --secret GITHUB_TOKEN=$(gh auth token)
```

### Release flow (maintainers)

```bash
# Create a specific version tag
git tag -a v4.0.1 -m "v4.0.1: <one-line changelog>"
git push origin v4.0.1

# Move the major moving tag
git tag -d v4
git push origin :refs/tags/v4
git tag v4
git push origin v4
```

After the new tag is pushed, every caller pinned to `@v4` will receive the update on their next run.

---

## License

[MIT](./LICENSE) © [@abxst](https://github.com/abxst)

If this project helps you, consider starring the repo ⭐ — it helps others discover it.

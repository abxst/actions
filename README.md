# Docker Publish Actions

> Production-ready composite GitHub Actions to build, scan, and publish Docker images to any OCI registry. Ships two variants with explicit security trade-offs so you pick the right tool for the job.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![GitHub tag](https://img.shields.io/github/v/tag/abxst/actions)](https://github.com/abxst/actions/tag)
[![GitHub issues](https://img.shields.io/github/issues/abxst/actions)](https://github.com/abxst/actions/issues)
[![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/abxst/actions/pulls)

A recurring pattern across containerized projects: build the image → scan it for vulnerabilities → push to a registry. Doing this consistently across many repos usually means copy-pasting 70+ lines of YAML, and forgetting to update one place when something changes.

This repo packages that pattern as two reusable composite actions with a clean, opinionated interface. One enforces a Trivy security gate before pushing; the other skips scanning for cases where it's not appropriate.

## Features

- Build and push Docker images to any OCI registry (defaults to GHCR)
- Two-pass Trivy scan: informational full report + hard gate on HIGH/CRITICAL
- Automatic image-name lowercasing (GHCR requirement)
- GitHub Actions layer caching out of the box
- Multi-arch builds (amd64, arm64) by default — QEMU set up automatically
- Customizable tag strategies (sha+date, semver, custom)
- Build args, custom Dockerfile path, custom build context
- Two variants sharing the same interface, easy to switch between

---

## 1. Choose a variant

| | **strict** | **semi-strict** | **with-scan** | **no-scan** |
|---|---|---|---|---|
| **Path** | `abxst/actions/strict@v2` | `abxst/actions/semi-strict@v2` | `abxst/actions/with-scan@v2` | `abxst/actions/no-scan@v2` |
| **Trivy hard gate** | Yes (all severities UNKNOWN→CRITICAL, incl. unfixed CVEs) | Yes (all severities UNKNOWN→CRITICAL, unfixed ignored) | Yes (HIGH/CRITICAL with available fix) | No |
| **Speed** | ~2-4 min (build twice + scan) | ~2-4 min | ~2-4 min | ~1-2 min (build once) |
| **Use for** | Highest-sensitivity images (payment, auth, PII) | Sensitive images where unfixable CVEs shouldn't block | Production-facing images | Internal tools, dev images, images already scanned elsewhere |

**Rule of thumb**: if your image will run somewhere a user or customer can reach (even indirectly via a production container) → use `with-scan`. Otherwise → `no-scan`.

Examples by category:
- Public-facing frontend, API backend, anything in production → **with-scan**
- Internal tooling, dev playgrounds, sandboxes → **no-scan**

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

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - uses: abxst/actions/with-scan@v1
        with:
          push: ${{ github.event_name != 'pull_request' }}
          registry-username: ${{ github.actor }}
          registry-password: ${{ secrets.GITHUB_TOKEN }}
```

That's the whole thing. Behavior:

- **PR into `main`** → build + scan, but DON'T push (acts as a gate for code review)
- **Push to `main`** → build + scan + push to `ghcr.io/<owner>/<repo>`

### 2.2. Check repo permissions

Go to **Settings → Actions → General → Workflow permissions** and make sure:

- ✅ **Read and write permissions** is selected
- ✅ Allow GitHub Actions to create and approve pull requests (optional)

Without this, the push step fails with `denied: permission_denied`.

### 2.3. (Optional) Branch protection

To make the Trivy gate actually enforce anything, go to **Settings → Branches → Add rule** for `main`:

- ✅ **Require status checks to pass before merging**
- Select the `build` check (matches the job name in the workflow)

Now a PR with HIGH/CRITICAL CVEs cannot be merged.

### 2.4. Verify your first run

Push a commit to a test branch:

```bash
git checkout -b test/docker-action
git commit --allow-empty -m "test: trigger docker action"
git push origin test/docker-action
```

Open a PR into `main`, then check the **Actions** tab. If Trivy fails, read the table in the step log to see which CVEs were flagged.

---

## 3. Inputs

### 3.1. Common to both variants

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
| `platforms` | no | `linux/amd64,linux/arm64` | Target platforms for the pushed image. Multi-arch by default; QEMU is set up automatically. Set to a single platform (e.g. `linux/amd64`) to build native-only. |

Default `tags`:

```
type=raw,value={{sha}}-{{date 'DDMMYYYY'}},enable={{is_default_branch}}
type=raw,value=latest,enable={{is_default_branch}}
```

→ Produces tags like `abc123f-24052026` and `latest`, but only when pushing to the default branch.

### 3.2. Scanning variants only (`strict`, `semi-strict`, `with-scan`)

| Input | Required | Default | Description |
|---|---|---|---|
| `gate-severity` | no | `HIGH,CRITICAL` (with-scan) / `UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL` (strict, semi-strict) | Severity levels that fail the pipeline. |
| `gate-ignore-unfixed` | no | `'true'` (with-scan, semi-strict) / `'false'` (strict) | Ignore CVEs that have no available fix. |
| `scan-platform` | no | `linux/amd64` | Single platform built locally for the Trivy scan. A `load: true` build can't hold a multi-arch image, so the scan runs on one arch while the pushed image stays multi-arch (`platforms`). |

---

## 4. Outputs

Both variants expose the same three outputs for use in downstream steps:

| Output | Description |
|---|---|
| `image-tags` | Tags generated by `docker/metadata-action` (multiline string) |
| `image-digest` | `sha256:...` digest of the pushed image (empty if not pushed) |
| `image-name-lc` | Image name in lowercase (`owner/my-app`) |

Usage:

```yaml
- id: docker
  uses: abxst/actions/with-scan@v1
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

### 5.1. Next.js application

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
      - uses: abxst/actions/with-scan@v1
        with:
          push: ${{ github.event_name != 'pull_request' }}
          registry-username: ${{ github.actor }}
          registry-password: ${{ secrets.GITHUB_TOKEN }}
          build-args: |
            NODE_ENV=production
            NEXT_PUBLIC_API_URL=https://api.example.com
```

### 5.2. Go service

```yaml
- uses: abxst/actions/with-scan@v1
  with:
    push: ${{ github.event_name != 'pull_request' }}
    registry-username: ${{ github.actor }}
    registry-password: ${{ secrets.GITHUB_TOKEN }}
    build-args: |
      GO_VERSION=1.23
      LDFLAGS=-s -w -X main.version=${{ github.sha }}
```

### 5.3. Split: validate on PR, publish on merge

Keep the two flows in separate files for clarity:

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
      - uses: abxst/actions/with-scan@v1
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
      - uses: abxst/actions/with-scan@v1
        with:
          push: true
          registry-username: ${{ github.actor }}
          registry-password: ${{ secrets.GITHUB_TOKEN }}
```

### 5.4. Multi-arch build (amd64 + arm64)

Multi-arch is the **default** — `platforms` is `linux/amd64,linux/arm64` and QEMU is
set up inside the action, so no extra steps are needed:

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: abxst/actions/with-scan@v1
    with:
      push: true
      registry-username: ${{ github.actor }}
      registry-password: ${{ secrets.GITHUB_TOKEN }}
```

The scanning variants scan a single arch (`scan-platform`, default `linux/amd64`)
and push the full multi-arch manifest.

⚠️ Building arm64 on an amd64 runner via QEMU is 3-5x slower than native. To build a
single arch only, set `platforms` explicitly:

```yaml
  - uses: abxst/actions/with-scan@v1
    with:
      push: true
      platforms: linux/amd64          # native-only, no emulation
      registry-username: ${{ github.actor }}
      registry-password: ${{ secrets.GITHUB_TOKEN }}
```

### 5.5. Trigger ArgoCD after push (GitOps pattern)

```yaml
- id: docker
  uses: abxst/actions/with-scan@v1
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

### 5.6. Semver release from git tag

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
      - uses: abxst/actions/with-scan@v1
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

### `denied: permission_denied` when pushing to GHCR

**Cause**: Repo settings block workflows from writing packages.

**Fix**: Settings → Actions → General → Workflow permissions → **Read and write permissions**.

### `repository name must be lowercase`

**Cause**: GHCR requires lowercase image names, but `github.repository` may contain uppercase characters (e.g. `MyOrg/MyApp`).

**Fix**: The action handles this internally via the `Lowercase image name` step. If you still see this error, check whether your `image-name` input contains uppercase characters and make sure downstream steps reference the `image-name-lc` output, not the input.

### Trivy fails on CVEs that have no available fix

**Cause**: HIGH/CRITICAL CVEs in your base image or dependencies that upstream hasn't patched yet.

**Fixes** depending on context:

1. Wait for the upstream fix (not great for active development)
2. Temporarily narrow the gate: `gate-severity: CRITICAL` (block CRITICAL only)
3. Set `gate-ignore-unfixed: 'true'` (already the default)
4. Add a `.trivyignore` file at your project root listing CVE IDs to suppress, **with a comment explaining why and a review-by date**
5. Switch base image (e.g. `node:20` → `node:20-alpine` or `node:20-slim`, or distroless)

### Cache isn't speeding up builds

**Cause**: GitHub Actions cache is scoped per branch and per repo. PRs from forks don't share cache with `main`.

**Fix**: Make sure CI runs on the upstream repo (not a fork), and order your Dockerfile layers so things that rarely change come first. For example, `COPY package*.json ./` then `RUN npm ci` **before** `COPY . .`.

### Lock file out of sync (npm)

**Cause**: `package.json` and `package-lock.json` are not in sync. Often caused by running `npm install` for a CVE fix instead of `npm ci`.

**Fix**: Run `npm install` locally, commit the updated `package-lock.json`. In your Dockerfile, use `npm ci` to enforce lock-file usage.

### Build args not applied

**Cause**: Missing `ARG` declaration in Dockerfile, or declared in the wrong stage of a multi-stage build.

**Fix**: Each stage needs its own `ARG` (args don't cross stages). For example:

```dockerfile
ARG NODE_ENV=production
FROM node:20-alpine AS builder
ARG NODE_ENV       # must be re-declared in this stage
RUN echo "Building for $NODE_ENV"
```

---

## 7. FAQ

**Q: Why does `with-scan` build twice (once locally, once for push)?**
A: To scan the local image first and fail fast if it has CVEs. The second build hits the cache and only takes a few seconds. Pushing the unscanned image to the registry first and scanning there would waste storage and bandwidth.

**Q: Can I use this with registries other than GHCR?**
A: Yes. Set `registry` to your registry URL (e.g. `docker.io`, `harbor.example.com`, `<account>.dkr.ecr.<region>.amazonaws.com`) and pass appropriate `registry-username`/`registry-password`. The default `${{ secrets.GITHUB_TOKEN }}` only works for GHCR; other registries need their own secret.

**Q: Is pinning `@v1` safe?**
A: Reasonably safe for most use cases — the `v1` major tag follows semver and only receives backward-compatible changes. If you need full reproducibility, pin to a specific version (`@v1.0.0`) or a commit SHA (`@<sha>`).

**Q: Do I need `actions/checkout@v4` before this action?**
A: Yes. The action needs access to your Dockerfile and build context. Put `checkout` as your first step.

**Q: Is `packages: write` permission too broad?**
A: It's the minimum scope needed to push to GHCR. Workflows that only validate (build + scan, no push) still need this permission because `docker/login-action` verifies credentials against GHCR. This is a known trade-off.

**Q: Trivy reports are long — how do I read them on the GitHub UI?**
A: The `Trivy scan (informational, full)` step prints a table directly into the Actions log. Click the job → expand the step. For persistent reports, fork the action and add `format: sarif` plus an upload step to feed into GitHub Code Scanning.

**Q: Why two separate variants instead of a single action with a `scan: true/false` input?**
A: Explicit is better than implicit. Two named paths make the security intent clear at a glance in workflow files — there's no risk of forgetting to set a flag and accidentally publishing unscanned images.

---

## 8. Versioning

This project follows the GitHub Actions ecosystem convention:

| Pin pattern | When to use | Trade-off |
|---|---|---|
| `@v1` | Default for production workloads | Auto-receives patches and minor updates; behavior may shift slightly |
| `@v1.0.0` | When you need strict reproducibility | Never updates; manual bumps required |
| `@<sha>` | Critical infrastructure | Maximum security and reproducibility; tedious to maintain |

**Recommendation**: pin `@v1` for normal use to receive bug fixes automatically. Major version bumps (`v2`, `v3`) signal breaking changes and require explicit migration.

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
git tag -a v1.0.1 -m "v1.0.1: <one-line changelog>"
git push origin v1.0.1

# Move the major moving tag
git tag -d v1
git push origin :refs/tags/v1
git tag v1
git push origin v1
```

After the new tag is pushed, every caller pinned to `@v1` will receive the update on their next run.

---

## License

[MIT](./LICENSE) © [@abxst](https://github.com/abxst)

If this project helps you, consider starring the repo ⭐ — it helps others discover it.

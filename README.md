# Actions

> Bộ composite GitHub Actions cá nhân của [@abxst](https://github.com/abxst) — build và publish Docker image, với 2 variant tách biệt rõ ràng theo mức độ security gate.

Mục đích: chuẩn hoá pipeline Docker build/push cho các project (chủ yếu dùng cho stack của CCVI Technology). Thay vì copy-paste 70 dòng YAML giữa các repo, mỗi project chỉ cần 1 step `uses:`.

---

## 1. Chọn variant nào?

| | **with-scan** | **no-scan** |
|---|---|---|
| **Path** | `abxst/actions/with-scan@v1` | `abxst/actions/no-scan@v1` |
| **Trivy hard gate** | Có (fail nếu có CVE HIGH/CRITICAL có fix) | Không |
| **Tốc độ** | ~2-4 phút (build 2 lần + scan) | ~1-2 phút (build 1 lần) |
| **Dùng cho** | Image production-facing | Internal tool, dev image, image đã scan ở pipeline khác |

**Quy tắc đơn giản**: Nếu user/khách hàng có thể tiếp cận được image (kể cả gián tiếp qua container chạy production) → dùng `with-scan`. Còn lại → `no-scan`.

Với các project hiện tại của CCVI:
- `johub-fe`, `johub-be`, `eduhubvn-fe`, `eduhubvn-be`, `eduhubvn-welcome` → **with-scan**
- Internal tooling, dev playground → **no-scan**

---

## 2. Setup lần đầu (5 phút)

### 2.1. Workflow file

Tạo `.github/workflows/docker.yml` trong repo project:

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

Đó là toàn bộ. Behaviour:
- **PR vào `main`** → build + scan, KHÔNG push (gate cho code review)
- **Push lên `main`** → build + scan + push lên `ghcr.io/ccvi-technology/<repo-name>`

### 2.2. Kiểm tra permission ở repo

Vào **Settings → Actions → General → Workflow permissions**, đảm bảo:
- ✅ **Read and write permissions** được chọn
- ✅ Allow GitHub Actions to create and approve pull requests (optional)

Nếu chưa bật, workflow sẽ fail ở step push với lỗi `denied: permission_denied`.

### 2.3. (Optional) Branch protection

Để Trivy gate có tác dụng thực sự, vào **Settings → Branches → Add rule** cho `main`:
- ✅ **Require status checks to pass before merging**
- Chọn check `build` (job name trong workflow)

Như vậy PR có CVE HIGH/CRITICAL sẽ không merge được.

### 2.4. Verify lần chạy đầu

Push 1 commit lên branch test:

```bash
git checkout -b test/docker-action
git commit --allow-empty -m "test: trigger docker action"
git push origin test/docker-action
```

Mở PR vào `main`, xem tab **Actions** → workflow `Docker Image CI` chạy. Nếu Trivy fail, đọc bảng table trong log để xem CVE nào.

---

## 3. Inputs

### 3.1. Chung cho cả 2 variant

| Input | Required | Default | Mô tả |
|---|---|---|---|
| `registry` | no | `ghcr.io` | Container registry URL |
| `image-name` | no | `${{ github.repository }}` | Image name không kèm registry (vd `ccvi-technology/johub-fe`) |
| `context` | no | `.` | Docker build context |
| `dockerfile` | no | `./Dockerfile` | Đường dẫn Dockerfile |
| `push` | no | `'false'` | Push image (chuỗi `'true'`/`'false'`) |
| `registry-username` | **yes** | — | Username đăng nhập registry |
| `registry-password` | **yes** | — | Token/password đăng nhập registry |
| `tags` | no | xem dưới | Input tags cho `docker/metadata-action` (multiline) |
| `build-args` | no | `''` | Build args (multiline `KEY=VALUE`) |
| `platforms` | no | `''` | Target platforms (vd `linux/amd64,linux/arm64`). Để trống = native |

Default của `tags`:
```
type=raw,value={{sha}}-{{date 'DDMMYYYY'}},enable={{is_default_branch}}
type=raw,value=latest,enable={{is_default_branch}}
```

→ Tag dạng `abc123f-24052026` và `latest`, chỉ khi push lên branch mặc định.

### 3.2. Riêng `with-scan`

| Input | Required | Default | Mô tả |
|---|---|---|---|
| `gate-severity` | no | `HIGH,CRITICAL` | Severity gây fail. Bỏ `HIGH` nếu chỉ muốn block CRITICAL |
| `gate-ignore-unfixed` | no | `'true'` | Bỏ qua CVE chưa có fix. Đặt `'false'` cho image cực sensitive |
| `trivy-version` | no | `v0.36.0` | Version `aquasecurity/trivy-action` |

---

## 4. Outputs

Cả 2 variant trả về 3 outputs có thể dùng ở step sau:

| Output | Mô tả |
|---|---|
| `image-tags` | Tags đã generate từ metadata-action (multiline string) |
| `image-digest` | Digest dạng `sha256:...` của image đã push (rỗng nếu chưa push) |
| `image-name-lc` | Image name dạng lowercase (`ccvi-technology/johub-fe`) |

Cách dùng:

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

## 5. Scenarios thực tế

### 5.1. Next.js app (vd `eduhubvn-fe`, `johub-fe`)

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
            NEXT_PUBLIC_API_URL=https://api.eduhubvn.ccvi.com.vn
```

### 5.2. Go service (vd `johub-be`, `eduhubvn-be`)

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

### 5.3. Split: validate ở PR, publish khi merge

Cách này tách thành 2 workflow file để clear hơn:

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

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: docker/setup-qemu-action@v3   # cần thêm QEMU cho cross-build
  - uses: abxst/actions/with-scan@v1
    with:
      push: true
      platforms: linux/amd64,linux/arm64
      registry-username: ${{ github.actor }}
      registry-password: ${{ secrets.GITHUB_TOKEN }}
```

⚠️ Build arm64 trên runner amd64 sẽ chậm gấp 3-5 lần do QEMU emulate. Chỉ bật khi thực sự cần (vd deploy lên Raspberry Pi, M-series Mac).

### 5.5. Trigger ArgoCD sau khi push

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
    git clone https://x-access-token:${MANIFEST_TOKEN}@github.com/abxst/k8s-manifest.git
    cd k8s-manifest/apps/johub-fe
    yq -i ".spec.template.spec.containers[0].image = \"ghcr.io/ccvi-technology/johub-fe@${DIGEST}\"" deployment.yaml
    git config user.email "ci@ccvi.com.vn"
    git config user.name "CCVI CI"
    git commit -am "ci: update johub-fe to ${DIGEST:7:7}"
    git push
```

Cách này thay cho ArgoCD Image Updater nếu muốn deterministic hơn (commit-based GitOps thay vì polling).

### 5.6. Semver release từ git tag

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

Push tag `v1.2.3` sẽ tạo `1.2.3`, `1.2`, `1`, `latest`.

---

## 6. Troubleshooting

### `denied: permission_denied` khi push GHCR

**Nguyên nhân**: Repo setting chặn workflow ghi packages.

**Fix**: Settings → Actions → General → Workflow permissions → **Read and write permissions**.

### `repository name must be lowercase`

**Nguyên nhân**: GHCR yêu cầu image name lowercase nhưng `github.repository` có thể chứa chữ hoa (vd `CCVI-Technology/Johub-fe`).

**Fix**: Action đã tự xử lý chỗ này qua step `Lowercase image name`. Nếu vẫn lỗi → check `image-name` bạn truyền vào có chứa chữ hoa và bạn đang dùng outputs `image-name-lc` chứ không phải input `image-name`.

### Trivy fail nhưng CVE không có cách fix

**Nguyên nhân**: Có CVE HIGH/CRITICAL trong base image hoặc dependency mà upstream chưa release fix.

**Fix tuỳ tình huống**:
1. Đợi fix → tạm skip pipeline (không khuyến khích)
2. Override severity tạm thời: `gate-severity: CRITICAL` (chỉ block CRITICAL)
3. Đặt `gate-ignore-unfixed: 'true'` (đã là default)
4. Tạo file `.trivyignore` ở root project, list CVE ID muốn ignore + comment lý do và ngày review lại
5. Đổi base image (vd `node:20` → `node:20-alpine` hoặc `node:20-slim`)

### Cache không hoạt động (build mãi không nhanh hơn)

**Nguyên nhân**: GHA cache theo branch + repo. PR từ fork không share cache với main.

**Fix**: Đảm bảo CI chạy ở repo gốc (không phải fork), và Dockerfile layer không thay đổi mỗi lần (vd `COPY package*.json ./` rồi `RUN npm ci` TRƯỚC khi `COPY . .`).

### Lock file out of sync (npm)

**Nguyên nhân**: `package.json` và `package-lock.json` không khớp. Thường do CVE fix dùng `npm install` thay vì `npm ci`.

**Fix**: Chạy `npm install` ở local, commit lại `package-lock.json`. Trong Dockerfile dùng `npm ci` để bắt buộc dùng lock file.

### Build args không apply

**Nguyên nhân**: Quên khai báo `ARG` trong Dockerfile, hoặc khai báo sai stage trong multi-stage build.

**Fix**: Mỗi stage cần `ARG` riêng (ARG không cross stage). Vd:
```dockerfile
ARG NODE_ENV=production
FROM node:20-alpine AS builder
ARG NODE_ENV       # phải khai báo lại trong stage
RUN echo "Building for $NODE_ENV"
```

---

## 7. FAQ

**Q: Tại sao build 2 lần ở `with-scan` (1 lần local, 1 lần push)?**
A: Để scan image LOCAL trước, fail sớm nếu có CVE. Lần build thứ 2 dùng cache nên chỉ tốn vài giây. Nếu chỉ push 1 lần và scan trên registry sẽ tốn dung lượng + bandwidth không cần thiết.

**Q: Có thể dùng action này với registry khác GHCR không?**
A: Có. Đổi `registry` input (vd `docker.io`, `harbor.ccvi.com.vn`) và truyền đúng `registry-username` / `registry-password`. Token GHCR mặc định là `${{ secrets.GITHUB_TOKEN }}`, registry khác cần secret riêng.

**Q: Pin `@v1` có an toàn không?**
A: An toàn vì repo do bạn (hoặc team) tự quản, không phụ thuộc upstream. Với 3rd-party action, nên pin commit SHA. Nếu cần immutability tuyệt đối → pin `@v1.0.0`.

**Q: Có cần `actions/checkout@v4` trước action không?**
A: Có, vì action cần access Dockerfile và context. Đặt `checkout` là step đầu tiên.

**Q: Permissions `packages: write` có quá rộng không?**
A: Đây là scope tối thiểu để push GHCR. Nếu workflow KHÔNG push (chỉ build + scan ở PR), vẫn cần permission này vì `docker/login-action` verify credentials với GHCR. Trade-off chấp nhận được.

**Q: Trivy report dài, làm sao đọc trên GitHub UI?**
A: Step `Trivy scan (informational, full)` in ra bảng table trong log Actions. Click vào job → expand step → đọc trực tiếp. Nếu cần persist, fork action và thêm `format: sarif` + upload lên GitHub Code Scanning.

---

## 8. Versioning

Theo convention chuẩn của ecosystem GitHub Actions:

| Pin pattern | Khi nào dùng | Trade-off |
|---|---|---|
| `@v1` | Default cho team CCVI / production | Auto nhận patch + minor, có thể có behaviour thay đổi nhẹ |
| `@v1.0.0` | Khi cần lock cứng | Không nhận update, phải manual bump |
| `@<sha>` | Critical infrastructure | Tối đa security/reproducibility, maintain phiền |

Khuyến nghị: pin `@v1` cho stack CCVI Technology để tự nhận patch.

---

## 9. Đóng góp

### Báo bug

Mở issue ở repo này với:
- Workflow YAML đang dùng (mask sensitive value)
- Log lỗi (paste vào code block)
- Project repo (nếu public)

### Đề xuất feature

Mở issue mô tả use case TRƯỚC khi mở PR. Action này được dùng bởi nhiều project nên mọi thay đổi cần review tác động.

### Local development

```bash
git clone git@github.com:abxst/actions.git
cd actions

# Test với act (https://github.com/nektos/act)
cd /path/to/test-project
act -W .github/workflows/docker.yml \
    --secret GITHUB_TOKEN=$(gh auth token)
```

### Release flow (maintainer only)

```bash
git tag -a v1.0.1 -m "v1.0.1: <one-line changelog>"
git push --tags

# Move major moving tag
git tag -d v1
git push origin :refs/tags/v1
git tag v1
git push --tags
```

Sau push tag mới, mọi workflow caller pin `@v1` sẽ tự nhận update lần chạy kế tiếp.

---

## License

MIT. Xem [LICENSE](./LICENSE).
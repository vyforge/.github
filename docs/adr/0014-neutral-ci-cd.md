# ADR-0014: CI/CD và registry trung lập với nền tảng git

- **Trạng thái:** Chấp nhận — 26/10/2026. Thay phần vị trí preset Renovate của [ADR-0004](0004-renovate.md) (các luật
  của preset giữ nguyên). Thuộc kế hoạch trong [MODULE_PLAN.md](../MODULE_PLAN.md), mục "CI/CD và registry trung lập với
  nền tảng git"; nghiên cứu ở mục 8 của [báo cáo quy hoạch](../research/2026-10-quy-hoach-vyforge.md).

## Bối cảnh

Dự án dùng Vyforge có thể đặt git trên GitHub, GitLab hoặc Forgejo/Gitea on-prem. Hiện toàn bộ đường đi từ commit tới
deploy gắn với GitHub:

- 7 reusable workflow `@v1` trong repo `.github` (`java-ci`, `node-ci`, `container`, `gitops-bump`, `maven-publish`,
  `npm-publish`, `helm-publish`) chứa logic build, publish và ký. Template sinh `.github/workflows` gọi các workflow đó.
- Lệnh grep tìm `ghcr.io`, `npm.pkg.github.com`, `maven.pkg.github.com`, `PACKAGES_TOKEN`, `vyforge/.github/`,
  `attest-build-provenance` khớp **70 file trong 10 repo**, cộng 26 file workflow. Ví dụ: `container.yml` ghi cứng
  `ghcr.io`, `maven-publish.yml` ghi cứng `maven.pkg.github.com`, `publishConfig.registry` trong 11 `package.json` của
  `platform-web`, `<repository>` GitHub Packages trong `pom.xml` của ba service.
- Forgejo chỉ cho gọi workflow ở repo khác khi repo đó **public**, còn code Vyforge là private. Reusable workflow vì vậy
  không mang sang Forgejo được.
- `actions/attest-build-provenance` cho repo private cần gói GitHub Enterprise Cloud.
- Từ 26/10/2026, việc phát hành đã tạm chạy từ máy local bằng script `tools/vyforge-release` để không phụ thuộc GitHub
  Actions.

## Quyết định

### 1. Logic pipeline nằm trong CLI `vyforge-ci`, đóng trong image `ci-toolbox`

- Repo mới **`platform-ci`** chứa CLI `vyforge-ci` (bash, test bằng bats). Mỗi bước là một lệnh: `java-verify`,
  `node-verify`, `maven-publish`, `npm-publish`, `chart-publish`, `image`, `gitops-bump`, `tag`, `npm-login`,
  `maven-settings`, `version`. Logic giữ đúng như các reusable workflow hiện có, kể cả kiểm tag semver.
- CLI được đóng trong image **`ci-toolbox:<semver>`**, ghim version và checksum hoặc digest của: JDK 25, Node 24 +
  pnpm 11, Helm 4, cosign, Syft, Trivy, crane, Copier, just, yq, Docker CLI + buildx, git, bash, jq.
- `platform-ci` phát hành theo semver: tag `vX.Y.Z` đi cùng image `ci-toolbox:X.Y.Z`. Bản đầu là `0.1.0`.
- Version phát hành lấy từ tag `vX.Y.Z`: `CI_TAG`, tag do CI cung cấp (GitLab, GitHub, Forgejo, Gitea), rồi
  `git describe`. Tag không phải semver thì lệnh dừng.
- `VYFORGE_CI_DRY_RUN=1` in lệnh thay vì chạy, dùng cho test và kiểm tra trước khi phát hành.

### 2. File CI của từng nền tảng chỉ là adapter mỏng

- Adapter khoảng 20 dòng: chọn image toolbox, khai báo trigger, map secret sang biến môi trường, gọi `vyforge-ci <bước>`.
- `platform-ci` có adapter mẫu cho GitHub Actions, Forgejo/Gitea Actions (cùng nội dung, khác thư mục) và GitLab CI, mỗi
  nền tảng có năm loại: service Java, app Node, thư viện Maven, thư viện npm, chart.
- Không dùng `uses:` gọi workflow ở repo khác. Logic dùng chung đi qua image, không đi qua reusable workflow.
- Mỗi repo dự án có `justfile` (task runner [`just`](https://github.com/casey/just)) để dev chạy đúng lệnh mà CI chạy:
  `just verify`, `just image`. Không chọn Dagger vì còn 0.x và cần engine chạy đặc quyền.

### 3. Mọi đầu mối bên ngoài là tham số

| Biến | Dùng cho |
|---|---|
| `VYFORGE_MAVEN_URL` | Repository Maven chứa `com.vyforge.platform:*` (đọc và deploy) |
| `VYFORGE_MAVEN_MIRROR` | (tuỳ chọn) Proxy cache cho các repository còn lại, kiểu Nexus/Reposilite |
| `VYFORGE_NPM_REGISTRY` | Registry của scope `@vyforge` |
| `VYFORGE_OCI_REGISTRY` | Tiền tố image và nơi đặt `ci-toolbox` |
| `VYFORGE_CHART_REPO` | Repo chart OCI (`oci://...`) |
| `REGISTRY_USER`, `REGISTRY_TOKEN` | Tài khoản và token cho các registry trên |
| `COSIGN_KEY` | Khoá ký image |
| `GITOPS_REPO`, `GITOPS_FILE`, `GITOPS_TOKEN` | Repo infra, file values và token ghi repo đó |

- Không biến nào có giá trị mặc định trỏ về GitHub. Không repo nào ghi cứng `ghcr.io`, `npm.pkg.github.com` hay
  `maven.pkg.github.com`, trừ file adapter GitHub và tài liệu lịch sử.
- Tên secret không mang tên nền tảng: bỏ `PACKAGES_TOKEN`.
- **Maven:** `vyforge-ci` sinh `settings.xml` có repository `vyforge` (bật qua profile) và mirror `*,!vyforge` khi có
  `VYFORGE_MAVEN_MIRROR`. Server đọc `${env.REGISTRY_USER}`/`${env.REGISTRY_TOKEN}`, nên file không chứa token. POM bỏ
  `<repositories>`. Deploy qua `-DaltDeploymentRepository=vyforge::$VYFORGE_MAVEN_URL`, POM không cần
  `<distributionManagement>`.
- **npm:** `.npmrc` của repo và `package.json` không ghi registry (`publishConfig.registry` bị bỏ). `vyforge-ci
  npm-login` ghi registry và token vào `~/.npmrc`. Repo đổi registry phải sinh lại `pnpm-lock.yaml`, vì lockfile ghi URL
  tarball.
- **Dockerfile:** token tải artifact nội bộ đi qua BuildKit secret `REGISTRY_TOKEN`; registry npm qua build arg
  `NPM_REGISTRY`. `vyforge-ci image` vẫn truyền thêm secret `GITHUB_TOKEN` và build arg `GITHUB_ACTOR` cho Dockerfile cũ
  trong lúc chuyển đổi.

### 4. Image: quét trước khi push, SBOM và provenance của BuildKit, ký bằng cosign

- Build và quét Trivy trước; image có lỗ hổng HIGH/CRITICAL đã có bản vá thì không push. PR/MR chỉ build và quét.
- Push kèm SBOM và provenance **SLSA v1** của BuildKit (`--sbom=true --provenance=mode=max,version=v1`). Tag:
  `sha-<sha>`, tên nhánh, `X.Y.Z` khi build từ tag. Cache build cấu hình qua `VYFORGE_BUILD_CACHE` (thay `type=gha`).
- Khi có `COSIGN_KEY`, ký `image@digest` và attest provenance (`cosign attest --type slsaprovenance1`). Khoá nằm trong
  OpenBao Transit (`hashivault://`) on-prem, hoặc KMS của cloud. Không dùng keyless và không dùng Rekor công khai:
  cosign 3 ký với signing config không có transparency log; khi verify cần `--insecure-ignore-tlog` hoặc policy tương
  ứng trong Kyverno. Toolbox ghim version cosign, vì hành vi này đã đổi giữa các bản patch 3.0.x.
- Bỏ `actions/attest-build-provenance`.

### 5. Deploy bằng GitOps pull

Argo CD đọc repo infra trên bất kỳ git server nào. CI chỉ commit digest mới bằng `vyforge-ci gitops-bump`: clone, sửa
bằng `yq`, commit, push qua git thuần, không gọi API của nền tảng. Token đi qua `GIT_ASKPASS`, không nằm trong URL. Chỉ
bump môi trường dev; staging và prod đi bằng PR với cùng digest. CI không giữ quyền vào cluster.

### 6. Bộ on-prem tham chiếu

| Vai trò | Chọn |
|---|---|
| Git server và CI | Forgejo 15 LTS + Forgejo Runner (Gitea dùng chung adapter; GitLab CE khi khách hàng đã có) |
| Registry Maven và npm của Vyforge | Package registry của Forgejo |
| Proxy cache Maven Central / npmjs | Nexus sẵn có của công ty (nếu dưới ngưỡng CE), hoặc Reposilite + Verdaccio |
| Image và chart | Harbor (quét Trivy, giữ chữ ký cosign, proxy cache, replication) |
| Khoá ký | OpenBao Transit |
| Cập nhật dependency | Renovate self-host (`platform: forgejo`), chạy dạng CronJob |
| Deploy | Argo CD |

Không dùng Nexus CE làm registry chính vì giới hạn 40.000 component và 100.000 request/ngày. Không chọn Woodpecker,
Jenkins hay Tekton làm runner mặc định; Tekton Chains chỉ cân nhắc khi cần SLSA Build L3.

### 7. Repo `.github` tách thành `platform-ci` và `platform-docs`

- `platform-ci`: CLI, Dockerfile của `ci-toolbox`, adapter, preset Renovate, `justfile` mẫu. 7 reusable workflow `@v1`
  đổi thành adapter GitHub chỉ gọi `vyforge-ci`, để dự án đang dùng `@v1` vẫn chạy trong lúc chuyển đổi.
- `platform-docs`: ADR, `ARCHITECTURE.md`, `MODULE_PLAN.md`, `REPOSITORY_GUIDELINES.md`, research; build thành site
  tài liệu tĩnh.
- Trên GitHub chỉ còn repo `.github` rất mỏng, chứa profile org.
- Preset Renovate chuyển từ `github>vyforge/.github:renovate-config` sang **`local>vyforge/platform-ci:renovate`**. Cú
  pháp `local>` dùng được trên mọi nền tảng Renovate hỗ trợ. Nội dung preset giữ nguyên luật của ADR-0004.

## Hệ quả

- Một service đi được từ commit tới deploy dev mà không cần GitHub. Tiêu chí kiểm: tắt quyền truy cập GitHub của runner
  on-prem mà `platform-gateway` vẫn build, test, publish, đẩy image lên Harbor, ký, quét, bump GitOps và được Argo CD
  deploy.
- Logic pipeline có version và có test. Sửa một lần trong `platform-ci`, các dự án nhận qua bump version image toolbox
  (Renovate).
- Dev chạy được đúng lệnh của CI ở local (`just verify`, `just image`), kể cả trong image toolbox.
- Mất một số tiện ích riêng của GitHub Actions: cache `type=gha` (thay bằng cache registry), giao diện attestation của
  GitHub, upload artifact báo cáo test (adapter của từng nền tảng tự thêm nếu cần).
- Image toolbox lớn (khoảng 1,4 GB) và kéo theo lỗ hổng của binary upstream (Go stdlib trong Helm, Syft, crane, yq,
  buildx). Toolbox chỉ chạy trong CI, không deploy, nhưng vẫn phải bump khi upstream ra bản vá; Renovate theo dõi version
  qua comment `renovate:` trong Dockerfile, checksum sửa tay trong cùng PR.
- Job cần Docker daemon (Testcontainers, buildx): GitHub mount socket sẵn; Forgejo Runner cần
  `container.docker_host: automount`; GitLab dùng DinD với runner privileged.
- Lộ trình:
  - **Q4/2026:** ADR này; `platform-ci` với `vyforge-ci` 0.1 và `ci-toolbox` 0.1.0; tham số hoá registry trong
    `platform-java`, `platform-web`, `platform-helm-charts` (GitHub vẫn là giá trị đang dùng). Xong khi mọi repo build
    được trên GitHub qua `vyforge-ci`.
  - **Q1/2027:** môi trường on-prem tham chiếu (Forgejo, Runner, Harbor, OpenBao, Renovate self-host); câu hỏi Copier
    `ci_platform: github | forgejo | gitea | gitlab` trong template; tách `platform-docs`; ký image bằng cosign +
    OpenBao. Xong khi `platform-gateway` chạy toàn bộ CI/CD trên Forgejo không cần GitHub.
  - **Q2/2027:** mọi service và thư viện chạy được trên Forgejo; adapter GitLab dùng thật khi có khách hàng GitLab;
    `copier update` qua Renovate.
  - **Q3/2027:** chart `platform-policies` (Kyverno) bắt buộc chữ ký cosign của Vyforge trên mọi cluster.

# Quy hoạch vyforge: danh mục module và lộ trình (Q4/2026 – Q4/2027)

*Báo cáo nghiên cứu, 26/10/2026. Phạm vi gồm mọi repo trong `~/forge`. Bằng chứng trích theo `repo/file:dòng` hoặc số đo
lấy trực tiếp từ mã nguồn (số dòng `src/main`, grep). Version của OSS hạ tầng kiểm qua GitHub Releases API ngày
26/10/2026. Chỗ nào chưa kiểm chứng được thì ghi rõ.*

Báo cáo song song [`2026-10-module-thong-dung.md`](2026-10-module-thong-dung.md) (nhánh `docs/research-common-modules`)
phân tích sâu từng module ứng dụng: thư viện, license, ước lượng công sức. Tài liệu này dùng kết quả đó để **chốt danh mục
module của cả hệ sinh thái**: cái nào giữ, cái nào gộp, tách, bỏ hoặc thêm mới, kèm lộ trình theo quý.

## Tóm tắt

- **Bỏ 2 repo rỗng.** `nextjs-ui` và `spring-boot-starter` được tạo ngày 28/05/2026 và chưa từng có commit. Vai trò của
  chúng đã nằm trong `platform-web` (`@vyforge/ui`) và `platform-java`. Xoá trong Q4/2026.
- **Gộp `platform-iam-server` vào repo `platform-java`.** Repo này chỉ có 171 dòng: một class `main`, một file cấu hình và
  một migration giống hệt `iam.sql` của starter. Sau khi gộp, nó thành module `platform-iam-server` trong reactor, vẫn ra
  image riêng nhưng release cùng `platform-starter-authz`. **Không gộp vào auth-server**, vì như vậy sẽ phá phương án đổi
  sang Keycloak của ADR-0008 và nguyên tắc database riêng của ADR-0010.
- **Đổi tên `platform-auth-admin` thành `platform-admin`.** Repo này đang trở thành console chung cho auth, IAM, tenant
  và IdP. UI IAM tiếp tục nằm trong package `@vyforge/iam`, không tách thành repo riêng.
- **Giữ nguyên các module thư viện, không gộp hay tách.** `platform-starter-audit` giữ riêng vì auth-server dùng audit mà
  không dùng authz. `TokenExchangeClient` giữ trong `-security`. `@vyforge/server` và `@vyforge/query` giữ riêng. Hai việc
  cần làm: hợp nhất bảng audit của IAM vào `-audit`, và cho các package `platform-web` **release cùng một version**.
- **Thêm 12 starter Java (10 cái P0–P1, 2 cái P2), 4 package web, 2 chart và 3 template/repo hạ tầng.**
  - P0 (Q4/2026 – Q1/2027): `-scheduling`, `-ratelimit`, `-privacy`, `-notification`, `-files`, `template-infra`, chart
    `platform-observability`, repo `platform-infra`.
  - P1 (Q1–Q2/2027): `-export`, `-payment`, `-webhooks`, `-features`, `-realtime`, chart `platform-policies`.
  - P2 (khi có dự án cần): `-ai`, `template-expo`, `-einvoice`.
- **Không tự xây:** máy chủ thông báo, search engine, workflow engine, feature-flag server, secret store, CMS, dev portal
  Backstage, service mesh, `platform-mongo-migrations`. Dùng OSS hoặc SaaS, hoặc chưa làm (mục 1c).
- **Số repo.** Đầu kỳ có 12 repo, cuối 2027 còn 12 repo đang hoạt động: bớt 3, thêm 3 (`template-infra`, `platform-infra`,
  `template-expo`). Không chuyển sang một monorepo duy nhất. Code đã gom theo ngôn ngữ; các service tách repo vì cần image,
  quyền truy cập và nhịp release riêng. Mức trưởng thành theo CNCF hiện ở **giữa mức 1 và mức 2** (mục 2).

## 1. Quy hoạch danh mục module: thêm, giữ, gộp, tách, bỏ

Nguyên tắc ra quyết định:
1. Chỉ tạo **artifact riêng** khi phụ thuộc ngoài, nhịp release hoặc người dùng khác nhau. Nếu chỉ là ít dòng code thì chưa
   đủ lý do.
2. Chỉ tạo **repo riêng** khi cần image riêng, quyền truy cập riêng (code bảo mật) hoặc vòng đời riêng.
3. Đội có một người nên phải **bớt đơn vị phải release** ở chỗ nào được.
4. Mọi module mới là **lớp mỏng trên OSS** (theo tab 5). Engine thì không tự viết.

### 1a. Module hiện có

#### Repo

| Repo | Quyết định | Lý do (bằng chứng) | Hành động · quý |
|---|---|---|---|
| `.github` | **GIỮ** | Workflow reusable `@v1` (7 file), preset Renovate, ADR. Là điểm phân phối chuẩn cho mọi repo | Thêm `copier-update.yml`, `openapi-sdk.yml`, `load-test.yml`; ký cosign trong `container.yml` · Q1–Q2/2027 |
| `platform-java` | **GIỮ** (monorepo Java) | Reactor 10 module, release đồng bộ một version (`pom.xml:20-31`) | Nhận thêm module `platform-iam-server` và 12 starter mới (1b) · từ Q4/2026 |
| `platform-web` | **GIỮ** (monorepo TS); **đổi cách version** | 11 package, mỗi package một version riêng (0.1.0–0.5.0). Template phải ghim 9 version khác nhau (`template-nextjs/template/package.json.jinja:34-42`) và đã lệch (`ui` 0.3.0 trong khi bản mới nhất là 0.4.0) | Release **lockstep**: mọi package cùng một version từ 1.0.0 · Q2/2027 |
| `platform-helm-charts` | **GIỮ** | Chart `app` dùng cho mọi service. `Chart.yaml:6` ghi 0.1.0, lệch với tag v0.2.0 | Sửa version khi phát hành; thêm chart `platform-observability` (Q1/2027) và `platform-policies` (Q2/2027) |
| `template-spring-boot` | **GIỮ** | Golden path backend; Copier cần repo/tag riêng để `copier update` và manager `copier` của Renovate hoạt động | Chuyển sang auth-server và `platform-*` 0.9 (Q4/2026); thêm câu hỏi cho starter mới (Q1/2027) |
| `template-nextjs` | **GIỮ** | Golden path frontend | Như trên; thêm `include_admin` (Q2/2027) |
| `platform-auth-server` | **GIỮ** (repo riêng) | 9.787 dòng; là code bảo mật nhạy cảm, cần CODEOWNERS ≥ 2 người (ADR-0008) và lịch pentest riêng | Tenant/danh bạ, IdP mở rộng (Q4/2026); thay `RateLimitFilter` riêng bằng `-ratelimit` (Q1/2027); 1.0 sau pentest (Q2/2027) |
| `platform-iam-server` | **CHUYỂN repo** vào `platform-java`, archive repo cũ | Chỉ 171 dòng (`IamServerApplication.java`, `application.yml`, `V001__iam.sql`). `V001__iam.sql` **giống hệt** `platform-starter-authz/.../iam.sql` (đã diff), tức đang phải đồng bộ tay. Nó luôn phải release cùng starter | Module `platform-java/platform-iam-server`, image `ghcr.io/vyforge/platform-iam-server` giữ nguyên tên; migration lấy thẳng từ starter · Q1/2027, sau khi merge nhánh `fix/authz-hardening` |
| `platform-gateway` | **GIỮ** (repo riêng) | Theo release train của Spring Cloud (2025.1 → 2026.0), BOM của `platform-java` không quản lý. Nhỏ (569 dòng) nhưng phụ thuộc khác hẳn | Thay rate limit trong bộ nhớ bằng `-ratelimit` Redis (Q1/2027); Spring Cloud 2026.0 cùng Boot 4.2 (Q3/2027) |
| `platform-auth-admin` | **ĐỔI TÊN** thành `platform-admin` | Đã có trang users, roles, clients, sign-ins, activity, settings (`src/app/[locale]/`). Đợt này đang thêm khu Phân quyền (IAM), tenant/danh bạ và IdP, nên nó không còn là "auth admin" | Đổi tên repo; GitHub tự chuyển hướng URL cũ. Image mới `platform-admin` · Q1/2027 |
| `nextjs-ui` | **BỎ** (xoá) | Repo rỗng (`isEmpty: true`, private, tạo ngày 28/05/2026). Vai trò thư viện UI đã thuộc `@vyforge/ui` | Xoá sau khi người phụ trách xác nhận · Q4/2026 |
| `spring-boot-starter` | **BỎ** (xoá) | Repo rỗng, tạo cùng ngày. Vai trò starter đã thuộc `platform-java` | Xoá · Q4/2026 |

#### Module của `platform-java`

| Module | Cỡ (dòng main/test) | Quyết định | Lý do | Hành động · quý |
|---|---|---|---|---|
| `platform-bom`, `platform-parent` | — | **GIỮ** | Điểm ghim version duy nhất (B1) | BOM quản lý thêm các starter mới |
| `platform-starter-web` | 1436/726 | **GIỮ** | Problem Details, request id, `RestClient`: mọi service dùng | — |
| `platform-starter-security` | 576/510 | **GIỮ**; **không tách** `TokenExchangeClient` | `TokenExchangeClient` chỉ có 141 dòng và đã là tuỳ chọn: chỉ bật khi có `spring-security-oauth2-client` (`PlatformSecurityAutoConfiguration.java:103-128`). Tách ra sẽ thêm một artifact phải release mà không bớt được phụ thuộc nào | Đổi tên trong ADR-0010 cho khớp code (`TokenExchangeInterceptor` → `TokenExchangeClient#interceptor`) · Q4/2026 |
| `platform-starter-jpa` | 253/126 | **GIỮ**, mở rộng | Nhỏ nhưng là ranh giới phụ thuộc (JPA, Flyway) | Thêm `LocalizedText`, `@SoftDeletable`, `unaccent` + trigram, converter mã hoá cột. Đưa `AesGcmKeyCipher` của auth-server lên đây · Q1/2027 |
| `platform-starter-messaging` | 529/331 | **GIỮ** | Nền của notification, webhooks, payment (tab 5) | — |
| `platform-starter-observability` | 104/86 | **GIỮ** | Ít dòng nhưng mang phụ thuộc OpenTelemetry; gateway và mọi service đều dùng | Thêm quy ước tên metric khớp chart `platform-observability` · Q1/2027 |
| `platform-starter-authz` | 4103/1503 | **GIỮ** một artifact; **sửa phụ thuộc** | Không tách được rõ ràng: app ở chế độ `local` cần cả engine lẫn API quản trị. Nhưng `spring-boot-starter-jdbc` đang không optional (`pom.xml:29`), khiến service chế độ `remote` bị kéo theo DataSource | Đặt `jdbc` thành optional; nạp migration bằng Flyway (`classpath:db/migration/iam`) thay cho copy tay `iam.sql`; thêm metric · Q4/2026 – Q1/2027 |
| `platform-starter-audit` | 719/363 | **GIỮ riêng, không gộp vào authz** | `platform-auth-server` dùng `-audit` (`PlatformAuditStore.java`) mà **không** dùng authz. Gộp sẽ kéo IAM vào auth-server. Hiện có **hai bảng audit**: `audit_log` của starter và `iam_audit_log` (`iam.sql:74-85`) | Hợp nhất: khi có cả hai starter, authz phát event và `-audit` ghi lại; bỏ `iam_audit_log` ở bản 1.0. Không thêm phụ thuộc authz → audit, để tránh vòng phụ thuộc (audit đã phụ thuộc tuỳ chọn vào authz cho `AuditLogController`) · Q2/2027 |
| `platform-test` | 599/455 | **GIỮ**, mở rộng | — | `TestDataFactory` (Datafaker locale `vi`), luật ArchUnit `@PersonalData` · Q1/2027 |
| *(mới)* `platform-iam-server` | 171 | **CHUYỂN vào** | Xem bảng repo | Q1/2027 |

#### Package của `platform-web`

| Package | Cỡ (dòng src) | Quyết định | Lý do | Hành động · quý |
|---|---|---|---|---|
| `tsconfig`, `eslint-config`, `prettier-config`, `test-config` | 0 (chỉ cấu hình) | **GIỮ** riêng | Công cụ tìm cấu hình theo tên package (`extends` của tsconfig, khoá `prettier` trong package.json). Mỗi package có peer dependency khác nhau (eslint, prettier, vitest/playwright) | Chỉ đổi sang version lockstep |
| `ui` | 1582 | **GIỮ**, mở rộng | Dùng ở mọi app | `DataTable`, `ResourceForm`, `PermissionGate` · Q2/2027 |
| `api-client` | 328 | **GIỮ** | Client **server-only** gắn token ở server (B13) | — |
| `auth` | 357 | **GIỮ**; **tách core** khi có mobile | Hiện gắn với Better Auth và Next.js. Mobile cần phần dùng chung (đọc claim, `can()`) mà không kéo Next theo | Tách entry `@vyforge/auth/core` (cùng package, không tạo package mới) · Q3/2027, cùng `template-expo` |
| `i18n` | 136 | **GIỮ** | Dùng ở mọi app, mobile dùng lại phần messages | — |
| `iam` | 1347 | **GIỮ** (không tách thành app riêng) | Là thư viện UI. `template-nextjs` (`use_iam`) và `platform-admin` dùng lại; app riêng cho IAM sẽ thành console thứ hai | — |
| `query` | 80 | **GIỮ** riêng | Hiện chưa app nào dùng: không có trong `platform-auth-admin/package.json` và `template-nextjs/template/package.json.jinja`. Nhưng đây là nơi đặt code **phía client** (TanStack, `useEventStream`, inbox thông báo theo tab 5). Gộp vào `api-client` (server-only) sẽ trộn bundle server và client | Thêm `useEventStream` (Q2/2027); `@vyforge/notifications` dùng nó |
| `server` | 117 | **GIỮ** riêng | Cả hai app hiện có đều dùng (`createEnv`, `healthRoute`); không phụ thuộc React | Thêm logger JSON và OTel cho Next · Q1/2027 |

### 1b. Module mới cần thêm

Chi tiết thư viện, license và công sức nằm ở báo cáo tab 5 (mục 3 và mục 6). Bảng dưới chốt tên, loại, quý và phụ thuộc.

| Module đề xuất | Loại | Bài toán | Tự viết / OSS | Ưu tiên · quý | Phụ thuộc |
|---|---|---|---|---|---|
| `platform-starter-scheduling` | thư viện | Job định kỳ chạy đúng một lần khi có nhiều replica | Lớp mỏng trên **ShedLock 7.10** (Apache-2.0) | P0 · Q4/2026 | `-jpa` |
| `platform-starter-ratelimit` | thư viện | Rate limit dùng chung giữa các replica. Hiện gateway và auth-server mỗi nơi tự viết một bộ lọc chạy trong bộ nhớ | **Bucket4j 8.21** + Redis (lettuce) | P0 · Q4/2026 | `-web`, Redis |
| `platform-starter-privacy` + `@vyforge/privacy` | thư viện + package | Luật 91/2025 và NĐ 356/2025: `@PersonalData`, masking, consent, yêu cầu của chủ thể dữ liệu (xuất/xoá) | Tự viết (không có OSS cho luật VN) | P0 · Q4/2026 – Q1/2027 | `-jpa`, `-security`, `-audit`, `-scheduling` |
| `platform-starter-notification` + `@vyforge/notifications` | thư viện + package | Email, in-app, Zalo ZNS; SMS/push qua adapter; template; outbox | Lớp mỏng: Angus Mail, adapter ZNS, Firebase Admin cho push. **Không** tự host Novu | P0 · Q4/2026 – Q1/2027 | `-messaging`, `-jpa`, `-privacy` |
| `platform-starter-files` + `@vyforge/files` | thư viện + package | Upload/download qua presigned URL, quét virus, ảnh | **AWS SDK v2 S3**; SeaweedFS khi chạy local; ClamAV tuỳ chọn. Không dùng MinIO (đã archive) | P0 · Q4/2026 – Q1/2027 | `-web`, `-authz`, `-messaging` |
| `template-infra` | template (repo mới) | Repo `<du-an>-infra`: Argo CD app, values cho chart `app`, overlay dev/staging/prod, ESO | Argo CD **3.5.6**, Kargo **1.12.4**, ESO **2.14.0** | P0 · Q4/2026 – Q1/2027 | chart `app`, `gitops-bump` |
| `platform-infra` | repo GitOps (mới) | Triển khai chính các service nền tảng (auth, iam, gateway, admin) và các operator dùng chung | Sinh từ `template-infra`; CloudNativePG **1.30.1** + Barman Cloud Plugin | P0 · Q1/2027 | `template-infra` |
| `platform-observability` | chart | OTel Collector + LGTM, dashboard và alert theo SLO đóng gói sẵn | Grafana/Loki/Tempo/Mimir, Alloy, Sloth (theo tab 5) | P0 · Q1/2027 | `-observability` |
| `platform-starter-export` | thư viện | Xuất Excel/PDF lớn chạy bất đồng bộ, có link presigned | FastExcel/POI, OpenHTMLtoPDF (LGPL, cần pháp chế duyệt) | P1 · Q1/2027 | `-scheduling`, `-files`, `-notification` |
| `platform-starter-webhooks` | thư viện | Gửi webhook ra ngoài có ký HMAC, retry, idempotent | **Standard Webhooks** (Java) | P1 · Q1/2027 | `-messaging`, `-jpa` |
| `platform-starter-features` + `@vyforge/features` | thư viện + package | Feature flag theo tenant hoặc gói dịch vụ | **OpenFeature** SDK + **flagd** | P1 · Q1/2027 | `-security` |
| `platform-starter-payment` | thư viện | VNPay, MoMo, VietQR (payOS/SePay); ZaloPay sau; IPN kiểm chữ ký, đối soát | Tự viết adapter (SDK chính thức không dùng được trực tiếp) | P1 · Q1–Q2/2027 | `-webhooks`, `-scheduling`, `-audit` |
| `platform-starter-realtime` | thư viện | SSE cho inbox thông báo, tiến độ export, trạng thái thanh toán | Spring SSE + Redis pub/sub. Chỉ dùng Centrifugo khi có >10 nghìn kết nối | P1 · Q2/2027 | `-security`, Redis |
| `platform-policies` | chart | Kyverno: kiểm chữ ký cosign, bắt buộc digest, non-root. Chạy chế độ Audit trước, sau đó Enforce | **Kyverno 1.19.2** `ImageValidatingPolicy`, **cosign 3.1.4** | P1 · Q2/2027 (Enforce ở Q3) | `container.yml` có ký image |
| `.github`: `copier-update.yml`, `openapi-sdk.yml`, `load-test.yml` | workflow | Đồng bộ khung Copier; sinh SDK từ OpenAPI; mẫu chạy k6 | Copier 9.19, OpenAPI Generator, k6 | P0/P1 · Q1–Q2/2027 | — |
| `platform-starter-ai` | thư viện | Gọi LLM qua Spring AI 2.0, pgvector, chặn PII trước khi gửi ra ngoài | Spring AI (Apache-2.0) | P2 · khi có dự án | `-privacy`, `-ratelimit` |
| `template-expo` | template (repo mới) | Golden path mobile: OIDC native app (PKCE, public client), gọi API qua gateway | **Expo SDK 58** (npm `expo@58.0.19`, React Native 0.88) | P2 · Q3/2027 | `@vyforge/auth/core`, gateway |
| `platform-starter-einvoice`, adapter eKYC | thư viện | Hoá đơn điện tử, eKYC theo nhà cung cấp mà khách hàng đã có hợp đồng | Adapter mỏng | P2 · khi có hợp đồng | `-files`, `-audit` |
| Workflow/BPM | (chưa có module) | Quy trình duyệt nhiều bước | Flowable 8 (Apache-2.0) hoặc Temporal; **chỉ viết ADR**, chưa làm starter | P2 · khi có dự án | — |

Tìm kiếm không cần module riêng. Mặc định dùng PostgreSQL với `unaccent` và trigram, thêm vào `-jpa`. Secrets cũng không
cần module code: dùng ESO + OpenBao 2.7.1 hoặc secret manager của cloud, khai báo trong `template-infra`.

### 1c. Không nên làm

| Không làm | Thay bằng | Lý do |
|---|---|---|
| Máy chủ thông báo riêng (`platform-notification-server`), hoặc tự host Novu | `-notification` chạy trong app, dùng outbox | Thêm một service phải trực on-call cho một nhu cầu mà outbox + adapter đã giải được. Phần `enterprise/` của Novu có license thương mại |
| Search engine riêng, hoặc chạy Elasticsearch mặc định | PostgreSQL FTS; Meilisearch hoặc OpenSearch khi có ADR | Phần lớn dự án tier 0–1 không cần |
| Workflow engine tự viết; Camunda 8 | Flowable hoặc Temporal khi có dự án cần | Camunda 8 cần license thương mại cho production (tab 5) |
| Feature-flag server tự viết; Unleash tự host | flagd (OpenFeature) | Unleash server dùng AGPL; flagd chỉ đọc file hoặc ConfigMap, vận hành nhẹ |
| Secret store tự viết; Vault | OpenBao hoặc secret manager của cloud, qua ESO | Vault đã chuyển sang BUSL-1.1 |
| Spring Cloud Config server | Biến môi trường + ESO | Thêm một service mà không có lợi gì so với GitOps |
| Backstage (trong 2027) | Site tài liệu tĩnh từ `.github/docs` + README | Chỉ đáng khi có khoảng 5 team cần self-service. Backstage 1.56 vẫn cần một người lo riêng |
| CMS nhúng trong backend, framework admin chung | Payload CMS (MIT) khi dự án cần; khối admin trong `@vyforge/ui` | Admin của mỗi dự án quá khác nhau để làm thành framework |
| `platform-mongo-migrations` (F12) | Bỏ khỏi kế hoạch | ADR-0007 đã chọn PostgreSQL; chỉ làm lại khi một dự án MongoDB cũ thực sự gặp lỗi Mongock |
| Gateway làm BFF (phương án B của ADR-0006) | Next.js BFF như hiện tại | Không có dự án nào yêu cầu; gateway đã nhận vai trò khác (ADR-0010) |
| Service mesh, mTLS/SPIFFE | `client_credentials` và token exchange | Chưa có tier 3 |
| LLM gateway riêng | Spring AI trong `-ai`; LiteLLM khi cần quản lý nhiều nhà cung cấp | Nhu cầu chưa rõ |
| SDK native iOS/Android | `template-expo` | Một đội nhỏ không duy trì nổi ba nền tảng |
| Monorepo duy nhất cho toàn org | Giữ hai monorepo theo ngôn ngữ + các repo service | Auth-server cần quyền truy cập và lịch pentest riêng. Copier cần tag riêng cho từng template. Image và release của service độc lập |

### 1d. Bản đồ module đích cuối 2027

Ký hiệu: **[có]** giữ nguyên; **[mới]** thêm; **[gộp]** chuyển hoặc hợp nhất vào chỗ khác; **[đổi tên]**; **[bỏ]**
xoá hoặc archive.

| Tầng | Module |
|---|---|
| Golden path | `template-spring-boot` [có] · `template-nextjs` [có] · `template-infra` [mới] · `template-expo` [mới] |
| Thư viện Java (`platform-java`) | `platform-bom` [có] · `platform-parent` [có] · `-web` [có] · `-security` [có] · `-jpa` [có] · `-messaging` [có] · `-observability` [có] · `-authz` [có] · `-audit` [có] · `platform-test` [có] · `-scheduling` [mới] · `-ratelimit` [mới] · `-privacy` [mới] · `-notification` [mới] · `-files` [mới] · `-export` [mới] · `-webhooks` [mới] · `-features` [mới] · `-payment` [mới] · `-realtime` [mới] · `-ai` [mới, P2] · `-einvoice` [mới, P2] · `platform-iam-server` [gộp từ repo riêng] |
| Package web (`platform-web`, version lockstep) | `tsconfig` · `eslint-config` · `prettier-config` · `test-config` · `ui` · `api-client` · `auth` (+ `/core`) · `i18n` · `iam` · `query` · `server` [có] · `privacy` · `files` · `notifications` · `features` [mới] |
| Service nền tảng | `platform-auth-server` [có] · `platform-gateway` [có] · `platform-admin` [đổi tên từ `platform-auth-admin`] · `platform-iam-server` [gộp vào `platform-java`] |
| Chart (`platform-helm-charts`) | `app` [có] · `platform-observability` [mới] · `platform-policies` [mới] |
| GitOps / hạ tầng | `platform-infra` [mới] · `<du-an>-infra` (sinh từ `template-infra`) [mới] |
| CI (`.github`) | `java-ci` · `node-ci` · `container` (+ cosign) · `gitops-bump` · `maven-publish` · `npm-publish` · `helm-publish` [có] · `copier-update` · `openapi-sdk` · `load-test` [mới] |
| Bỏ | `nextjs-ui` [bỏ] · `spring-boot-starter` [bỏ] · repo `platform-iam-server` [bỏ sau khi gộp] · bảng `iam_audit_log` [bỏ ở 1.0] · `RateLimitFilter` riêng trong gateway và auth-server [bỏ, thay bằng `-ratelimit`] |

Cạnh phụ thuộc (`A -> B`: A dùng hoặc phụ thuộc B; cạnh có `(opt)` là tuỳ chọn). Phiên điều phối vẽ sơ đồ từ danh sách này.

```
# Tầng golden path
template-spring-boot [có] -> platform-parent
template-spring-boot [có] -> platform-starter-{web,security,jpa,observability} ; platform-test
template-spring-boot [có] -> platform-starter-{authz,audit} (opt, use_iam)
template-spring-boot [có] -> platform-starter-messaging (opt, use_kafka)
template-spring-boot [có] -> platform-starter-{scheduling,ratelimit,privacy} [mới]
template-spring-boot [có] -> platform-starter-{notification,files,export,payment,webhooks,features,realtime,ai} [mới] (opt, câu hỏi Copier)
template-spring-boot [có] -> platform-auth-server            # thay Keycloak trong compose
template-nextjs [có] -> @vyforge/{ui,api-client,auth,i18n,server} ; @vyforge/iam (opt)
template-nextjs [có] -> @vyforge/{privacy,files,notifications,features} [mới]
template-nextjs [có] -> platform-auth-server
template-expo [mới] -> @vyforge/{i18n,query} ; @vyforge/auth/core [mới]
template-expo [mới] -> platform-auth-server ; platform-gateway
template-infra [mới] -> platform-helm-charts:app ; platform-observability [mới] ; platform-policies [mới]
template-infra [mới] -> argo-cd@3.5 ; kargo@1.12 ; external-secrets@2.14 ; openbao@2.7 (opt)
platform-infra [mới] -> template-infra [mới]
platform-infra [mới] -> platform-auth-server ; platform-iam-server ; platform-gateway ; platform-admin
platform-infra [mới] -> cloudnative-pg@1.30 ; barman-cloud-plugin

# Tầng thư viện Java (platform-java)
platform-parent -> platform-bom
platform-bom -> {mọi platform-starter-*, platform-test}
platform-starter-authz -> platform-starter-web ; platform-starter-security
platform-starter-audit -> platform-starter-authz (opt, chỉ cho AuditLogController)
platform-starter-authz -> event "IamChanged" -> platform-starter-audit (opt, qua ApplicationEvent, không phụ thuộc compile)
platform-starter-messaging -> platform-starter-web (opt)
platform-starter-scheduling [mới] -> platform-starter-jpa
platform-starter-ratelimit [mới] -> platform-starter-web ; redis
platform-starter-privacy [mới] -> platform-starter-{jpa,security,audit} ; platform-starter-scheduling [mới]
platform-starter-notification [mới] -> platform-starter-{messaging,jpa,security} ; platform-starter-privacy [mới]
platform-starter-files [mới] -> platform-starter-{web,authz,messaging,jpa} ; s3 ; clamav (opt)
platform-starter-export [mới] -> platform-starter-{scheduling,files,notification} [mới]
platform-starter-webhooks [mới] -> platform-starter-{messaging,jpa,authz}
platform-starter-features [mới] -> platform-starter-security ; flagd
platform-starter-payment [mới] -> platform-starter-{webhooks,scheduling} [mới] ; platform-starter-{messaging,jpa,audit}
platform-starter-realtime [mới] -> platform-starter-{security,authz} ; redis
platform-starter-ai [mới] -> platform-starter-{privacy,ratelimit} [mới] ; platform-starter-observability
platform-starter-einvoice [mới] -> platform-starter-{files} [mới] ; platform-starter-audit
platform-iam-server [gộp] -> platform-starter-{authz,web,security,jpa,observability} ; postgresql ; redis

# Tầng package web (platform-web)
@vyforge/iam -> @vyforge/{api-client,ui}
@vyforge/privacy [mới] -> @vyforge/{ui,api-client}
@vyforge/files [mới] -> @vyforge/{ui,api-client}
@vyforge/notifications [mới] -> @vyforge/{ui,api-client,query}
@vyforge/features [mới] -> @vyforge/auth
@vyforge/* -> @vyforge/tsconfig (dev)

# Tầng service nền tảng
platform-auth-server [có] -> platform-parent ; platform-starter-{web,jpa,audit,observability}
platform-auth-server [có] -> platform-starter-ratelimit [mới]   # thay RateLimitFilter riêng
platform-auth-server [có] -> platform-starter-notification [mới] (opt, thay ThymeleafAccountMailer)
platform-gateway [có] -> platform-starter-{security,web,observability} ; platform-starter-ratelimit [mới]
platform-gateway [có] -> platform-auth-server (JWKS) ; platform-iam-server (route /api/v1/iam/**)
platform-admin [đổi tên] -> @vyforge/{ui,api-client,auth,i18n,server,iam}
platform-admin [đổi tên] -> platform-auth-server (OIDC + /api/v1/admin/**, /api/v1/directory/**)
platform-admin [đổi tên] -> platform-iam-server (/api/v1/iam/**)
app(authz remote) -> platform-iam-server ; redis vyforge:iam:changed
app(RestClient) -> platform-auth-server (token exchange)

# Tầng chart và hạ tầng
platform-observability [mới] -> otel-collector/alloy ; loki ; tempo ; mimir ; grafana ; sloth
platform-policies [mới] -> kyverno@1.19 ; cosign@3 (khoá KMS/OpenBao)
platform-helm-charts:app -> external-secrets (opt) ; servicemonitor (opt)

# Tầng CI (.github)
platform-java ; platform-web ; platform-helm-charts -> .github:{java-ci,node-ci,maven-publish,npm-publish,helm-publish}
platform-auth-server ; platform-gateway ; platform-admin ; platform-java(platform-iam-server) -> .github:container ; .github:gitops-bump -> platform-infra
.github:container -> trivy ; sbom+provenance ; cosign [mới]
.github:copier-update [mới] -> template-{spring-boot,nextjs,infra,expo}
.github:openapi-sdk [mới] -> openapi-generator
.github:load-test [mới] -> k6

# Bỏ
nextjs-ui [bỏ] -x
spring-boot-starter [bỏ] -x
repo platform-iam-server [bỏ] -> platform-java/platform-iam-server
```

## 2. Hiện trạng và mức trưởng thành (rút gọn)

Bản 1.0 của [CNCF Platform Engineering Maturity Model](https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/)
chấm 5 chiều theo 4 mức (Provisional, Operationalized, Scalable, Optimizing). Kết quả của vyforge:

| Chiều | Mức | Bằng chứng chính |
|---|---|---|
| Investment | 1 | API `orgs/vyforge/members` trả về 1 thành viên; CODEOWNERS ghi "thay bằng team owner khi đã lập" |
| Adoption | 1 | Có 20 quy tắc bắt buộc, nhưng chưa repo dự án nào sinh từ template |
| Interfaces | 2 | Copier, 7 workflow `@v1`, artifact có version. Chưa có self-service hạ tầng |
| Operations | 1–2 | Thư viện có semver; 4 service chưa có tag; template dùng `platform-parent` 0.7.0 trong khi bản mới nhất là 0.8.0; không có SLO, runbook, backup |
| Measurement | 1 | Không đo gì |

Điểm mạnh nên giữ:
- Thiết kế hai lớp: artifact có version, cộng khung Copier mỏng.
- Ranh giới module được công cụ kiểm tra.
- Action ghim theo SHA, có SBOM và Trivy.
- Bộ IdP/IAM khá đầy đủ so với quy mô đội.

DORA cũng cảnh báo đường cong chữ J: hiệu suất có thể giảm khi platform mới thêm độ phức tạp
([DORA](https://dora.dev/capabilities/platform-engineering/)). Vì vậy mục 1 ưu tiên bớt đơn vị phải release trước khi thêm
module mới.

## 3. Kiến trúc đích 12–24 tháng

### 3.1 Golden path

| Đường | Hiện có | Đích |
|---|---|---|
| Backend | `template-spring-boot` v1.4.0, compose vẫn dùng Keycloak | Dùng auth-server; câu hỏi `iam_mode`; câu hỏi cho các starter mới; sinh sẵn values cho repo infra |
| Frontend | `template-nextjs` v1.3.0, provider Keycloak | Provider `vyforge`; `@vyforge/*` lockstep; tuỳ chọn gọi qua gateway |
| Mobile | Chưa có | `template-expo` (Expo SDK 58): OIDC native app, token trong secure storage, gọi qua gateway. Cần ADR vì khác mô hình BFF của ADR-0006 |
| Hạ tầng | Chưa có (F1) | `template-infra` sinh `<du-an>-infra` |

### 3.2 Runtime và GitOps

- **Kubernetes:** bám hai minor được hỗ trợ gần nhất. 1.35 hết hạn 28/02/2027, 1.36 hết hạn 28/06/2027
  ([kubernetes.io](https://kubernetes.io/releases)), nên phải nâng 2 lần mỗi năm. Ưu tiên K8s managed; on-prem dùng K3s.
- **GitOps:** Argo CD 3.5.6 (3.2 đã hết hỗ trợ ngày 04/08/2026, [endoflife.date](https://endoflife.date/argo-cd)). Kargo
  1.12.4 để promote cùng một digest dev → staging → prod. `gitops-bump` hiện chưa repo nào gọi
  (`platform-gateway/.github/workflows/cd.yml:27`); sẽ bật khi có `platform-infra`.
- **Preview environment theo PR:** ApplicationSet với
  [Pull Request generator](https://oneuptime.com/blog/post/2026-02-09-argocd-pr-preview-environments/view), PR có nhãn
  `preview`, mỗi PR một namespace, có TTL và quota
  ([Signadot](https://www.signadot.com/articles/argocd-preview-environments/) nêu giới hạn khi số PR lớn).
- **Helm 4:** Helm 3 hết vá bảo mật từ 11/11/2026 (research cũ).

### 3.3 Danh tính và quyền

- **`platform-auth-server`:** 1.0 sau pentest. Tenant và danh bạ theo ADR-0011, IdP mở rộng theo ADR-0012 (đang làm).
  Keycloak vẫn là phương án dự phòng. Lưu ý Keycloak **không có LTS** (bản mới nhất 26.7.0, 07/2026,
  [Skycloak](https://skycloak.io/blog/keycloak-25-released-new-features-and-enhancements/)).
- **IAM:** sửa authz (0.9.0), bắt buộc claim tenant, lọc danh sách theo policy ở truy vấn.
- **Gateway:** rate limit qua Redis; route để `/me/evaluate` chạy trong app.

### 3.4 Dữ liệu

- PostgreSQL 18 dùng [CloudNativePG 1.30](https://www.postgresql.org/about/news/cloudnativepg-1300-released-3337/)
  (bản 1.30.1, 23/09/2026) hoặc bản managed của cloud.
- Backup và PITR dùng [Barman Cloud Plugin](https://cloudnative-pg.io/plugin-barman-cloud/docs/intro/); phần backup
  có sẵn trong core đã deprecated. Diễn tập restore mỗi quý.
- Migration Flyway theo kiểu expand/contract; quy ước dải số migration cho các nhánh chạy song song.
- Phương án DR phải sao lưu cả master key mã hoá JWK của auth-server.

### 3.5 Messaging, observability, chuỗi cung ứng, secrets, chi phí

- **Kafka 4.x** chỉ còn chế độ KRaft; chạy bằng Strimzi; chỉ dùng cho tier ≥ 2.
- **Observability:** OTel Collector đẩy vào LGTM, đóng gói trong chart `platform-observability`.
- **Chuỗi cung ứng:** theo [SLSA v1.2](https://slsa.dev/blog/2025/11/announce-slsa-v1.2) (11/2025, thêm Source Track).
  - GitHub artifact attestation **không dùng được cho repo private** nếu không có gói Enterprise Cloud
    ([GitHub docs](https://docs.github.com/de/actions/how-tos/security-for-github-actions/using-artifact-attestations/using-artifact-attestations-to-establish-provenance-for-builds));
    `container.yml:125-129` đang tắt bước này.
  - Thay bằng ký **cosign 3.1.4** với khoá trong KMS/OpenBao. Không dùng keyless trên Rekor công khai, vì tên repo sẽ lộ
    ra log minh bạch.
  - Admission bằng **Kyverno 1.19.2** `ImageValidatingPolicy`, đã stable ở `policies.kyverno.io/v1`
    ([docs](https://kyverno.io/docs/policy-types/overview/)).
  - Mục tiêu: Build L2 trong Q1/2027, hướng tới L3.
- **Secrets:** ESO 2.14.0 (API `v1`) với OpenBao 2.7.1 hoặc secret manager của cloud.
- **FinOps:** [OpenCost](https://www.cncf.io/blog/2024/10/31/opencost-advances-to-the-cncf-incubator/); nhãn
  `vyforge.io/project|env|tier` bắt buộc trong chart `app`; showback theo dự án khách hàng.

## 4. Năng lực ngoài code (rút gọn)

- **Version và deprecation (ADR-0013).**
  - `platform-java`, `platform-web` và auth-server lên **1.0.0** khi pilot tier 1 chạy production.
  - Mỗi minor của Spring Boot ứng với một dòng release. Boot ra minor vào tháng 5 và tháng 11, mỗi bản được hỗ trợ
    [≥ 12 tháng](https://github.com/spring-projects/spring-boot/wiki/Supported-Versions);
    [4.2.0-M1](https://spring.io/blog/2026/08/20/spring-boot-4-2-0-M1-available-now/) đã có.
  - Hỗ trợ đồng thời 2 dòng; không có LTS dài hơn thời gian OSS của Boot.
  - Deprecate trước ít nhất một minor, kèm CHANGELOG tự sinh và recipe OpenRewrite hoặc codemod.
  - Ma trận tương thích: `platform-java` 1.x ↔ Boot 4.1 / Spring Cloud 2025.1 / Java 25; 2.x ↔ Boot 4.2 / 2026.0.
    `platform-web` ↔ Next ≥ 16.4.1, React 19, Node 24 LTS (Node 26 lên LTS khoảng 28/10/2026;
    [từ v27 phát hành mỗi năm một bản](https://nodejs.org/en/about/releases/)).
- **Nâng cấp hàng loạt (ADR-0014).**
  - Renovate với nhóm "platform" cho `com.vyforge.platform:*` và `@vyforge/*`.
  - Khung Copier: dùng [manager `copier`](https://docs.renovatebot.com/modules/manager/copier/) của Renovate. Manager
    này cần Renovate self-host với `allowScripts`, vì template của vyforge dùng `--trust`. Nếu không thì dùng workflow
    `copier-update.yml`.
  - Bảng "fleet" đọc `.copier-answers.yml` và version của từng repo.
  - Mục tiêu: 90% repo dùng dòng `platform-*` còn được hỗ trợ; bản vá bảo mật lan tới mọi repo trong 7 ngày.
- **Đo lường.**
  - DORA 2025 có 5 chỉ số; mới thêm **rework rate**
    ([dora.dev](https://dora.dev/research/2025/dora-report/), [tóm tắt](https://redmonk.com/rstephens/2025/12/18/dora2025/)).
  - Khảo sát [SPACE](https://www.microsoft.com/en-us/research/?p=733804) mỗi quý.
  - Chỉ số adoption: % repo sinh bằng Copier, % theo kịp template, time-to-first-deploy dưới 1 ngày.
  - Chấm lại mức trưởng thành CNCF 6 tháng một lần.
- **Tài liệu, hỗ trợ, SLO.**
  - Site tài liệu tĩnh; runbook cho từng service; GitHub Discussions; office hours với dự án pilot.
  - On-call chỉ trong giờ hành chính cho tới khi có người thứ 2 và thứ 3.
  - SLO 28 ngày, theo [SRE Workbook](https://oreilly.com/library/view/the-site-reliability/9781492029496):
    - auth `/oauth2/token`: 99,9%
    - iam `resolve`: 99,5% (app có stale cache)
    - gateway: 99,9%, overhead p99 < 20 ms
- **DR/BCP.** RPO/RTO theo tier (service nền tảng: 5 phút / 1 giờ). Git là nguồn sự thật để dựng lại; backup ở vùng khác;
  bản sao offline của secret gốc; mirror GHCR và Packages cho khách on-prem; diễn tập 6 tháng một lần.
- **Tuân thủ.**
  - **Luật 91/2025/QH15** và **NĐ 356/2025/NĐ-CP** có hiệu lực từ 01/01/2026. Nghị định 356 **thay thế NĐ 13/2023**
    ([Công báo](https://congbao.chinhphu.vn/van-ban/nghi-dinh-so-356-2025-nd-cp-468371.htm),
    [LuatVietnam](https://luatvietnam.vn/dan-su/diem-moi-cua-nghi-dinh-356-2025-so-voi-nghi-dinh-13-2023-ve-bao-ve-du-lieu-ca-nhan-568-106269-article.html)).
  - Các nghĩa vụ chính: lưu bằng chứng đồng ý (Điều 6); thời hạn xử lý yêu cầu của chủ thể, 2 đến 20 ngày (Điều 5); MFA
    và phân quyền khi xử lý dữ liệu lớn (Điều 9). Các nghĩa vụ này được giải bằng `-privacy`.
  - ISO/IEC 27001 chỉ còn bản 2022 (hạn chuyển đổi
    [31/10/2025](https://dnv.com/assurance/Management-Systems/new-iso/transition/transition-to-iso-iec-27001-2022)).
    Chỉ làm chứng chỉ khi hợp đồng yêu cầu.
- **Pentest.**
  - Theo [OWASP ASVS 5.0](https://owasp.org/projects/asvs?tab=main): Level 3 cho auth-server, Level 2 cho phần còn lại.
  - Làm trước lần đầu lên production, sau đó mỗi năm và sau mỗi thay đổi lớn ở luồng đăng nhập.
  - SLA sửa lỗi: critical 7 ngày, high 30 ngày.

## 5. Lộ trình theo quý (Q4/2026 – Q4/2027)

| Quý | Module **tạo mới** | Module **gộp / đổi tên / bỏ** | Việc chính khác | Tiêu chí hoàn thành | Phụ thuộc |
|---|---|---|---|---|---|
| **Q4/2026** | `platform-starter-scheduling`, `platform-starter-ratelimit`; bắt đầu `-privacy`, `-notification`, `-files`, `template-infra` | **Xoá** `nextjs-ui`, `spring-boot-starter`. `-authz`: `jdbc` optional, migration Flyway | Merge đợt hiện tại: authz 0.9.0, tenant/danh bạ (ADR-0011), IdP (ADR-0012), UI phân quyền. Tag v0.1.0 cho 4 service. Template chuyển sang auth-server. Compose E2E. ADR-0013, 0014 | 4 nhánh đã merge; E2E xanh; app sinh từ template đăng nhập được bằng auth-server | Contract chung |
| **Q1/2027** | Hoàn thành `-privacy` (+`@vyforge/privacy`), `-notification` (+`@vyforge/notifications`), `-files` (+`@vyforge/files`), `template-infra`; tạo `platform-infra`, chart `platform-observability`, `-export`, `-webhooks`, `-features` (+`@vyforge/features`), workflow `copier-update` | **Chuyển** `platform-iam-server` vào `platform-java`, archive repo cũ. **Đổi tên** `platform-auth-admin` → `platform-admin`. Gateway và auth-server bỏ `RateLimitFilter` riêng | Argo CD, CNPG, ESO chạy cho dev/staging; ký cosign; hợp đồng pentest; khởi động pilot tier 1 | Service nền tảng chạy qua GitOps; restore PITR thành công; một PR `copier update` tự động được merge | Chọn nơi chạy (ADR-0015); có người thứ 2 |
| **Q2/2027** | `-payment`, `-realtime`, chart `platform-policies` (chế độ Audit), workflow `openapi-sdk`, `load-test`; khối admin trong `@vyforge/ui` | **Hợp nhất audit**: authz ghi qua `-audit`, bỏ `iam_audit_log`. `platform-web` chuyển sang version lockstep 1.0.0 | Pentest xong → auth-server 1.0.0; `platform-java` 1.0.0; Kargo; SLO và runbook; diễn tập DR lần 1 | Pilot tier 1 chạy production ≥ 4 tuần không có sự cố P1 do platform; lần đầu đo DORA | Pentest không còn lỗi critical/high (nếu còn thì pilot dùng Keycloak) |
| **Q3/2027** | `template-expo`, entry `@vyforge/auth/core`; `-ai` nếu có dự án cần | `platform-policies` chuyển sang Enforce | Pilot tier 2 (gateway, IAM remote, Kafka). `platform-java` 2.x trên Boot 4.2 và Spring Cloud 2026.0. Dashboard DORA, khảo sát SPACE | Pilot tier 2 lên production; một app mobile đăng nhập qua auth-server; ≥ 80% repo dùng dòng còn được hỗ trợ | Kết quả Q2; nhân sự mobile |
| **Q4/2027** | `-einvoice` và workflow (Flowable/Temporal) chỉ khi có hợp đồng | Rà soát lại: module nào không có dự án dùng sau 2 quý thì đánh dấu deprecated | Chấm lại mức trưởng thành (mục tiêu: mức 2 mọi chiều, mức 3 Interfaces); pentest năm; quyết định Backstage và ISO 27001 | Mỗi service nền tảng có ≥ 2 owner; bảng fleet công khai | ≥ 3 dự án dùng platform |

## 6. ADR nên viết tiếp

ADR-0011 (tenant và danh bạ) và ADR-0012 (IdP mở rộng) đang được viết ở nhánh khác.

| ADR | Tiêu đề | Mô tả một dòng |
|---|---|---|
| 0013 | Version, release, deprecation | Semver sau 1.0; dòng release theo minor của Boot; hỗ trợ 2 dòng; `platform-web` lockstep; ma trận tương thích |
| 0014 | Nâng cấp hàng loạt | Renovate nhóm "platform", manager `copier` hoặc workflow `copier-update`, OpenRewrite/codemod, bảng fleet |
| 0015 | Danh mục repo và module | Ghi chính thức mục 1: xoá 2 repo rỗng, gộp iam-server vào `platform-java`, đổi tên `platform-admin`, tiêu chí tạo repo hoặc artifact mới |
| 0016 | Runtime đích | K8s managed / K3s on-prem; nâng 2 minor mỗi năm; tách cụm theo môi trường hoặc khách hàng |
| 0017 | GitOps, môi trường, preview | Argo CD + Kargo, repo `platform-infra` và `<du-an>-infra`, preview theo PR có TTL |
| 0018 | PostgreSQL trên K8s và migration | CloudNativePG + Barman plugin, PITR, RPO/RTO; Flyway expand/contract, dải số cho nhánh song song |
| 0019 | Secrets | ESO 2.x + OpenBao hoặc secret manager của cloud; xoay vòng |
| 0020 | Chuỗi cung ứng | SLSA Build L2→L3, Source L2; cosign khoá KMS; chart `platform-policies` (Kyverno) |
| 0021 | Observability | Chart `platform-observability`, quy ước metric/log/trace |
| 0022 | SLO, hỗ trợ, on-call | SLI/SLO cho auth, iam, gateway; error budget policy; mức phản hồi |
| 0023 | DR/BCP | RPO/RTO, secret gốc, mirror registry, diễn tập |
| 0024 | Bảo vệ dữ liệu cá nhân | Luật 91/2025 + NĐ 356/2025: phân định trách nhiệm platform/khách hàng; `-privacy` |
| 0025 | Pentest | ASVS 5.0 L3/L2, tần suất, SLA sửa lỗi |
| 0026 | Job và rate limit | ShedLock (`-scheduling`); Bucket4j + Redis (`-ratelimit`) thay cho filter riêng |
| 0027 | Thông báo | `-notification`: kênh, outbox, ZNS, không tự host Novu |
| 0028 | Lưu file | `-files`: S3 presigned, SeaweedFS khi local, không dùng MinIO, ClamAV |
| 0029 | Xuất file, webhook, feature flag | `-export` (license LGPL), `-webhooks` (Standard Webhooks), `-features` (OpenFeature + flagd) |
| 0030 | Thanh toán nội địa | `-payment`: IPN kiểm chữ ký, idempotent, đối soát |
| 0031 | Golden path mobile | `template-expo`, OIDC native app, `@vyforge/auth/core` |
| 0032 | Hợp nhất audit | authz ghi qua `-audit` bằng event; bỏ `iam_audit_log` |
| 0033 | Lọc danh sách theo policy | Dịch điều kiện policy thành điều kiện truy vấn (phần còn lại của ADR-0010) |
| 0034 | Đo lường platform | DORA 5 chỉ số, SPACE, adoption, chấm CNCF 6 tháng một lần |
| 0035 | Owner và mức hỗ trợ | ≥ 2 owner mỗi service; quy trình nhận đóng góp ngược |

## 7. Rủi ro lớn và cách giảm

| # | Rủi ro | Cách giảm |
|---|---|---|
| R1 | **Bus factor = 1**: org có 1 thành viên; ADR-0008 đòi ≥ 2 owner | Có người thứ 2 trước Q2/2027 (điều kiện để pilot lên production); bớt đơn vị phải release (mục 1); runbook và ADR đủ để người khác tiếp quản; ưu tiên dịch vụ managed |
| R2 | **Auth-server tự viết** chưa pentest (`README.md:8`); IdP mở rộng làm tăng bề mặt tấn công | Keycloak làm phương án B; pentest ASVS L3 trước 1.0; IdP mở rộng bật theo cờ, mặc định tắt; chưa đủ 2 owner trước 07/2027 thì dự án dùng Keycloak |
| R3 | **Quá nhiều repo và module cho một người** | Mục 1: bỏ 3 repo, giữ số repo hoạt động ở mức 12 dù thêm năng lực; module mới đặt trong monorepo sẵn có; `platform-web` lockstep; module không có người dùng sau 2 quý thì deprecated |
| R4 | **Xây trước khi có khách hàng** (F19) | Pilot tier 1 bắt đầu Q1/2027; module P1/P2 chỉ làm khi pilot cần |
| R5 | **Code ra nhanh hơn tốc độ review** (`platform-java` 8 minor trong 6 ngày, có commit do AI sinh; DORA 2025: AI tăng throughput nhưng giảm ổn định) | Semver nghiêm sau 1.0; đo rework rate; mỗi lỗi có test tái hiện; review bảo mật cho auth/authz |
| R6 | **Phụ thuộc GitHub** (Packages, GHCR, attestation trên repo private) | Ký bằng cosign; mirror registry cho khách on-prem; export repo định kỳ |
| R7 | **License OSS thay đổi** (MinIO đã archive, Vault chuyển BUSL, Camunda cần license) | Mỗi module mới đứng sau một port; thư viện và license ghi trong ADR (tab 5) |
| R8 | **Nhịp nâng cấp của hệ sinh thái** (Boot 6 tháng, K8s khoảng 14 tháng, Keycloak không có LTS) | Rà soát hằng quý; ma trận tương thích |

Sơ đồ quy hoạch: dùng danh sách cạnh ở mục **1d**. Danh sách này thay cho bản cũ.

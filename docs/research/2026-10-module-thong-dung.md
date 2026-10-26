> Báo cáo nghiên cứu ngày 26/10/2026. Bổ sung cho [ARCHITECTURE.md](../ARCHITECTURE.md) và [báo cáo kiến trúc template](2026-10-kien-truc-template.md). Đây là đề xuất, chưa phải quyết định: mỗi module sẽ được chốt bằng ADR riêng trước khi làm.

# Các module thông dụng mà hệ sinh thái vyforge còn thiếu

Hiện vyforge đã có phần **khung**: template Copier, `platform-parent`/BOM, các starter web/security/jpa/messaging/observability/authz/audit, package `@vyforge/*`, Helm app chart, reusable workflow. Phần **danh tính** cũng đã có: `platform-auth-server`, `platform-iam-server`, `platform-gateway`. Thứ còn thiếu là các module "việc nào dự án cũng làm lại", tức những gì ABP, JHipster, Supabase hay các SaaS starter đã đóng gói sẵn. Gồm: gửi thông báo (email, SMS, Zalo ZNS), lưu file, job định kỳ chạy an toàn khi có nhiều replica, rate limit dùng chung, xuất Excel/PDF, webhook ra ngoài, feature flag, tìm kiếm, thanh toán nội địa. Ngoài ra còn một lớp **tuân thủ** mà mọi dự án ở Việt Nam đều phải làm từ 01/01/2026, khi Luật Bảo vệ dữ liệu cá nhân 91/2025/QH15 và Nghị định 356/2025/NĐ-CP có hiệu lực. Khuyến nghị của báo cáo:

- **Không tự viết engine.** Mỗi module là một lớp mỏng trên thư viện OSS có license thoáng: ShedLock, Bucket4j, AWS SDK v2, Apache POI/FastExcel, OpenHTMLtoPDF, OpenFeature, Standard Webhooks. Lớp này chỉ thêm cấu hình mặc định, port/adapter theo chuẩn Modulith và test.
- **Đóng gói theo đúng ba lớp hiện có:** starter Java trong `platform-java`, package trong `platform-web`, và chart hoặc service riêng chỉ khi thật cần chạy tiến trình riêng.
- **Làm theo ba đợt.** P0 là những gì dự án pilot cần ngay và phải đúng luật: job, thông báo (email trước), file, rate limit Redis, bộ công cụ PDP, observability stack, `template-infra`.

> Phạm vi và độ tin cậy. Version của thư viện Java lấy trực tiếp từ `maven-metadata.xml` trên Maven Central, package JS từ registry npm, server OSS từ GitHub Releases API. Tất cả kiểm ngày 26/10/2026. Phần pháp lý Việt Nam dựa trên tóm tắt của báo chính thống và công ty tư vấn (EY, PwC, KPMG); chưa đối chiếu toàn văn nghị định. Một số trang tài liệu (MoMo, ZaloPay, PwC) chặn truy cập tự động, nên nội dung lấy từ đoạn trích kết quả tìm kiếm. Những chỗ như vậy được đánh dấu **(chưa kiểm chứng)**.

## Tóm tắt quyết định: thêm, gộp, bỏ

**Module mới** (chi tiết ở mục 3, lộ trình ở mục 4):

| Module | Loại | Ưu tiên | Quý |
|---|---|---|---|
| `platform-starter-scheduling` | starter Java | P0 | Q4/2026 |
| `platform-starter-ratelimit` | starter Java | P0 | Q4/2026 |
| `platform-starter-privacy` + `@vyforge/privacy` | starter + package web | P0 | Q4/2026 – Q1/2027 |
| `platform-starter-notification` + `@vyforge/notifications` | starter + package web | P0 | Q4/2026 – Q1/2027 |
| `platform-starter-files` + `@vyforge/files` | starter + package web | P0 | Q4/2026 – Q1/2027 |
| `template-infra` | template Copier (repo mới) | P0 | Q4/2026 – Q1/2027 |
| `platform-observability` | chart trong `platform-helm-charts` | P0 | Q1/2027 |
| `platform-starter-export` | starter Java | P1 | Q1/2027 |
| `platform-starter-payment` | starter Java | P1 | Q1–Q2/2027 |
| `platform-starter-webhooks` | starter Java | P1 | Q1/2027 |
| `platform-starter-features` + `@vyforge/features` | starter + package web | P1 | Q1/2027 |
| `platform-starter-realtime` | starter Java (+ hook trong `@vyforge/query`) | P1 | Q2/2027 |
| `.github`: `openapi-sdk.yml`, `load-test.yml` | reusable workflow | P1 | Q2/2027 |
| `platform-starter-ai` | starter Java | P2 | khi có dự án |
| `template-expo` | template Copier (repo mới) | P2 | khi có dự án |
| `platform-starter-einvoice`, adapter eKYC | starter Java | P2 | khi có dự án |

**Bổ sung vào module có sẵn (không tạo module mới):**

| Module có sẵn | Thêm gì |
|---|---|
| `platform-starter-jpa` | `LocalizedText`, `@SoftDeletable` + purge, `immutable_unaccent()` + index trigram mẫu, converter mã hoá cột (đưa `AesGcmCipher` từ auth-server lên) |
| `platform-test` | `TestDataFactory` (Datafaker locale `vi`), luật ArchUnit cho `@PersonalData` và cấm `LocalDateTime` trong entity |
| `@vyforge/ui` | `DataTable`, `ResourceForm`, `PermissionGate` |
| `template-spring-boot` | compose thêm SeaweedFS và `grafana/otel-lgtm`; câu hỏi Copier cho các starter mới; sub-template `module`; mẫu k6 |
| `template-nextjs` | câu hỏi `include_admin` |
| `.github` `java-ci.yml` | bước OpenAPI diff |

**Gộp hoặc thay thế:**

| Hiện có | Quyết định |
|---|---|
| `RateLimitFilter` tự viết trong `platform-gateway` và `platform-auth-server` | Bỏ, thay bằng `platform-starter-ratelimit` (Bucket4j + Redis) |
| Mailer riêng của `platform-auth-server` (`ThymeleafAccountMailer`) | Giữ lúc đầu, chuyển sang `platform-starter-notification` khi starter ổn định (tuỳ chọn) |
| Object storage (chưa repo nào dùng MinIO) | Không đưa MinIO vào; local dùng SeaweedFS, production dùng S3-compatible của khách |

**Không làm** (có lý do ở mục 3): Spring Cloud Config, Quartz, Camunda, Novu tự host, Backstage (trước khi có khoảng 5 team), framework admin chung, CMS nhúng trong backend Spring. Hai repo rỗng `nextjs-ui` và `spring-boot-starter` không có vai trò gì trong bản đồ module này. Đề xuất **archive** cả hai sau khi người phụ trách xác nhận.

## 1. Các nền tảng làm chuẩn so sánh

| Nền tảng | Phiên bản hiện tại | Điều đáng học |
|---|---|---|
| **ABP Framework** (.NET) | 10.7.0 ([GitHub releases](https://github.com/abpframework/abp/releases)) | Danh mục 28 module dạng "cài là dùng" ([abp.io/modules](https://abp.io/modules)): Account, Identity, Tenant Management/SaaS, Feature Management, Setting Management, Audit Logging, File Management, Text Template Management, Language Management, GDPR, Payment, Chat, CMS Kit, Forms, Elsa Workflows, Operation Rate Limiting, AI Management, Twilio SMS. Có một nửa là bản Pro (trả phí) |
| **JHipster** | 9.x; 9.0 (03/2026) bỏ Java 17, chạy Boot 4 ([release 9.0.0](https://jhipster.tech/2026/03/10/jhipster-release-9.0.0.html)); 9.3 chuyển hẳn sang Boot 4.1 ([changelog 9.3.0](https://newreleases.io/project/github/jhipster/generator-jhipster/release/v9.3.0)) | Câu hỏi generator chính là danh sách "module thông dụng" của thế giới Spring ([creating-an-app](https://www.jhipster.tech/creating-an-app/)): cache (Caffeine/Redis/Hazelcast…), Elasticsearch, WebSocket, Kafka/Pulsar, gateway, OAuth2/Keycloak, API-first, i18n (chỉ UI), Gatling/Cucumber |
| **Spring ecosystem** | Boot 4.1.2, Modulith 2.1.2, Spring AI 2.0.2 GA (Maven Central). Boot 4.2/Modulith 2.2/Spring Cloud 2026.0 đang ở RC1 | Modulith 2.1 thêm outbox externalization qua **Namastack** và **JobRunr** ([blog 11/06/2026](https://spring.io/blog/2026/06/11/spring-modulith-2-1-ga-2-0-7-and-1-4-12-released/)). Spring AI 2.0 GA ngày 12/06/2026, dùng Boot 4/Jackson 3, MCP nằm trong core ([blog](https://spring.io/blog/2026/06/12/spring-ai-2-0-0-GA-available-now/)) |
| **Backstage** | v1.56.2 (22/10/2026, [releases](https://github.com/backstage/backstage/releases)) | Golden path = software template + catalog + TechDocs. Vyforge đã có golden path bằng Copier, nên Backstage chỉ đáng làm khi có nhiều team cần self-service |
| **CNCF Platforms white paper** | [TAG App Delivery](https://tag-app-delivery.cncf.io/whitepapers/platforms/) | 13 nhóm capability: web portal, API, golden path, build/test, deliver/verify (Argo, Flux, **OpenFeature**), dev environment, observability, infrastructure, data, messaging, identity & secret (Keycloak, **External Secrets**, cert-manager), security (OPA, Kyverno), artifact storage |
| **Supabase** | [supabase.com/features](https://supabase.com/features) | Ngoài DB/Auth: Storage (S3-compatible, image transform, resumable upload), Realtime (broadcast, presence, Postgres changes), Edge Functions, **Queues**, **Cron**, Vector, **Database Webhooks**, Vault, Log drains |
| **Appwrite** | 2.6.2 (24/10/2026, [GitHub](https://github.com/appwrite/appwrite)), BSD-3 | Có module **Messaging** riêng gồm email (SMTP, SendGrid, Resend, Mailgun), SMS (Twilio, Vonage, MSG91…) và push (APNs, FCM), gửi theo topic ([docs](https://appwrite.io/docs/products/messaging)) |
| **SaaS starter** (MakerKit, SupaStarter…) | — | Bộ "tối thiểu" thường gồm team/organization, billing theo seat, **in-app + realtime notification**, **feature flag theo biến môi trường**, super-admin có impersonation kèm audit ([MakerKit flags](https://makerkit.dev/docs/next-supabase-turbo/configuration/feature-flags-configuration)). Đây là tài liệu của nhà bán, chỉ nên dùng để tham khảo |

## 2. Bảng so sánh module × nền tảng

Ký hiệu: ● có sẵn, ◐ một phần hoặc qua plugin, ○ không có. Cột vyforge: **✓** đã có, **◐** một phần, **✗** chưa có. Bằng chứng cho cột vyforge lấy từ kiểm kê hệ sinh thái ngày 26/10/2026 và grep mã nguồn các repo `platform-*`/`template-*`.

| Module | ABP | JHipster | Spring | Supabase | Appwrite | SaaS starter | **vyforge** | Ghi chú vyforge |
|---|---|---|---|---|---|---|---|---|
| Đăng nhập OIDC, MFA, social | ● | ● | ● (SAS) | ● | ● | ● | **✓** | `platform-auth-server` |
| Phân quyền chi tiết, multi-tenant | ● | ◐ | ◐ | ● (RLS) | ● | ◐ | **✓** | `platform-starter-authz`, `platform-iam-server`; tenant trong token do tab khác làm |
| Audit log nghiệp vụ | ● | ◐ | ○ | ◐ | ○ | ◐ | **✓** | `platform-starter-audit` |
| Gateway, rate limit | ◐ | ● | ● | ● | ● | ○ | **◐** | Gateway có rate limit nhưng bucket nằm trong bộ nhớ từng instance |
| Messaging / outbox | ◐ | ● | ● | ● (Queues) | ○ | ○ | **✓** | `platform-starter-messaging` + Modulith outbox |
| Observability (thư viện) | ◐ | ● | ● | ● | ◐ | ○ | **◐** | Có starter OTel. **Chưa có stack** (Loki/Tempo/Mimir/Grafana), alerting, SLO |
| Secrets | ◐ | ○ | ● (Vault) | ● | ○ | ○ | **◐** | Chart có `ExternalSecret`. Chưa chọn backend secret |
| Thông báo email/SMS/push/in-app | ● (template, SMS) | ◐ (mail) | ◐ | ◐ (email auth) | ● | ● | **✗** | Chỉ auth-server tự gửi mail (`ThymeleafAccountMailer`) |
| Lưu trữ file, presigned URL | ● (Pro) | ○ | ○ | ● | ● | ● | **✗** | |
| Job/scheduler phân tán | ● (BackgroundJobs) | ○ | ◐ (`@Scheduled`) | ● (Cron) | ● (Functions schedule) | ○ | **✗** | Không có `@Scheduled`, ShedLock hay JobRunr trong platform |
| Feature flags | ● | ○ | ○ | ○ | ○ | ● | **✗** | |
| Tìm kiếm | ○ | ● (Elasticsearch) | ○ | ● (FTS, vector) | ◐ | ○ | **✗** | |
| Webhook ra ngoài | ◐ | ○ | ○ | ● | ● | ◐ | **✗** | |
| Workflow/BPM | ● (Elsa, Pro) | ○ | ○ | ○ | ○ | ○ | **✗** | |
| Báo cáo, xuất Excel/PDF | ○ | ○ | ○ | ○ | ○ | ○ | **✗** | Không nền tảng nào có sẵn. Dự án VN gần như luôn cần |
| Thanh toán | ● (Payment, Pro; Stripe/PayPal) | ○ | ○ | ○ | ○ | ● (Stripe) | **✗** | Không ai có VNPay/MoMo/ZaloPay/VietQR |
| Đa ngôn ngữ dữ liệu, tiền tệ, múi giờ | ● (Language Mgmt) | ◐ (UI) | ○ | ○ | ○ | ○ | **◐** | `@vyforge/i18n` vi/en cho UI, có `Asia/Ho_Chi_Minh`. Chưa có dữ liệu đa ngôn ngữ và kiểu tiền VND |
| PII, consent, quyền chủ thể dữ liệu | ● (GDPR) | ○ | ○ | ○ | ○ | ○ | **✗** | Bắt buộc theo luật VN từ 01/01/2026 |
| Soft delete, retention, export dữ liệu | ● | ○ | ○ | ◐ | ○ | ○ | **✗** | |
| CDC | ○ | ○ | ○ | ● (Postgres changes) | ○ | ○ | **✗** | |
| API docs, SDK, contract test | ◐ | ● | ● (springdoc, Contract) | ● | ● | ○ | **◐** | Có springdoc, `@vyforge/api-client`, Pact trong BOM. Chưa có portal tài liệu |
| Admin/backoffice, CMS | ● (CMS Kit) | ● (admin UI) | ○ | ● (Studio) | ● (Console) | ● | **◐** | Có `platform-auth-admin` và `@vyforge/iam`. Chưa có khung CRUD admin chung |
| Realtime WebSocket/SSE | ● (Chat, SignalR) | ● | ● | ● | ● | ● | **✗** | |
| AI/LLM | ● (AI Mgmt, Pro) | ○ | ● (Spring AI) | ● (vector) | ○ | ○ | **✗** | |
| Mobile | ◐ (MAUI/React Native) | ◐ (Ionic blueprint) | ○ | ● (SDK) | ● (SDK) | ● (Expo) | **✗** | |
| Dev portal, gitops, seed, load test | ○ | ◐ (Gatling) | ○ | ● (CLI, local dev) | ● | ○ | **◐** | Có Copier và `gitops-bump`. Chưa có `template-infra`, seed, k6 |

**Nhận xét.** Theo bảng, vyforge mạnh hơn mặt bằng chung ở identity/IAM/audit, ngang bằng ở khung code, nhưng thiếu gần hết các module "ứng dụng". Có ba nhóm không nền tảng ngoại nào có sẵn mà dự án VN chắc chắn cần: **xuất Excel/PDF**, **thanh toán nội địa** và **tuân thủ luật dữ liệu cá nhân VN**. Đây là chỗ vyforge tạo được giá trị riêng.

## 3. Phân tích từng module còn thiếu

Ước lượng công sức tính bằng **tuần-người** (1 dev backend hoặc frontend có kinh nghiệm với platform). Con số đã gồm test, tài liệu, câu hỏi tuỳ chọn trong template và một phiên bản phát hành. Mọi starter mới tuân theo quy ước có sẵn: `@ConfigurationProperties` dạng record, auto-configuration tự lùi khi dự án tự khai báo bean, lỗi trả Problem Details có `code`, i18n vi/en, Testcontainers.

### 3.1 Thông báo: email, SMS, push, Zalo ZNS, in-app, template

**Vì sao cần.** Hầu như mọi dự án đều gửi email giao dịch, OTP, nhắc việc. Ở VN, Zalo ZNS đang thay dần SMS brandname cho tin chăm sóc khách hàng. Hiện mỗi dự án tự viết, mỗi nơi một kiểu retry và lưu template.

**Hiện trạng thị trường.**
- **Zalo ZNS** chỉ gửi theo template đã được Zalo duyệt trước, tối đa 400 ký tự. Từ 10/12/2025 có thêm kênh **ZBS** gửi theo Zalo UID, giá thấp hơn; template duyệt trước ngày đó chỉ gửi được theo số điện thoại ([Infobip](https://www.infobip.com/docs/zalo/message-templates)). Giá tham khảo qua reseller là 200 VND/tin chăm sóc và 300 VND/tin khuyến mại ([8x8](https://cpaas.8x8.com/en/pricing/messaging-apps-pricing/)) **(chưa kiểm chứng với bảng giá chính thức của Zalo)**.
- **Novu** (`@novu/framework` v2.14.0, 28/09/2026) có inbox in-app, workflow đa kênh. License là MIT cho phần lõi, nhưng thư mục `enterprise/` theo license thương mại ([LICENSE](https://github.com/novuhq/novu)). Muốn dùng phải chạy thêm một cụm service (API, worker, MongoDB, Redis), nặng so với nhu cầu của dự án tier 0–1.
- Appwrite Messaging cho thấy một mô hình gọn: một API, ba kênh, adapter theo provider.

**Phương án đề xuất: tự build lớp mỏng, không chạy Novu.**
- `platform-starter-notification`:
  - API `Notifications.send(NotificationRequest)` (người nhận, template key, locale, kênh ưu tiên, biến).
  - Gửi qua **outbox của Modulith** (event `NotificationRequested` → listener theo kênh), nên retry, DLT và idempotent dùng lại cơ chế đã có trong `platform-starter-messaging`.
  - SPI `NotificationChannel` với các adapter:
    - email: Spring Mail, Angus Mail 2.0.5, SMTP hoặc API provider.
    - SMS: HTTP chung, kèm adapter cho nhà mạng hoặc aggregator (eSMS, SpeedSMS…) khi có dự án cần.
    - Zalo ZNS: gọi OpenAPI của Zalo, lưu `template_id` và mapping tham số.
    - push: Firebase Admin 9.11.0 cho FCM, Pushy 0.15.6 cho APNs.
    - in-app: bảng `notification_inbox` cùng API `/api/v1/notifications/me`.
  - Template email viết bằng Thymeleaf 3.1.5 (đã dùng trong auth-server), có bản vi/en. Nếu cần email responsive thì biên dịch MJML 5.4.1 (MIT) lúc build. Không nên render MJML lúc chạy vì phải kéo Node vào runtime.
  - Bảng `notification_log` ghi kết quả từng lần gửi, phục vụ đối soát chi phí ZNS/SMS.
- `@vyforge/notifications` (platform-web): component inbox, badge chưa đọc. Cập nhật realtime qua SSE (mục 3.19).

**Công sức:**
- email + in-app + outbox + template: 3 tuần-người
- adapter ZNS: 1 tuần-người
- SMS/push: 1 tuần-người mỗi loại, chỉ làm khi có dự án cần

**Rủi ro:**
- ZNS cần OA đã xác thực và template được duyệt. Không tự động hoá được phần thủ tục này.
- Gửi email bị chấm spam nếu SPF/DKIM/DMARC của khách hàng chưa đúng. Cần checklist vận hành.
- Số điện thoại và email là dữ liệu cá nhân: log gửi phải che (mục 3.14).

### 3.2 Lưu trữ file: S3 presigned, quét virus, xử lý ảnh

**Vì sao cần.** Upload avatar, chứng từ, hợp đồng, ảnh sản phẩm. Làm sai thì hoặc file đi qua app server (tốn RAM, timeout), hoặc bucket bị public.

**Hiện trạng thị trường.**
- **MinIO bản cộng đồng đã ngừng.** Repo `minio/minio` đã bị archive, README hướng người dùng sang AIStor trả phí. Bản phát hành cuối là `RELEASE.2025-10-15T17-29-55Z` (GitHub API; [pinggy](https://pinggy.io/blog/minio_archived_self_hosted_s3_alternatives/)). Không nên chọn MinIO cho dự án mới.
- Ba lựa chọn S3-compatible tự host đang được dùng thay MinIO ([so sánh của Elest.io](https://blog.elest.io/rustfs-vs-seaweedfs-vs-garage-which-minio-alternative-should-you-pick/)):
  - **SeaweedFS** 4.51 (Apache-2.0): lâu năm nhất.
  - **RustFS** 1.0.4 (Apache-2.0): gần MinIO nhất nhưng mới.
  - **Garage**: nhỏ gọn, chỉ có một tập con API S3.
- Quét virus: **ClamAV** 1.5.4 (GPL-2.0), chạy như một daemon riêng. Client Java cho `clamd` trên Maven Central đã bỏ lâu (`xyz.capybara:clamav-client` 2.1.2 từ 2022), nên tự viết client giao thức INSTREAM (khoảng 100 dòng) an toàn hơn.
- Ảnh: **imgproxy** v4.0.19 (Apache-2.0) resize theo URL có ký; hoặc Thumbnailator 0.4.21 + TwelveMonkeys 3.15.3 (WebP) nếu xử lý trong JVM.

**Phương án đề xuất.**
- `platform-starter-files`:
  - Dùng AWS SDK v2 (`software.amazon.awssdk:s3` 2.57.3) với `endpoint-override` và `path-style`, nên một code chạy được với AWS S3, Cloudflare R2, SeaweedFS hoặc RustFS.
  - Luồng chuẩn:
    1. `POST /files/uploads` trả presigned PUT có giới hạn `Content-Type`/`Content-Length`, kiểm quyền bằng IAM action `files:object:Upload`.
    2. Client upload thẳng lên S3.
    3. `POST /files/uploads/{id}/complete` ghi metadata và phát event `FileUploaded`.
    4. Listener quét ClamAV, đánh dấu `CLEAN`/`INFECTED`.
    5. Chỉ file `CLEAN` mới được cấp presigned GET.
  - Bảng `file_object` (tenant, owner, bucket, key, sha256, size, mime thật xác định bằng Apache Tika 4.1.0, trạng thái quét, retention).
- Local và CI: `compose.yaml` của template thêm **SeaweedFS** (chế độ `server -s3`). Testcontainers dùng cùng image.
- `@vyforge/files`: hook và component upload trực tiếp lên presigned URL, kèm progress.
- Chart: không thêm. Object storage là hạ tầng của khách hàng hoặc cloud.

**Công sức:**
- starter + quét virus + package web: 3 tuần-người
- xử lý ảnh qua imgproxy: thêm 0,5 tuần

**Rủi ro:**
- Khác biệt nhỏ giữa các bản S3-compatible (checksum mới của SDK v2, multipart). Phải chạy test tích hợp trên đúng backend sẽ dùng ở production.
- ClamAV tốn khoảng 1–1,5 GB RAM cho cơ sở dữ liệu chữ ký **(chưa kiểm chứng)**.

### 3.3 Job và scheduler phân tán: ShedLock, JobRunr, Quartz

**Vì sao cần.** Với `@Scheduled` thuần, chạy 2 replica thì job chạy 2 lần. Ngoài ra dự án cần job nền có retry, như gửi báo cáo hay đồng bộ với hệ thống ngoài.

| Lựa chọn | Phiên bản | License | Nhận xét |
|---|---|---|---|
| **ShedLock** | 7.10.1, hỗ trợ Spring 7/Boot 4, JDK ≥ 17 ([README](https://github.com/lukas-krecan/ShedLock)) | Apache-2.0 | Chỉ khoá `@Scheduled` để chạy đúng một nơi. `JdbcTemplateLockProvider.usingDbTime()` lấy giờ từ DB để tránh lệch đồng hồ. Rất nhẹ |
| **JobRunr** | 9.0.2 (20/10/2026, Maven Central). Có `jobrunr-spring-boot-4-starter` | **LGPL-3.0** cho bản OSS, có bản Pro thương mại | Job nền và job định kỳ lưu trong PostgreSQL, có dashboard, retry. Modulith 2.1 dùng JobRunr làm một lựa chọn outbox |
| **Quartz** | 2.5.2 (12/2025) | Apache-2.0 | Cluster qua JDBC; API cũ, nhiều bảng |
| Supabase Cron / pg_cron | — | — | Gắn với DB, khó test và khó version |

**Phương án đề xuất.**
- `platform-starter-scheduling`:
  - ShedLock JDBC với `usingDbTime()`, tự cấu hình khi có `DataSource`.
  - Bảng `shedlock` phát hành dưới dạng file SQL mẫu, theo cách `iam.sql` đang làm.
  - Annotation tiện dụng `@ClusterScheduled`, mặc định `lockAtMostFor` lấy từ cấu hình.
  - Mỗi lần chạy job ghi metric/trace qua Micrometer.
- Job nền có retry: dùng **sự kiện Modulith + listener** cho phần lớn trường hợp, vì đã có outbox. Chỉ dùng **JobRunr** khi cần dashboard hoặc lập lịch động do người dùng tạo. Khi đó phải có ADR, vì LGPL-3.0 cần pháp chế xác nhận (dùng như thư viện, không sửa mã thì thường chấp nhận được, nhưng là quyết định của công ty).
- Không dùng Quartz cho dự án mới.

**Công sức:** 1 tuần-người. **Rủi ro:** thấp. Cần hướng dẫn cách chọn `lockAtMostFor` để job dài không bị chạy chồng.

### 3.4 Feature flags: OpenFeature, flagd, Unleash

**Vì sao cần.** Bật tính năng theo tenant hoặc theo khách hàng, phát hành dần (dark launch), tắt nhanh khi sự cố. ABP gọi đây là Feature Management và gắn với gói dịch vụ (edition) của tenant.

**Hiện trạng.**
- **OpenFeature** là chuẩn API trung lập, có trong CNCF white paper:
  - Java SDK `dev.openfeature:sdk` 1.23.0.
  - Provider `flagd` 0.15.0.
  - JS `@openfeature/server-sdk` 1.23.0, `@openfeature/web-sdk` 1.10.0, `@openfeature/react-sdk` 1.4.1.
  - Tất cả Apache-2.0.
- **flagd** core v0.19.0 (Apache-2.0): flag khai báo bằng JSON/YAML, đọc từ file, ConfigMap hoặc HTTP; chạy sidecar hoặc in-process.
- **Unleash** server v8.2.1 (**AGPL-3.0**), client Java 12.3.1 (Apache-2.0): có UI quản trị, nhưng thêm một service phải vận hành.

**Phương án đề xuất.**
- `platform-starter-features`:
  - Đăng ký `OpenFeatureAPI` làm bean.
  - `EvaluationContext` tự điền `tenantId`, `sub`, roles từ JWT.
  - Provider mặc định là **in-process** đọc flag từ cấu hình ứng dụng hoặc file flagd, không cần service riêng.
  - Đổi sang flagd hoặc Unleash chỉ là đổi provider.
- Flag theo **gói dịch vụ của tenant** (kiểu ABP edition): giải trên cùng khung IAM. Policy IAM đã có `Condition` theo `principal.*`, nên thêm thuộc tính `principal.plan` qua `PrincipalAttributesProvider` là đủ. Không làm hệ thống "edition" riêng.
- `@vyforge/features`: helper server-side `isEnabled()` cho Server Component. Hydrate xuống client qua props, không gọi provider từ trình duyệt.

**Công sức:** 1,5 tuần-người. **Rủi ro:** flag rác tích tụ. Cần quy ước đặt hạn sử dụng cho từng flag và lint cảnh báo flag quá hạn.

### 3.5 Cấu hình và secrets: Vault, External Secrets, Spring Cloud Config

**Hiện trạng.**
- **External Secrets Operator** (v2.14.0, 23/10/2026, Apache-2.0) đã là cách chart `app` của vyforge nhận secret (`ExternalSecret`), khớp quy tắc B18.
- **HashiCorp Vault** v2.2.1 đã chuyển sang **BUSL-1.1**. Dùng nội bộ được, nhưng cấm cung cấp dạng hosted để cạnh tranh với IBM ([LICENSE](https://github.com/hashicorp/vault)).
- Fork mã nguồn mở của Vault là **OpenBao** v2.7.1 (MPL-2.0, 01/10/2026).
- Spring Cloud Config 5.0.6 vẫn được duy trì, nhưng khi đã có Kubernetes, ESO và cấu hình qua biến môi trường thì nó trùng vai.

**Phương án đề xuất.**
- Không thêm starter. Ghi chuẩn: secret đi qua ESO, backend tuỳ khách hàng (AWS/GCP Secret Manager, Azure Key Vault, OpenBao).
- Khi khách hàng tự host trên hạ tầng riêng thì khuyến nghị **OpenBao** thay Vault vì license.
- Spring Cloud Config: không dùng.
- Cấu hình thay đổi lúc chạy (ngưỡng, bật/tắt) đi qua feature flag ở mục 3.4, không qua config server.
- Bổ sung vào `template-infra`: `ClusterSecretStore` mẫu cho OpenBao và cho cloud.

**Công sức:** 0,5 tuần (tài liệu + mẫu). **Rủi ro:** thấp.

### 3.6 Tìm kiếm: PostgreSQL FTS, Meilisearch, OpenSearch

**Hiện trạng.**
- **PostgreSQL** không có cấu hình FTS cho tiếng Việt. Cách thường dùng ([Cybrosys](https://www.cybrosys.com/research-and-development/postgres/how-to-use-the-unaccent-extension-in-postgresql), [Ora2Pg](https://ora2pg.darold.net/docs/configuration/full-text-search)):
  - `unaccent` bọc trong hàm `IMMUTABLE` để dựng được index;
  - kèm `pg_trgm` (GIN `gin_trgm_ops`) để tìm không dấu và tìm gần đúng.
- **Meilisearch** v1.55.0. License là **MIT cho lõi + BUSL-1.1 cho phần Enterprise** ([LICENSE](https://github.com/meilisearch/meilisearch)). Tiếng Việt dùng pipeline Latin mặc định, tách từ theo khoảng trắng ([docs](https://www.meilisearch.com/docs/learn/what_is_meilisearch/language)). Chưa xác nhận được cách nó xử lý dấu thanh **(chưa kiểm chứng)**.
- **OpenSearch** 3.9.0 (Apache-2.0), client Java 3.10.0: mạnh nhất nhưng vận hành nặng nhất.
- ParadeDB (BM25 trong Postgres) và Typesense đều theo AGPL/GPL.

**Phương án đề xuất.**
- Mặc định là **PostgreSQL**. Cung cấp trong `platform-starter-jpa`:
  - migration mẫu tạo `immutable_unaccent()` và index trigram;
  - helper `Specification` cho tìm không dấu.
- Đủ cho danh sách admin và tìm theo tên/mã, tới vài triệu bản ghi **(ngưỡng ước lượng, cần benchmark)**.
- Khi cần ranking, facet hoặc gợi ý: viết ADR chọn Meilisearch (dự án vừa, lưu ý license EE) hoặc OpenSearch (dữ liệu lớn, log). Đồng bộ dữ liệu qua event Modulith, hoặc qua CDC (mục 3.15) khi có nhiều nguồn.

**Công sức:** 1 tuần (PG) + 2 tuần (adapter Meilisearch, khi cần). **Rủi ro:** tìm tiếng Việt có dấu/không dấu. Phải có bộ test dữ liệu tiếng Việt thật.

### 3.7 Cache và rate limit phân tán: Bucket4j + Redis

**Hiện trạng.**
- Gateway và auth-server đều tự viết token bucket trong bộ nhớ, nên với N replica giới hạn thực tế gấp N lần.
- Spring Cloud Gateway **Server WebMVC** có sẵn filter `Bucket4jFilterFunctions.rateLimit`. Filter cần bean `AsyncProxyManager`. Tài liệu chỉ có ví dụ Caffeine (bộ nhớ local) và hướng sang tính năng phân tán của Bucket4j ([docs](https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webmvc/filters/ratelimiter.html)).
- Bucket4j 8.21.0 (`com.bucket4j:bucket4j_jdk17-lettuce`, Apache-2.0) có proxy manager trên Redis/Lettuce.

**Phương án đề xuất.**
- `platform-starter-ratelimit`, hoặc gộp vào `platform-starter-web` dưới dạng tuỳ chọn:
  - Tạo `AsyncProxyManager` trên Redis khi có `RedisConnectionFactory`, không có Redis thì dùng Caffeine.
  - Annotation `@RateLimited(key = "sub|ip|tenant", capacity, period)` cho endpoint nhạy cảm (OTP, quên mật khẩu, export).
- Gateway thay `RateLimitFilter` tự viết bằng filter Bucket4j của Spring Cloud Gateway, dùng chung proxy manager.
- Auth-server dùng cùng starter.
- Cache dữ liệu: đã có Spring Cache + Redis của Boot. Chỉ cần thêm quy ước key có `tenantId` và TTL mặc định.

**Công sức:** 1,5 tuần. **Rủi ro:** Redis thành điểm phụ thuộc. Cần chính sách khi Redis lỗi: mặc định cho qua (fail-open) với API thường, từ chối (fail-closed) với OTP/đăng nhập.

### 3.8 Webhook ra ngoài: ký HMAC, retry

**Hiện trạng.**
- **Standard Webhooks** là spec do Svix, Zapier, Twilio, Supabase, Kong… cùng theo ([spec](https://github.com/standard-webhooks/standard-webhooks/blob/main/spec/standard-webhooks.md)):
  - header `webhook-id`, `webhook-timestamp`, `webhook-signature`;
  - ký HMAC-SHA256 (secret 24–64 byte) hoặc ed25519;
  - xoay khoá bằng cách gửi nhiều chữ ký trong một header;
  - bên nhận dùng `webhook-id` làm khoá idempotency và kiểm độ lệch timestamp;
  - bên gửi retry theo lịch backoff kéo dài nhiều ngày, có jitter.
- Thư viện Java `com.standardwebhooks:standardwebhooks` 1.2.0 (Apache-2.0).
- **Svix** server v2.9.0 (MIT) là bản tự host đầy đủ, nhưng là thêm một service.

**Phương án đề xuất.**
- `platform-starter-webhooks`:
  - bảng `webhook_endpoint` (tenant, url, secret mã hoá, event đăng ký) và `webhook_delivery`;
  - phát từ event Modulith qua outbox;
  - worker gửi có backoff + jitter, ký theo Standard Webhooks;
  - **chặn SSRF**: chỉ `https`, phân giải DNS rồi chặn IP nội bộ/link-local, không theo redirect;
  - API quản trị endpoint và gửi lại, bảo vệ bằng IAM action `webhooks:*`.
- Phía nhận (webhook từ VNPay/MoMo… vào hệ thống): dùng chung tiện ích kiểm chữ ký và idempotency (bảng `processed_messages` có sẵn).

**Công sức:** 2,5 tuần. **Rủi ro:** SSRF. Bắt buộc có test cho các trường hợp IP nội bộ.

### 3.9 Workflow/BPM: Temporal, Camunda, Flowable

| Lựa chọn | Phiên bản | License | Nhận xét |
|---|---|---|---|
| **Temporal** | server v1.32.1 (MIT); SDK Java + `temporal-spring-boot-starter` 1.40.0, hỗ trợ Boot 2/3/4 ([docs](https://docs.temporal.io/develop/java/integrations/spring-boot-integration)) | MIT/Apache | Workflow viết bằng code, bền khi process chết. Cần cụm Temporal (có Helm chart chính thức) |
| **Camunda 8** | 8.10.6 | **Camunda License 1.0**: từ 8.6 production self-managed **phải mua license** ([Camunda](https://camunda.com/blog/2024/04/licensing-update-camunda-8-self-managed/)) | Không phù hợp làm mặc định cho dự án outsourcing |
| **Flowable** | 8.0.0 (02/2026), có Spring Boot starter | Apache-2.0 | BPMN nhúng trong app, dùng PostgreSQL của app. Hợp với quy trình phê duyệt nhiều bước có người tham gia |

**Phương án đề xuất.**
- Không có module mặc định. Saga kỹ thuật ngắn dùng event Modulith.
- **Phê duyệt nghiệp vụ có người tham gia** (đơn từ, mua sắm): ADR chọn **Flowable** nhúng, kèm mẫu tích hợp với IAM (task chỉ hiện cho người có quyền).
- **Quy trình dài, gọi nhiều hệ thống ngoài**: **Temporal**.
- Làm khi có dự án thật.

**Công sức:** 3–4 tuần cho mẫu tích hợp đầu tiên. **Rủi ro:** độ phức tạp vận hành; người dùng nghiệp vụ muốn tự vẽ quy trình (cần modeler).

### 3.10 Báo cáo và xuất file: Excel, PDF

**Vì sao cần.** Mọi màn hình danh sách ở dự án VN đều có nút "Xuất Excel". Hoá đơn, phiếu, hợp đồng cần xuất PDF tiếng Việt đúng font.

**Hiện trạng thư viện** (Maven Central):
- **Apache POI** 6.0.0 (22/10/2026, major mới, Apache-2.0).
- **FastExcel** 0.20.2 (Apache-2.0): chỉ ghi, streaming, rất nhanh, ít RAM.
- **OpenHTMLtoPDF** `io.github.openhtmltopdf` 1.1.100 (LGPL-2.1, fork còn duy trì; group cũ `com.openhtmltopdf` dừng ở 2021).
- PDFBox 3.0.8, OpenPDF 3.0.5 (LGPL/MPL).
- JasperReports 7.1.0 (LGPL, nặng, có designer).
- **Gotenberg** v8.38.1 (MIT): service chuyển HTML/Office sang PDF bằng Chromium/LibreOffice.

**Phương án đề xuất.**
- `platform-starter-export`:
  - `ExcelExporter` trên FastExcel, cho xuất danh sách lớn dạng streaming, có header i18n, định dạng số/ngày/VND.
  - `PdfRenderer`: template Thymeleaf → HTML → OpenHTMLtoPDF, kèm sẵn font Unicode (Be Vietnam Pro hoặc Noto Sans, OFL).
  - **Export bất đồng bộ**: request → job (mục 3.3) → ghi file lên S3 (mục 3.2) → gửi thông báo có link presigned (mục 3.1). Tránh timeout và lộ file.
  - Đọc/nhập Excel dùng POI.
- Nếu cần đúng bố cục Word/Excel của khách: Gotenberg như dịch vụ tuỳ chọn.

**Công sức:** 2,5 tuần. **Rủi ro:** font và ngắt dòng tiếng Việt trong PDF. Export lớn không phân trang làm cạn RAM, nên chỉ dùng API streaming.

### 3.11 Thanh toán VN: VNPay, MoMo, ZaloPay, VietQR

**Hiện trạng giao thức.**

| Cổng | Đặc điểm tích hợp |
|---|---|
| **VNPay** | `vnp_Version` 2.1.0. Chữ ký `vnp_SecureHash` tính trên tham số sắp xếp, HMAC-SHA512 (hoặc SHA256 tuỳ cấu hình). **Return URL chỉ để hiển thị**; cập nhật đơn phải qua **IPN** (server-to-server, cần SSL). Nếu không nhận đúng mã phản hồi, VNPay gọi lại tối đa 10 lần, cách nhau 5 phút. Truy vấn và hoàn tiền qua `merchant_webapi/api/transaction` ([docs](https://sandbox.vnpayment.vn/apis/docs/thanh-toan-pay/pay.html)) |
| **MoMo** | API v3. Chữ ký HMAC-SHA256 trên chuỗi `key=value` sắp xếp a–z, gồm `ipnUrl`, `redirectUrl`, `orderId`… IPN cũng có chữ ký phải kiểm ([signature](https://developers.momo.vn/v3/docs/payment/api/other/signature/)). Khoá production chỉ cấp sau khi MoMo duyệt **(một phần chưa kiểm chứng: trang chặn truy cập tự động)** |
| **ZaloPay** | API v2 `/v2/create`. `mac` = HMAC(key1) trên các trường nối bằng `\|`; callback kiểm bằng key2 ([docs](https://docs.zalopay.vn/en/v2/general/overview.html)) **(chưa kiểm chứng toàn văn)** |
| **VietQR / chuyển khoản** | QR theo chuẩn NAPAS. Việc xác nhận "đã nhận tiền" cần một dịch vụ đọc biến động số dư rồi gọi webhook: API VietQR ([Host To Client](https://api.vietqr.vn/vi/cac-dich-vu-api-khac/host-to-client)), payOS, SePay (hỗ trợ hơn 30 ngân hàng). Phí thấp, phù hợp B2B và đơn giá trị lớn |

Khung pháp lý: thanh toán không dùng tiền mặt theo Nghị định 52/2024/NĐ-CP (hiệu lực 01/07/2024), trung gian thanh toán theo Thông tư 40/2024/TT-NHNN ([thuvienphapluat](https://thuvienphapluat.vn/van-ban/Tien-te-Ngan-hang/Decree-52-2024-ND-CP-regulations-on-cashless-payments-613117.aspx)). Ứng dụng của khách hàng chỉ là merchant, nên không cần giấy phép trung gian.

**Phương án đề xuất.**
- `platform-starter-payment`:
  - Domain chung `PaymentIntent` (tenant, orderRef, amount kiểu VND số nguyên, trạng thái `PENDING/SUCCEEDED/FAILED/REFUNDED`).
  - Port `PaymentGateway`, mỗi cổng một adapter: `vnpay`, `momo`, `zalopay`, `vietqr` (qua payOS hoặc SePay).
  - Endpoint IPN chung `/api/v1/payments/{provider}/ipn`:
    - kiểm chữ ký;
    - **idempotent** theo mã giao dịch của cổng;
    - **đối chiếu số tiền với đơn** trước khi cập nhật;
    - phát event `PaymentSucceeded` qua outbox.
  - Job đối soát định kỳ gọi API truy vấn cho các giao dịch còn `PENDING` quá N phút (mục 3.3).
  - Secret của từng merchant để trong secret store (mục 3.5), không lưu trong DB dạng rõ.
  - Sandbox của từng cổng chạy trong test hợp đồng: dùng WireMock và mẫu payload lấy từ tài liệu.
- Không tích hợp Stripe/PayPal mặc định. Thêm adapter khi có dự án quốc tế.

**Công sức:** khung chung 2 tuần, mỗi cổng 1–1,5 tuần. **Rủi ro:**
- Tài liệu của các cổng hay thay đổi, sandbox thiếu ổn định.
- Xử lý sai IPN (tin Return URL, không kiểm số tiền) là lỗi bảo mật kinh điển. Cần checklist review bắt buộc.

### 3.12 Hoá đơn điện tử và eKYC/CCCD

**Hoá đơn điện tử.**
- Nghị định 70/2025/NĐ-CP sửa Nghị định 123/2020, hiệu lực 01/06/2025 ([MISA](https://www.meinvoice.vn/tin-tuc/31439/nghi-dinh-70-2025-nd-cp-sua-doi-nghi-dinh-123/)). Ví dụ: hoá đơn có thể ghi số định danh cá nhân của người mua thay mã số thuế.
- Có nguồn nhắc tới các nghị định mới hơn năm 2026 nhưng chưa tra được nội dung **(chưa kiểm chứng)**.
- Doanh nghiệp không gửi trực tiếp cho cơ quan thuế mà đi qua **nhà cung cấp dịch vụ hoá đơn** (MISA meInvoice, Viettel S-Invoice, VNPT Invoice…). Mỗi nhà cung cấp có API riêng. MISA có tài liệu tích hợp công khai, gồm môi trường test và production ([doc.meinvoice.vn](https://doc.meinvoice.vn/about)).
- **Đề xuất:**
  - Port `EInvoiceProvider` (phát hành, huỷ, thay thế, tải PDF/XML) nằm trong `platform-starter-payment` hoặc thành starter riêng `platform-starter-einvoice`.
  - Adapter chỉ viết theo nhà cung cấp mà khách hàng đã ký hợp đồng.
  - Chưa làm cho tới khi có dự án. Công sức: 2 tuần cho mỗi nhà cung cấp.

**eKYC/CCCD.**
- CCCD gắn chip lưu dữ liệu theo chuẩn ICAO. Ứng dụng di động đọc NFC, kiểm chữ ký của cơ quan cấp, rồi so khớp khuôn mặt (liveness) ([Didit](https://didit.me/blog/vietnam-cccd-chip-nfc-verification/), nguồn của nhà bán).
- Danh tính điện tử VNeID có giá trị pháp lý theo Nghị định 69/2024/NĐ-CP.
- **Đề xuất:**
  - Không tự build. Dùng nhà cung cấp eKYC có giấy phép (ngân hàng, viễn thông, hoặc dịch vụ chuyên biệt) qua port `IdentityVerification`.
  - Ảnh CCCD và dữ liệu sinh trắc là **dữ liệu nhạy cảm** theo luật (mục 3.14): phải có đánh giá tác động trước khi xử lý.
  - Đưa vào P2.

### 3.13 Đa ngôn ngữ cho dữ liệu, tiền tệ và múi giờ VN

**Hiện trạng.** `@vyforge/i18n` (next-intl) dịch giao diện vi/en và có `Asia/Ho_Chi_Minh`. Backend có bundle lỗi vi/en. Chưa có quy ước cho **dữ liệu** đa ngôn ngữ, chẳng hạn tên sản phẩm hai thứ tiếng (ABP gọi đây là Language Management).

**Đề xuất (quy ước + tiện ích nhỏ, không phải service):**
- **Dữ liệu đa ngôn ngữ:** cột `jsonb` dạng `{"vi": "...", "en": "..."}` với kiểu `LocalizedText` trong `platform-starter-jpa`. Chọn bản dịch theo `Accept-Language`, fallback `vi`. Bảng dịch riêng chỉ dùng khi cần workflow dịch.
- **Tiền:** VND không có phần lẻ, nên lưu `bigint` (đồng). Đa tiền tệ thì dùng `numeric(19,4)` + mã ISO 4217. Không dùng `double`. Thêm record `Money` vào shared kernel mẫu; JSR 354 (Moneta 1.4.5, phát hành cuối 03/2025) không đáng kéo vào. Frontend dùng `Intl.NumberFormat('vi-VN', {style: 'currency', currency: 'VND'})` qua `@vyforge/i18n`.
- **Thời gian:** DB lưu `timestamptz` (UTC), API trả ISO-8601 có offset, chỉ chuyển sang `Asia/Ho_Chi_Minh` khi hiển thị. Job "cuối ngày" (mục 3.3) chạy theo `zone = "Asia/Ho_Chi_Minh"` khai báo rõ.
- Thêm luật ArchUnit cấm `LocalDateTime` trong entity, trừ khi có `@AllowLocalDateTime`.

**Công sức:** 1 tuần. **Rủi ro:** thấp, nhưng sửa về sau rất tốn. Nên đưa vào template sớm.

### 3.14 PII, masking, consent: tuân thủ luật dữ liệu cá nhân VN

**Khung pháp lý hiện hành (đã kiểm tra):**
- **Luật Bảo vệ dữ liệu cá nhân số 91/2025/QH15**, thông qua 26/06/2025, **hiệu lực 01/01/2026** ([TTTC](https://thitruongtaichinhtiente.vn/nhung-diem-chinh-can-luu-y-tai-luat-bao-ve-du-lieu-ca-nhan-chinh-thuc-co-hieu-luc-tu-ngay-1-1-2026-75881.html)).
- **Nghị định 356/2025/NĐ-CP** hướng dẫn luật, ban hành 31/12/2025, cùng hiệu lực và **thay thế Nghị định 13/2023/NĐ-CP** ([TVPL](https://thuvienphapluat.vn/van-ban/Quyen-dan-su/Nghi-dinh-356-2025-ND-CP-huong-dan-Luat-Bao-ve-du-lieu-ca-nhan-687428.aspx)).
- Các điểm ảnh hưởng tới phần mềm, theo bản tóm tắt của EY ([EY](https://www.ey.com/vi_vn/technical/tax/tax-and-law-updates/nghi-dinh-so-356-2025-nd-cp-quy-dinh-chi-tiet-mot-so-dieu-va-bien-phap-thi-hanh-luat-bao-ve-du-lieu-ca-nhan)):
  - **Sự đồng ý** phải rõ ràng, kiểm chứng được, có **lưu vết** thời điểm và nội dung đồng ý. Cấm mặc định đồng ý hoặc dùng câu chữ gây hiểu nhầm.
  - **Yêu cầu của chủ thể dữ liệu:** phản hồi trong **2 ngày làm việc**. Sau đó thực hiện:
    - truy cập hoặc chỉnh sửa trong 10 ngày;
    - rút lại sự đồng ý hoặc hạn chế xử lý trong 20 ngày;
    - xoá trong 20 ngày, hoặc 30 ngày nếu có bên thứ ba.
  - **Vi phạm dữ liệu** liên quan vị trí và sinh trắc: thông báo cơ quan chức năng (A05) và chủ thể trong **72 giờ**. Lưu hồ sơ ít nhất 5 năm.
  - **Hồ sơ đánh giá tác động xử lý** và **chuyển dữ liệu ra nước ngoài**:
    - A05 thẩm định trong 15 ngày;
    - cập nhật định kỳ 6 tháng, hoặc trong 10 ngày khi có thay đổi lớn.
  - **Miễn trừ:**
    - hộ kinh doanh và doanh nghiệp siêu nhỏ được miễn bố trí nhân sự bảo vệ dữ liệu và lập hồ sơ đánh giá tác động;
    - doanh nghiệp nhỏ và startup được miễn 5 năm từ 01/01/2026;
    - **không** miễn nếu xử lý dữ liệu nhạy cảm, cung cấp dịch vụ xử lý dữ liệu, hoặc có từ **100.000 chủ thể** trở lên.
  - **Mức phạt:**
    - tối đa 5% doanh thu năm trước nếu vi phạm quy định chuyển dữ liệu xuyên biên giới;
    - tối đa 10 lần khoản thu nếu mua bán dữ liệu;
    - tối đa 3 tỷ đồng với vi phạm khác ([Đại biểu Nhân dân](https://daibieunhandan.vn/luat-bao-ve-du-lieu-ca-nhan-nghiem-cam-hanh-vi-mua-ban-du-lieu-ca-nhan-10382040.html)).

Phần lớn khách hàng outsourcing của vyforge **không được miễn**, vì app thường có trên 100.000 người dùng hoặc xử lý dữ liệu nhạy cảm (sức khoẻ, tài chính, vị trí). Mỗi dự án phải chứng minh được các năng lực trên, nên đây là P0.

**Phương án đề xuất: `platform-starter-privacy` + `@vyforge/privacy` + checklist.**
1. **Phân loại dữ liệu trong code:**
   - Annotation `@PersonalData(category = BASIC|SENSITIVE)` trên field của entity/DTO.
   - Luật ArchUnit yêu cầu mọi entity có cột định danh (email, phone, CCCD…) phải đánh dấu.
   - Sinh tự động **danh mục dữ liệu cá nhân**: bảng, cột, mục đích. Đây là đầu vào cho hồ sơ đánh giá tác động.
2. **Masking:**
   - Serializer Jackson che field `@PersonalData` trong log và trong Problem Details.
   - Layout log JSON có bộ lọc pattern (email, số điện thoại VN, CCCD 12 số).
   - `platform-starter-audit` lưu giá trị đã che.
3. **Consent:**
   - Bảng `privacy_consent` (tenant, sub, mục đích, phiên bản văn bản, kênh, thời điểm, thu hồi). Chỉ ghi thêm, không sửa.
   - API `/api/v1/privacy/me/consents`.
   - Component `@vyforge/privacy` cho màn hình đồng ý và quản lý đồng ý.
4. **Yêu cầu của chủ thể dữ liệu:**
   - Bảng `privacy_request` có SLA theo bảng thời hạn ở trên. Job nhắc khi sắp quá hạn (mục 3.3).
   - SPI `PersonalDataExporter`/`PersonalDataEraser` mà mỗi module nghiệp vụ cài đặt, để "xuất" và "xoá" chạy qua mọi module mà không JOIN chéo (đúng quy tắc B6).
5. **Mã hoá cột nhạy cảm:** converter JPA AES-GCM (auth-server đã có `AesGcmCipher`, có thể đưa lên platform), khoá lấy từ secret store.
6. **Checklist vận hành** trong `.github/docs`:
   - mẫu hồ sơ đánh giá tác động;
   - quy trình thông báo vi phạm trong 72 giờ;
   - danh sách bên xử lý dữ liệu (SaaS email, SMS, cloud) và nơi lưu dữ liệu.

**Công sức:** 4 tuần-người, cộng với thời gian làm việc cùng pháp chế. **Rủi ro:**
- Bản tóm tắt EY chưa phải văn bản gốc; cần pháp chế đối chiếu toàn văn trước khi chốt SLA trong code.
- Có nguồn nhắc tới nghị định xử phạt riêng năm 2026 **(chưa kiểm chứng)**.

### 3.15 Data export/retention, soft delete, CDC (Debezium)

**Đề xuất.**
- **Soft delete:**
  - Không bật mặc định cho mọi bảng, vì xung đột với nghĩa vụ xoá ở mục 3.14.
  - Cung cấp `@SoftDeletable` trong `platform-starter-jpa`: cột `deleted_at` dùng Hibernate `@SoftDelete` hoặc `@SQLRestriction`.
  - Job **purge** xoá cứng sau `retention`.
- **Retention:** khai báo theo loại dữ liệu (`vyforge.retention.<type>=P2Y`). Job dọn chạy qua ShedLock. Audit log và log thông báo có retention riêng.
- **Export dữ liệu** của tenant hoặc người dùng: dùng lại SPI `PersonalDataExporter` và `platform-starter-export` (ZIP JSON/CSV lên S3).
- **CDC:**
  - Debezium 3.7.0.Final (Apache-2.0) chỉ cần khi đồng bộ sang search hoặc kho dữ liệu, hoặc khi tích hợp hệ thống legacy không phát event.
  - Trong monolith đã có outbox Modulith thì **không cần**. Khi cần thì chạy Debezium Server hoặc Kafka Connect như hạ tầng, không nhúng vào app.

**Công sức:** soft delete + retention 1,5 tuần. CDC 2 tuần khi có nhu cầu. **Rủi ro:** soft delete làm unique index và JOIN sai nếu quên điều kiện `deleted_at is null`. Cần test mẫu.

### 3.16 API docs portal, SDK sinh tự động, contract testing

**Hiện trạng.**
- Đã có springdoc (bản mới 3.2.0).
- `@vyforge/api-client` (openapi-fetch) sinh type cho Next.js.
- Pact có trong BOM (provider `junit5spring` 4.7.5, `pact-js` v17.1.4).
- `platform-auth-admin` đã kiểm OpenAPI contract với auth-server trong CI.

**Đề xuất.**
- **Portal:** dùng **Scalar** (`@scalar/api-reference` 1.76.0, MIT) để render OpenAPI của từng service. Gom các spec vào một trang tĩnh build từ CI (GitHub Pages của org). Không cần Backstage cho việc này.
- **SDK cho đối tác/mobile:**
  - TypeScript: `@hey-api/openapi-ts` 0.99.0 hoặc `openapi-typescript` 7.13.0, đã dùng.
  - Kotlin/Swift/Java: OpenAPI Generator v7.26.0 (Apache-2.0), chạy trong reusable workflow mới `openapi-sdk.yml` và phát hành vào GitHub Packages.
- **Contract test:** chuẩn hoá thành job "OpenAPI diff" (phát hiện breaking change) trong `java-ci.yml`. Pact chỉ dùng cho API dùng chéo team (K5).

**Công sức:** 1,5 tuần. **Rủi ro:** thấp.

### 3.17 Observability stack (LGTM), alerting, SLO

**Hiện trạng.**
- App đã phát OpenTelemetry, nhưng chưa có nơi nhận chuẩn.
- Phiên bản hiện tại:
  - Grafana v13.2.4, Loki v3.7.9, Tempo v3.1.1, Mimir 3.2.3: đều **AGPL-3.0**. Dùng nguyên bản không sửa thì không phát sinh nghĩa vụ công bố mã của app.
  - Grafana Alloy v1.21.2 (Apache-2.0), OpenTelemetry Collector v0.163.2.
  - Image all-in-one `grafana/otel-lgtm` v0.38.0, dùng cho local/dev.
- SLO dạng code:
  - **Sloth** v0.16.0 sinh rule Prometheus từ spec SLO;
  - **Pyrra** v0.10.2 có thêm UI;
  - Alertmanager v0.34.1.

**Đề xuất.**
- Local: thêm `grafana/otel-lgtm` vào `compose.yaml` của template (profile `observability`). Dev có trace và log ngay.
- Cluster: trong `template-infra`, đưa chart chính thức của Grafana (Loki, Tempo, Mimir hoặc Prometheus, Grafana) và Alloy làm collector. Values mẫu cho môi trường nhỏ (single-binary) và môi trường production. Nếu khách hàng đã có Grafana Cloud hoặc Datadog thì chỉ đổi endpoint OTLP.
- **Dashboard + alert + SLO chuẩn** cho mọi service Spring trong một chart mới `platform-observability` (ConfigMap dashboard, `PrometheusRule`, spec Sloth):
  - availability 99,5% và latency p95 theo route;
  - lag Kafka;
  - outbox chưa gửi (staleness của Modulith);
  - lỗi đăng nhập ở auth-server.

**Công sức:** 3 tuần. **Rủi ro:** chi phí lưu trữ log. Mặc định retention ngắn (7 ngày log, 3 ngày trace), dự án tự tăng.

### 3.18 Admin/backoffice framework, CMS headless

**Hiện trạng.**
- Có `platform-auth-admin` (Next.js) và `@vyforge/iam`, nên đã có mẫu admin dùng shadcn/ui.
- Thư viện admin:
  - **react-admin** v5.15.5 (MIT);
  - **Refine** `@refinedev/core` 5.0.12 (MIT, bản phát hành chậm lại, gần nhất 04/2026).
- CMS headless:
  - **Payload** 3.92.0 (MIT, chạy trong Next.js);
  - Strapi v5.59.0 (MIT + `ee/`);
  - Directus v12.5.1 (license riêng "Monospace Sustainable Core", không phải OSI).

**Đề xuất.**
- **Không thêm framework admin.** Thêm các khối CRUD admin vào `@vyforge/ui` như `DataTable`: phân trang server khớp `platform-starter-web`, filter qua nuqs (K6), nút xuất Excel gọi mục 3.10. Thêm `ResourceForm` (RHF + Zod), và `PermissionGate` dùng `can()`.
- Thêm vào template-nextjs câu hỏi `include_admin` để sinh khu `/admin` mẫu.
- **CMS:** khi dự án cần trang nội dung do marketing tự sửa, dùng ADR chọn **Payload** (MIT, cùng stack Next.js/TypeScript), chạy như frontend riêng với database PostgreSQL riêng. Không nhúng CMS vào backend Spring.

**Công sức:** 2,5 tuần cho các khối admin. **Rủi ro:** dựng thành một "framework admin" quá chung. Giữ ở mức component, không làm thành DSL.

### 3.19 Realtime: WebSocket/SSE

**Đề xuất.**
- Mặc định là **SSE**: thông báo in-app, tiến độ export, trạng thái thanh toán.
  - Spring MVC có `SseEmitter`, chạy tốt với virtual thread (K4).
  - Đi qua gateway và Next.js route handler dễ hơn WebSocket.
  - Token vẫn nằm ở server BFF: route handler Next.js mở SSE tới backend kèm Bearer, rồi stream lại cho trình duyệt bằng cookie session.
- Fan-out khi có nhiều replica: Redis pub/sub, cùng cách IAM đang xoá cache (`vyforge:iam:changed`). Chuẩn hoá kênh `vyforge:rt:<tenant>:<topic>`.
- Chỉ dùng WebSocket/STOMP (`spring-boot-starter-websocket` 4.1.2) khi cần hai chiều tần suất cao như chat hay cộng tác. Nếu cần số kết nối rất lớn thì dùng **Centrifugo** v6.10.2 (Apache-2.0) như service riêng.
- Module: `platform-starter-realtime` (SSE hub + Redis fan-out + kiểm quyền theo topic bằng IAM), cùng hook `useEventStream()` trong `@vyforge/query`.

**Công sức:** 2 tuần. **Rủi ro:** timeout của proxy/ingress với kết nối dài. Phải gửi heartbeat và cấu hình timeout của gateway.

### 3.20 AI/LLM gateway: Spring AI

**Hiện trạng.**
- Spring AI 2.0.2 GA (2.1 đang RC). Hỗ trợ chính OpenAI, Anthropic, Google GenAI qua SDK của từng hãng. MCP nằm trong core.
- Phía JS: AI SDK `ai` 7.0.153 (Apache-2.0).
- **LiteLLM** v1.106.4 là proxy đa provider (MIT + thư mục `enterprise/`), có quota và log chi phí.

**Đề xuất.**
- `platform-starter-ai`, mỏng:
  - BOM Spring AI;
  - cấu hình mặc định một provider;
  - advisor bắt buộc **che PII** trước khi gửi prompt ra ngoài (dùng lại mục 3.14);
  - ghi token và chi phí vào metric;
  - giới hạn theo tenant (mục 3.7);
  - vector store mặc định là **pgvector** (dùng PostgreSQL sẵn có).
- Gửi dữ liệu cá nhân tới LLM ở nước ngoài là **chuyển dữ liệu xuyên biên giới**: cần hồ sơ đánh giá theo mục 3.14. Starter mặc định **chặn** khi chưa bật cờ xác nhận.
- Chỉ dùng LiteLLM khi nhiều app cần chung quota và hoá đơn.

**Công sức:** 2 tuần. **Rủi ro:** API của Spring AI còn đổi mạnh giữa các major. Ghim version theo BOM, Renovate gom nhóm.

### 3.21 Mobile: React Native/Expo dùng chung @vyforge/*

**Hiện trạng.**
- Expo SDK 58 (`expo` 58.0.19), React Native 0.88.0.
- Better Auth có plugin Expo: `@better-auth/expo` 1.7.13 lưu session trong `expo-secure-store`, quay lại app bằng deep link qua `trustedOrigins` ([docs](https://better-auth.com/docs/integrations/expo)).

**Đề xuất.**
- `template-expo` (Copier):
  - đăng nhập OIDC với `platform-auth-server` (public client + PKCE, đã hỗ trợ);
  - gọi API qua gateway;
  - dùng lại `@vyforge/api-client` và `@vyforge/i18n`;
  - `@vyforge/query` cho dữ liệu;
  - push qua FCM/APNs theo mục 3.1.
- **Package nào dùng chung được:** chỉ những package không phụ thuộc Next/DOM. Cần tách `@vyforge/auth` thành phần core (thuần TS) và phần Next. `@vyforge/ui` (Radix/Tailwind) **không** dùng được trên native.

**Công sức:** 4 tuần, gồm refactor package. **Rủi ro:** khác biệt mô hình bảo mật. Mobile là public client, token nằm trên thiết bị, không có BFF. Cần ADR riêng bổ sung cho ADR-0006.

### 3.22 Dev portal, scaffolding, gitops, test data, load test

- **Gitops (`template-infra`):**
  - Argo CD v3.5.6, Kargo v1.12.4 (Apache-2.0).
  - Đây là hạng mục F1/F2 đã nêu từ báo cáo trước mà vẫn chưa làm, trong khi `gitops-bump.yml` đã có nhưng chưa repo nào gọi. Cần làm trước mọi module chạy như hạ tầng (observability, SeaweedFS, ClamAV).
  - Nội dung: app-of-apps, values theo môi trường, `ClusterSecretStore`, stack observability, chart dịch vụ phụ.
  - Công sức: 3 tuần.
- **Scaffolding:**
  - Tiếp tục dùng Copier (v9.19.1).
  - Thêm workflow định kỳ `copier update` (F3).
  - Thêm **generator module** (`copier copy` sub-template `module` sinh khung Profile A/B trong repo đã có).
  - Công sức: 1 tuần.
- **Dev portal:** Backstage chỉ đáng làm khi có từ khoảng 5 team và vài chục service. Trước mốc đó dùng trang tài liệu tĩnh (ADR, ARCHITECTURE, API portal ở mục 3.16).
- **Test data/seeding:**
  - Datafaker 2.7.0 (Apache-2.0) có locale `vi`; Instancio 6.1.0 cho object test.
  - Đề xuất `platform-test` thêm `TestDataFactory` có locale `vi` và quy ước `db/seed/` (chỉ chạy profile `local`/`demo`, không bao giờ ở production).
  - Công sức: 1 tuần.
- **Load test:**
  - **k6** v2.3.0 (AGPL-3.0, dùng như công cụ thì không ảnh hưởng mã của app).
  - Mẫu script trong template (đăng nhập bằng client credentials của auth-server rồi gọi API) và reusable workflow `load-test.yml` chạy theo yêu cầu với ngưỡng p95/p99.
  - Công sức: 1 tuần.

## 4. Lộ trình ưu tiên 3 đợt

Nguyên tắc xếp hạng:
1. **Bắt buộc theo luật** hoặc **lỗi khi chạy nhiều replica** lên trước.
2. Module mà các module khác phụ thuộc lên trước (job, file, thông báo là nền cho export, thanh toán, quyền riêng tư).
3. Thứ gì phải chạy thêm service thì đi sau `template-infra`.
4. Chỉ làm module cho một ngành khi có dự án thật.

| Đợt | Thời gian (ước lượng) | Module | Công sức | Lý do |
|---|---|---|---|---|
| **P0** | 11/2026 – 01/2027 | `platform-starter-scheduling` (ShedLock) | 1 | Hai replica chạy job hai lần là lỗi dữ liệu thật; nền cho retention, đối soát, export |
| | | `platform-starter-ratelimit` (Bucket4j + Redis) và chuyển gateway/auth-server sang | 1,5 | Sửa lỗ hổng đã biết: rate limit của gateway và auth-server nhân theo số replica |
| | | `platform-starter-privacy` + `@vyforge/privacy` + checklist | 4 | Luật 91/2025 và NĐ 356/2025 đã có hiệu lực; hầu hết khách hàng không được miễn |
| | | `platform-starter-notification` (email + in-app + outbox; ZNS adapter) | 4 | Mọi dự án đều cần; nền cho DSR, export, thanh toán |
| | | `platform-starter-files` + `@vyforge/files` (SeaweedFS local) | 3 | Mọi dự án đều cần; MinIO đã ngừng nên phải chốt chuẩn mới ngay |
| | | Quy ước dữ liệu VN (tiền, thời gian, `LocalizedText`) trong template | 1 | Rẻ nếu làm sớm, rất đắt nếu sửa sau |
| | | `template-infra` + observability stack (LGTM, dashboard, SLO cơ bản) | 6 | Điều kiện để pilot lên production và để vận hành các service phụ |
| **P1** | 02 – 04/2027 | `platform-starter-export` (Excel/PDF bất đồng bộ) | 2,5 | Gần như mọi dự án VN có màn hình danh sách cần xuất file |
| | | `platform-starter-payment` (VNPay, MoMo, VietQR qua payOS/SePay; ZaloPay sau) | 2 + 1–1,5/cổng | Dự án thương mại cần; IPN sai là lỗi bảo mật kinh điển nên cần làm chuẩn một lần |
| | | `platform-starter-webhooks` | 2,5 | Tích hợp đối tác; dùng chung tiện ích ký/idempotency với IPN thanh toán |
| | | `platform-starter-features` (OpenFeature) | 1,5 | Phát hành dần, bật theo tenant/gói dịch vụ |
| | | `platform-starter-realtime` (SSE + Redis) | 2 | Inbox realtime, tiến độ export, trạng thái thanh toán |
| | | Tìm kiếm PostgreSQL (unaccent + trigram) | 1 | Tìm không dấu là yêu cầu mặc định của người dùng VN |
| | | Khối admin trong `@vyforge/ui`, câu hỏi `include_admin` | 2,5 | Rút ngắn phần backoffice của mọi dự án |
| | | Soft delete + retention | 1,5 | Đi cùng nghĩa vụ xoá và lưu trữ |
| | | API portal (Scalar), OpenAPI diff, seed data, k6 | 3,5 | DX và chất lượng; rẻ |
| **P2** | Khi có dự án cần (từ 05/2027) | Workflow (Flowable/Temporal) | 3–4 | Chỉ một số dự án cần; phải có ADR |
| | | `platform-starter-ai` (Spring AI + pgvector + chặn PII) | 2 | Nhu cầu tăng, nhưng API còn biến động; phụ thuộc privacy |
| | | `template-expo` + tách `@vyforge/auth` core | 4 | Cần ADR mô hình bảo mật mobile |
| | | Hoá đơn điện tử (theo nhà cung cấp của khách), eKYC qua nhà cung cấp | 2/nhà cung cấp | Phụ thuộc hợp đồng của khách hàng |
| | | Meilisearch/OpenSearch, CDC Debezium | 2 + 2 | Chỉ khi PostgreSQL không đủ |
| | | Payload CMS, Backstage, Centrifugo, SMS/push adapter | theo nhu cầu | |

Tổng P0 khoảng **20,5 tuần-người**: khoảng 3 tháng cho 2 dev platform, chạy song song với dự án pilot. P1 khoảng **22–24 tuần-người**.

## 5. Đề xuất cập nhật ARCHITECTURE.md (chưa sửa file)

1. **Bảng "Thư viện platform", hàng `platform-java`:** thêm các starter mới khi phát hành: `-scheduling`, `-ratelimit`, `-privacy`, `-notification`, `-files`, `-export`, `-payment`, `-webhooks`, `-features`, `-realtime`, `-ai`. Hàng `platform-web` thêm `privacy`, `files`, `notifications`, `features`. Hàng `platform-helm-charts` thêm chart `platform-observability`.
2. **Bảng "Thư viện platform", thêm hàng `template-infra`** (Argo CD app-of-apps, ESO, stack observability) và nêu rõ quan hệ với `gitops-bump.yml`.
3. **Quy tắc bắt buộc mới (đề xuất số tiếp theo):**
   - **B21:** field chứa dữ liệu cá nhân phải có `@PersonalData`; log và audit không chứa PII dạng rõ; có consent và luồng yêu cầu của chủ thể dữ liệu theo Luật 91/2025/QH15 và NĐ 356/2025/NĐ-CP.
   - **B22:** job định kỳ dùng `@ClusterScheduled` (ShedLock); không dùng `@Scheduled` thuần trong app chạy nhiều replica.
   - **B23:** file người dùng đi qua presigned URL và được quét virus trước khi cho tải; không stream file qua app server.
   - **B24:** tiền lưu dạng số nguyên hoặc `numeric` kèm mã tiền tệ, không dùng `double`; thời điểm lưu `timestamptz` (UTC), chỉ chuyển múi giờ khi hiển thị.
   - **B25:** thanh toán chỉ cập nhật trạng thái qua IPN/webhook đã kiểm chữ ký, idempotent và đối chiếu số tiền; không tin redirect của trình duyệt.
4. **Khuyến nghị mới:**
   - **K13:** rate limit dùng `platform-starter-ratelimit` (Redis) cho endpoint nhạy cảm.
   - **K14:** feature flag qua OpenFeature.
   - **K15:** SSE cho realtime, WebSocket chỉ khi cần hai chiều.
   - **K16:** tìm kiếm mặc định PostgreSQL (unaccent + trigram), search engine riêng phải có ADR.
   - **K17:** secret backend tự host khuyến nghị OpenBao (MPL-2.0) thay Vault (BUSL-1.1).
5. **Sửa K12:** `compose.yaml` thêm SeaweedFS (S3), Mailpit (đã có ở auth-server), `grafana/otel-lgtm` (profile `observability`); ghi rõ **không dùng MinIO** vì bản cộng đồng đã bị archive.
6. **Mục "Quyết định đã duyệt":** chừa chỗ cho các ADR mới; mỗi module P0 cần một ADR ngắn, chủ yếu ghi lựa chọn thư viện và license (LGPL của JobRunr và OpenHTMLtoPDF cần pháp chế xác nhận).

## 6. Bảng phiên bản đã kiểm (26/10/2026)

| Thành phần | Phiên bản | License | Nguồn |
|---|---|---|---|
| Spring Boot / Modulith / Spring AI | 4.1.2 / 2.1.2 / 2.0.2 (4.2, 2.2, 2.1 đang RC1) | Apache-2.0 | Maven Central |
| ShedLock | 7.10.1 | Apache-2.0 | Maven Central |
| JobRunr | 9.0.2 | LGPL-3.0 (OSS) | Maven Central, GitHub |
| Quartz | 2.5.2 | Apache-2.0 | Maven Central |
| Bucket4j (lettuce) | 8.21.0 | Apache-2.0 | Maven Central |
| OpenFeature Java SDK / flagd provider | 1.23.0 / 0.15.0 | Apache-2.0 | Maven Central |
| flagd / Unleash server | 0.19.0 / 8.2.1 | Apache-2.0 / AGPL-3.0 | GitHub |
| AWS SDK v2 S3 | 2.57.3 | Apache-2.0 | Maven Central |
| SeaweedFS / RustFS | 4.51 / 1.0.4 | Apache-2.0 | GitHub |
| MinIO | archive, bản cuối 2025-10-15 | AGPL-3.0 | GitHub |
| ClamAV / Apache Tika | 1.5.4 / 4.1.0 | GPL-2.0 / Apache-2.0 | GitHub, Maven Central |
| imgproxy | 4.0.19 | Apache-2.0 | GitHub |
| Apache POI / FastExcel | 6.0.0 / 0.20.2 | Apache-2.0 | Maven Central |
| OpenHTMLtoPDF (`io.github.openhtmltopdf`) / PDFBox | 1.1.100 / 3.0.8 | LGPL-2.1 / Apache-2.0 | Maven Central |
| Gotenberg | 8.38.1 | MIT | GitHub |
| Standard Webhooks (Java) / Svix | 1.2.0 / 2.9.0 | Apache-2.0 / MIT | Maven Central, GitHub |
| Temporal server / Java SDK | 1.32.1 / 1.40.0 | MIT / Apache-2.0 | GitHub, Maven Central |
| Flowable / Camunda | 8.0.0 / 8.10.6 | Apache-2.0 / Camunda License 1.0 | Maven Central |
| OpenSearch / Meilisearch | 3.9.0 / 1.55.0 | Apache-2.0 / MIT + BUSL-1.1 (EE) | GitHub |
| Debezium | 3.7.0.Final | Apache-2.0 | Maven Central |
| External Secrets / OpenBao / Vault | 2.14.0 / 2.7.1 / 2.2.1 | Apache-2.0 / MPL-2.0 / BUSL-1.1 | GitHub |
| Grafana / Loki / Tempo / Mimir | 13.2.4 / 3.7.9 / 3.1.1 / 3.2.3 | AGPL-3.0 | GitHub |
| Alloy / OTel Collector / otel-lgtm | 1.21.2 / 0.163.2 / 0.38.0 | Apache-2.0 | GitHub |
| Sloth / Pyrra / Alertmanager | 0.16.0 / 0.10.2 / 0.34.1 | Apache-2.0 | GitHub |
| Novu | framework 2.14.0 | MIT + license thương mại cho `enterprise/` | GitHub |
| Firebase Admin / Pushy / Angus Mail | 9.11.0 / 0.15.6 / 2.0.5 | Apache-2.0 / MIT / EPL-2.0 hoặc EDL-1.0 | Maven Central |
| Centrifugo | 6.10.2 | Apache-2.0 | GitHub |
| LiteLLM | 1.106.4 | MIT + `enterprise/` | GitHub |
| Expo / React Native / `@better-auth/expo` | 58.0.19 / 0.88.0 / 1.7.13 | MIT | npm |
| Payload / Strapi / Directus | 3.92.0 / 5.59.0 / 12.5.1 | MIT / MIT + EE / MSCL-1.0 | GitHub |
| react-admin / Refine | 5.15.5 / 5.0.12 | MIT | npm |
| Scalar / OpenAPI Generator / hey-api | 1.76.0 / 7.26.0 / 0.99.0 | MIT / Apache-2.0 / MIT | npm, GitHub |
| Datafaker / Instancio / k6 | 2.7.0 / 6.1.0 / 2.3.0 | Apache-2.0 / Apache-2.0 / AGPL-3.0 | Maven Central, GitHub |
| Argo CD / Kargo / Copier / Backstage | 3.5.6 / 1.12.4 / 9.19.1 / 1.56.2 | Apache-2.0 / Apache-2.0 / MIT / Apache-2.0 | GitHub |

## 7. Kết luận

So với ABP hay JHipster, vyforge không thua ở kiến trúc; nó chỉ chưa có những "viên gạch" mà dự án nào cũng phải tự nặn lại. Giá trị lớn nhất nằm ở ba chỗ các nền tảng ngoại không có:

- tuân thủ luật dữ liệu cá nhân VN;
- thanh toán nội địa;
- xuất Excel/PDF tiếng Việt.

Rủi ro lớn nhất là **license và tính bền vững của OSS**. Chỉ trong 12 tháng đã có MinIO bị archive, Vault chuyển BUSL và Camunda bắt mua license cho production. Vì vậy mọi module đề xuất đều đứng sau một port của vyforge, và mỗi lựa chọn thư viện được ghi trong ADR kèm license. Khi một dự án OSS đổi hướng, chỉ phải thay adapter, không phải sửa code nghiệp vụ.

## Phụ lục: cạnh phụ thuộc của các module đề xuất (A -> B)

```
# starter Java (platform-java)
platform-starter-scheduling -> platform-starter-jpa
platform-starter-scheduling -> platform-starter-observability (optional)
platform-starter-ratelimit -> platform-starter-web
platform-starter-ratelimit -> Redis (optional; fallback Caffeine)
platform-starter-privacy -> platform-starter-jpa
platform-starter-privacy -> platform-starter-security
platform-starter-privacy -> platform-starter-audit
platform-starter-privacy -> platform-starter-scheduling
platform-starter-notification -> platform-starter-messaging
platform-starter-notification -> platform-starter-jpa
platform-starter-notification -> platform-starter-privacy
platform-starter-notification -> platform-starter-security
platform-starter-files -> platform-starter-web
platform-starter-files -> platform-starter-authz
platform-starter-files -> platform-starter-messaging
platform-starter-files -> platform-starter-jpa
platform-starter-files -> S3-compatible storage (SeaweedFS/RustFS/AWS S3)
platform-starter-files -> ClamAV (optional)
platform-starter-export -> platform-starter-scheduling
platform-starter-export -> platform-starter-files
platform-starter-export -> platform-starter-notification
platform-starter-export -> platform-starter-web
platform-starter-payment -> platform-starter-messaging
platform-starter-payment -> platform-starter-jpa
platform-starter-payment -> platform-starter-scheduling
platform-starter-payment -> platform-starter-webhooks (verify util)
platform-starter-payment -> platform-starter-audit
platform-starter-webhooks -> platform-starter-messaging
platform-starter-webhooks -> platform-starter-jpa
platform-starter-webhooks -> platform-starter-authz
platform-starter-features -> platform-starter-security
platform-starter-features -> platform-starter-authz (optional, principal.plan)
platform-starter-realtime -> platform-starter-security
platform-starter-realtime -> platform-starter-authz
platform-starter-realtime -> Redis
platform-starter-ai -> platform-starter-privacy
platform-starter-ai -> platform-starter-ratelimit
platform-starter-ai -> platform-starter-observability
platform-starter-ai -> PostgreSQL pgvector (optional)
platform-starter-jpa -> LocalizedText, @SoftDeletable, immutable_unaccent (bổ sung)
platform-test -> TestDataFactory (bổ sung)
platform-bom -> {scheduling, ratelimit, privacy, notification, files, export, payment, webhooks, features, realtime, ai} (manages)
# gateway, auth-server
platform-gateway -> platform-starter-ratelimit
platform-auth-server -> platform-starter-ratelimit
platform-auth-server -> platform-starter-notification (optional, thay mailer riêng)
# platform-web
@vyforge/privacy -> @vyforge/ui
@vyforge/privacy -> @vyforge/api-client
@vyforge/files -> @vyforge/ui
@vyforge/files -> @vyforge/api-client
@vyforge/notifications -> @vyforge/ui
@vyforge/notifications -> @vyforge/api-client
@vyforge/notifications -> @vyforge/query
@vyforge/features -> @vyforge/auth
@vyforge/ui -> DataTable/ResourceForm/PermissionGate (bổ sung)
@vyforge/query -> useEventStream (bổ sung)
# chart, hạ tầng
platform-observability (chart) -> platform-starter-observability (metric/trace names)
template-infra -> platform-helm-charts:app
template-infra -> platform-observability (chart)
template-infra -> .github:gitops-bump
template-infra -> External Secrets Operator
template-infra -> OpenBao | cloud secret manager
template-infra -> Grafana LGTM / Alloy
template-infra -> SeaweedFS (optional)
template-infra -> ClamAV (optional)
# template
template-spring-boot -> platform-starter-{scheduling,ratelimit,privacy} (mặc định)
template-spring-boot -> platform-starter-{notification,files,export,payment,webhooks,features,realtime,ai} (câu hỏi Copier)
template-spring-boot -> SeaweedFS, otel-lgtm (compose)
template-nextjs -> @vyforge/{privacy,files,notifications,features}
template-expo -> @vyforge/{api-client,i18n,query}
template-expo -> @vyforge/auth (core, sau khi tách)
template-expo -> platform-auth-server (public client + PKCE)
# CI
.github:openapi-sdk -> OpenAPI Generator
.github:load-test -> k6
.github:java-ci -> OpenAPI diff (bổ sung)
```

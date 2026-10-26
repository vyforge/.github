# Quy hoạch vyforge thành nền tảng nội bộ trưởng thành (Q4/2026 – Q4/2027)

*Báo cáo nghiên cứu, 26/10/2026. Phạm vi: toàn bộ hệ sinh thái trong `~/forge` (`.github`, `platform-*`, `template-*`).
Bằng chứng trích theo `repo/file:dòng`. Mốc phiên bản lấy từ nguồn công khai tại thời điểm viết. Phần nào chưa kiểm chứng
được thì ghi rõ.*

Tài liệu này là **quy hoạch tổng thể**. Nó trả lời câu hỏi: vyforge cần thêm gì ngoài code để thành một internal developer
platform (IDP) trưởng thành, và nên làm theo thứ tự nào. Chi tiết từng module dùng chung còn thiếu (thông báo, lưu file,
job, feature flag, thanh toán VN…) thuộc báo cáo song song `docs/research/2026-10-module-thong-dung.md` (nhánh
`docs/research-common-modules`, lúc viết chưa merge). Ở đây chỉ nhắc tới các module đó khi chúng ảnh hưởng đến kiến trúc
đích hoặc lộ trình.

## Tóm tắt

- **Về mức trưởng thành, vyforge đang ở khoảng giữa mức 1 và mức 2** của CNCF Platform Engineering Maturity Model. Phần
  giao diện cho developer (template Copier, reusable workflow, artifact có version) đã đạt mức 2. Ba chiều đầu tư, mức
  dùng và đo lường vẫn ở mức 1: chỉ có một người làm, chưa có dự án nào dùng, và chưa đo gì.
- **Code đi trước vận hành.** Thư viện, IdP, IAM và gateway đã khá đầy đủ. Còn thiếu môi trường chạy (chưa có repo
  infra, `gitops-bump` chưa được gọi), cam kết dịch vụ (SLO, on-call, DR), chính sách version và quy trình nâng cấp hàng
  loạt. Đây là các điều kiện để dự án khách hàng dám dùng platform trên production.
- **Rủi ro lớn nhất là con người, không phải công nghệ.** Bus factor bằng 1, trong khi `platform-auth-server` là
  identity provider tự viết và ADR-0008 đòi vá bảo mật trong 7 ngày cùng tối thiểu 2 owner.
- **Lộ trình 5 quý.** Q4/2026 khép lại các việc đang làm (sửa authz, tenant, IdP mở rộng, UI phân quyền) và ra bản đầu
  tiên của các service. Q1/2027 dựng runtime và GitOps. Q2/2027 đưa pilot tier 1 lên production. Q3/2027 chạy pilot tier 2
  và golden path mobile. Q4/2027 đo lại mức trưởng thành, với mục tiêu đạt mức 2 ở mọi chiều và mức 3 ở chiều giao diện.

## 1. Hiện trạng và mức trưởng thành

### 1.1 Thang đo

Bản 1.0 của [CNCF Platform Engineering Maturity Model](https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/)
(TAG App Delivery) có 4 mức: **Provisional → Operationalized → Scalable → Optimizing**. Nó chấm theo 5 chiều:

| Chiều | Mức 1 Provisional | Mức 2 Operationalized | Mức 3 Scalable | Mức 4 Optimizing |
|---|---|---|---|---|
| Investment | Người làm tự nguyện hoặc tạm thời | Team riêng, cấp ngân sách tập trung (cost center) | Cấp vốn như product: PM, UX, phân bổ dựa trên dữ liệu | Hệ sinh thái: chuyên gia khác cùng mở rộng platform |
| Adoption | Rời rạc, không có chiến lược chung | Đẩy từ ngoài: bắt buộc, khuyến khích | Kéo từ trong: người dùng tự chọn vì giá trị | Người dùng đóng góp ngược |
| Interfaces | Quy trình thủ công, truyền tay | Công cụ chuẩn, template "paved road" | Self-service một cú nhấp, trải nghiệm nhất quán | Tích hợp sẵn vào công cụ đang dùng |
| Operations | Làm theo yêu cầu, bị động | Có danh mục và owner, nâng cấp thủ công | Vận hành tập trung, quy trình delivery chuẩn | Dịch vụ quản lý, vòng đời tự động |
| Measurement | Tuỳ hứng, phản hồi không chính thức | Thu thập nhất quán qua kênh chuẩn | Dữ liệu nhắm vào kết quả chiến lược | Văn hoá dữ liệu toàn tổ chức |

### 1.2 Chấm điểm vyforge (26/10/2026)

| Chiều | Mức | Bằng chứng |
|---|---|---|
| **Investment** | **1** | API `orgs/vyforge/members` trả về 1 thành viên. Commit của mọi repo đều thuộc một người (tên `Hoang Lam`/`Harley Hoang`) cộng các commit do công cụ sinh. `platform-auth-server/.github/CODEOWNERS:1-2` ghi `@vyforge/devops`, kèm chú thích "thay bằng team owner khi đã lập". Chưa có team platform được cấp nguồn lực riêng. |
| **Adoption** | **1** (thiết kế cho mức 2) | `ARCHITECTURE.md` có 20 quy tắc **bắt buộc** B1–B20, tức là đã chọn chiến lược "đẩy" của mức 2. Nhưng `~/forge` **chưa có repo dự án nào** sinh từ template (F19 trong kiểm kê). Người dùng duy nhất là các service nền tảng, như `platform-auth-admin` sinh từ `template-nextjs` (`platform-auth-admin/README.md:4`). |
| **Interfaces** | **2** | Có paved road: `copier copy` cho backend và frontend (`template-spring-boot/README.md`, `template-nextjs/README.md`), 7 reusable workflow `@v1` (`.github/.github/workflows/`), preset Renovate (`.github/renovate-config.json`), artifact có version (`platform-java` v0.8.0, `platform-web` v0.6.0, chart OCI v0.2.0). Chưa có self-service cho hạ tầng: không có `template-infra`, không có portal, mở môi trường vẫn phải làm tay. |
| **Operations** | **1–2** | Phía mức 2: có danh mục repo/owner (`REPOSITORY_GUIDELINES.md`), có semver tag cho thư viện và template, có Renovate. Phía mức 1: 4 service chưa có tag nào; hai template dùng `platform-parent` 0.7.0 trong khi bản mới nhất là 0.8.0 (`template-spring-boot/template/pom.xml.jinja:10`); `gitops-bump` chưa được repo nào gọi (`platform-gateway/.github/workflows/cd.yml:27`); không có SLO, runbook, kế hoạch backup hay on-call. Tìm `runbook`, `SLO`, `on-call`, `backup` trong các README đều không ra kết quả. |
| **Measurement** | **1** | Không đo DORA, không khảo sát developer, không theo dõi mức dùng template. Tín hiệu duy nhất là CI xanh/đỏ. |

**Kết luận:** vyforge đang ở giai đoạn mà Team Topologies gọi là
[thinnest viable platform](https://teamtopologies.com/key-concepts-content/what-is-a-thinnest-viable-platform-tvp): phần
nền dùng được, nhưng chưa được vận hành như một product. Số liệu DORA 2025 cũng nên lưu ý ở đây. Khảo sát của DORA ghi
nhận khoảng 90% tổ chức có "platform" nội bộ. Tuy nhiên DORA định nghĩa platform rất rộng: "tập năng lực dùng chung cho
nhiều ứng dụng" (theo [bài phân tích này](https://www.youngju.dev/blog/2026-07-16-dora-platform-engineering-evidence.en),
nguồn thứ cấp). Trang capability của DORA còn mô tả
[đường cong chữ J](https://dora.dev/capabilities/platform-engineering/): hiệu suất có thể giảm trong giai đoạn đầu khi độ
phức tạp tăng, trước khi platform trưởng thành. Như vậy có platform chưa đủ. Platform phải được dùng thật và được đo thật
thì mới có giá trị.

### 1.3 Điểm mạnh nên giữ

- Thiết kế hai lớp đúng hướng: phần dùng chung nằm trong **artifact có version**, Renovate tự nâng; phần khung mỏng sinh
  bằng Copier (research `2026-10-kien-truc-template.md`, mục Kết luận).
- Ranh giới được công cụ kiểm tra: Modulith `verify()`, ArchUnit, ESLint (B3, B12).
- Bảo mật chuỗi cung ứng có nền: ghim action theo SHA, Trivy, SBOM (`.github/.github/workflows/container.yml:95-129`).
- Bộ IdP/IAM khá hoàn chỉnh so với một team nhỏ: OIDC, MFA, token exchange RFC 8693, policy kiểu IAM, chế độ remote, xoá
  cache qua Redis. Đã đối chiếu từng ý với ADR-0008/0009/0010 trong báo cáo `5-adr-gap`.

## 2. Kiến trúc đích 12–24 tháng

Nguyên tắc: **mỗi tầng chọn một mặc định và có một phương án dự phòng**. Ưu tiên dịch vụ managed khi khách hàng cho
phép. Chỉ tự vận hành phần nào thực sự khác biệt. Đội nhỏ không đủ người để tự chạy mọi thứ.

### 2.1 Golden path

| Đường | Hiện có | Đích (Q3/2027) |
|---|---|---|
| **Backend** | `template-spring-boot` v1.4.0: tier 0–2, Kafka/Redis/IAM tuỳ chọn, compose vẫn dùng Keycloak | Mặc định dùng `platform-auth-server` trong compose và Testcontainers (ADR-0008 §Hệ quả). Có câu hỏi `iam_mode: local\|remote`. Sinh sẵn `values-*.yaml` cho repo infra. Theo `platform-java` 1.x |
| **Frontend** | `template-nextjs` v1.3.0: BFF Better Auth, provider vẫn cấu hình cho Keycloak | Provider `vyforge` trỏ tới auth-server; `@vyforge/ui` ≥ 0.4; có tuỳ chọn dùng gateway làm `API_BASE_URL` |
| **Mobile** | Chưa có. `REPOSITORY_GUIDELINES.md:13` mới chỉ có tên thành phần `mobile` | `template-expo`: [Expo SDK 55](https://expo.dev/changelog/sdk-55) (React Native 0.83, React 19.2, **chỉ New Architecture**), đăng nhập OIDC kiểu native app (auth code + PKCE, public client, redirect bằng custom scheme), token lưu trong secure storage, gọi API qua `platform-gateway`. Dùng lại `@vyforge/i18n` và type sinh từ OpenAPI. Cần ADR riêng vì khác mô hình BFF của ADR-0006: token nằm trên thiết bị |
| **Hạ tầng dự án** | Chưa có (F1) | `template-infra`: Argo CD Application/ApplicationSet, values cho `platform-helm-charts/app`, overlay `dev/staging/prod` |

### 2.2 Nền tảng runtime

- **Kubernetes.** Bám hai minor được hỗ trợ gần nhất. Theo trang [releases của Kubernetes](https://kubernetes.io/releases),
  1.35 hết hạn 28/02/2027, còn 1.36 (khoảng 05/2026) hết hạn 28/06/2027. Như vậy mỗi cụm phải nâng **2 lần mỗi năm**.
  Đây là lý do mạnh để ưu tiên K8s managed. Với khách hàng on-prem, dùng một bản phân phối có công cụ nâng cấp, ví dụ K3s
  (đã có [bản 1.36](https://docs.k3s.io/release-notes/v1.36.X)). Việc chọn nơi chạy cần một ADR riêng.
- **GitOps.** Dùng [Argo CD 3.x](https://endoflife.date/argo-cd). Bản 3.2 đã hết hỗ trợ ngày 04/08/2026; dòng mới nhất là
  3.5 (3.5.5, 12/10/2026). Argo CD đọc repo `<du-an>-infra`, còn CI chỉ bump digest ở dev (`gitops-bump.yml`, B16).
  [Kargo](https://github.com/akuity/kargo/releases) (dòng hiện tại 1.10/1.11) dùng để promote cùng một digest
  dev → staging → prod qua PR có duyệt, đúng như K11 đã gợi ý.
- **Môi trường.** Mỗi dự án có `dev` (tự deploy từ `main`), `staging` (giống prod, dùng cho UAT với khách) và `prod`. Ba
  môi trường dùng chung một image digest. Cấu hình đọc từ env runtime: B14 cho Next.js, profile cho Spring.
- **Preview environment theo PR.** Dùng ApplicationSet với
  [Pull Request generator](https://oneuptime.com/blog/post/2026-02-09-argocd-pr-preview-environments/view): mỗi PR có nhãn
  `preview` được một namespace và một URL riêng, đóng PR thì tự xoá. Ban đầu chỉ bật cho frontend và cho backend tier ≥ 1.
  Database dùng một cluster CloudNativePG nhỏ mỗi PR, hoặc một database tạm trong cluster dùng chung. Mỗi PR một
  Application sẽ tốn tài nguyên khi số service và số PR tăng
  ([Signadot](https://www.signadot.com/articles/argocd-preview-environments/)), nên phải có TTL và hạn mức.
- **Helm.** Chart `app` đã đủ các thành phần an toàn (digest, non-root, NetworkPolicy, ExternalSecret).
  `platform-helm-charts/charts/app/Chart.yaml:6` vẫn ghi 0.1.0, cần đồng bộ với tag. Helm 3 hết vá bảo mật từ 11/11/2026
  (research cũ), nên CI và Argo CD phải chạy Helm 4.

### 2.3 Danh tính và quyền

| Thành phần | Đích |
|---|---|
| `platform-auth-server` | Ra bản 1.0 sau **pentest** (ADR-0008). Phát `tenant_id` và có API danh bạ (ADR-0011, đang làm). Thêm passkey, SAML, LDAP, Zalo (ADR-0012, đang làm). Không có HA đặc biệt: đã chạy được nhiều replica nhờ trạng thái nằm trong PostgreSQL (README §Vận hành) |
| Phương án dự phòng | Keycloak. Lưu ý Keycloak **không có LTS**, chỉ bản mới nhất được vá ([Skycloak](https://skycloak.io/blog/keycloak-25-released-new-features-and-enhancements/)). Bản mới nhất là 26.7.0 (07/2026). Dự án nào chọn Keycloak thì phải theo nhịp nâng cấp khoảng 3 tháng một lần |
| `platform-iam-server` + `platform-starter-authz` | Sửa các lỗi trong báo cáo `1-authz-starter` và `2-iam-server` (đang làm ở nhánh `fix/authz-hardening`, 0.9.0). Bắt buộc claim tenant qua `vyforge.authz.tenant.required`. Lọc danh sách theo policy ở tầng truy vấn (F9, phần còn lại) |
| `platform-gateway` | Rate limit phân tán bằng Redis (F20). Route theo service sao cho `/me/evaluate` chạy trong app (vấn đề #2 của `5-adr-gap`). Nâng lên Spring Cloud 2026.0 cùng lúc với Boot 4.2 |
| Danh tính workload | Service gọi service bằng `client_credentials` hoặc token exchange. Chưa cần mTLS/SPIFFE; xem lại khi có tier 3 |
| Admin | MFA bắt buộc (đã có ở auth-admin). Thêm quy trình cấp và thu hồi quyền admin vào checklist on-boarding/off-boarding |

### 2.4 Dữ liệu

- **PostgreSQL HA trên K8s.** Dùng [CloudNativePG 1.30](https://www.postgresql.org/about/news/cloudnativepg-1300-released-3337/)
  (06–07/2026). Bản này mặc định PostgreSQL 18.4, hỗ trợ K8s 1.36, có CRD `DatabaseRole` hợp với GitOps và cơ chế chọn
  primary bằng Lease. Dòng 1.28 hết hạn 30/06/2026, dòng 1.29 hết hạn 29/09/2026. Nếu cloud của khách có PostgreSQL managed
  thì dùng bản managed.
- **Backup và PITR.** Dùng [Barman Cloud Plugin](https://cloudnative-pg.io/plugin-barman-cloud/docs/intro/), không dùng
  `barmanObjectStore` có sẵn trong core: phần này đã deprecated và
  [có lịch gỡ bỏ](https://cloudnative-pg.io/docs/1.27/release_notes/v1.27). Backup ra object storage tương thích S3, có WAL
  archive để khôi phục về một thời điểm bất kỳ. **Mỗi quý diễn tập khôi phục một lần**; backup chưa từng restore thì không
  được tính là có backup.
- **Migration.** Flyway (ADR-0007). Thay đổi phá vỡ tương thích phải theo kiểu expand/contract để rolling update không
  downtime (chart dùng `maxUnavailable: 0`). Điều này cần một ADR. Contract của tab 2/3 cũng đã cho thấy cần quy ước dải số
  migration khi nhiều nhánh cùng sửa một service.
- **Database của platform.** Auth-server, IAM server và database của từng app tách riêng (ADR-0008, ADR-0010). Auth-server
  có khoá ký JWK mã hoá AES-256-GCM. Master key nằm trong secret, nên phương án DR phải sao lưu secret đó cùng với database.

### 2.5 Messaging, observability, bảo mật chuỗi cung ứng

- **Messaging.** Kafka 4.x **chỉ còn chế độ KRaft**
  ([Instaclustr](https://www.instaclustr.com/support/documentation/announcements/apache-kafka-and-kafka-connect/instaclustr-for-apache-kafka-and-kafka-connect-4-1-1-are-generally-available/)).
  Trên K8s dùng operator Strimzi; chỉ dự án tier 2 trở lên mới cần. Redis dùng cho xoá cache IAM và cho rate limit phân tán.
  Outbox của Modulith đã có sẵn trong `platform-starter-messaging`.
- **Observability.** Chuẩn thu thập là OpenTelemetry (đã có trong `platform-starter-observability`). OTel Collector đẩy dữ
  liệu vào bộ LGTM: Loki, Grafana, Tempo, Mimir (Mimir 3.0 và Tempo 2.9 ra mắt
  [11/2025](https://grafana.com/press/2025/11/05/grafana-labs-launches-mimir-3-0-expanding-open-observability-at-scale-at-kubecon--cloudnativecon-north-america-2025/)).
  Dashboard mẫu và alert theo SLO đóng gói kèm chart. Chi tiết thuộc báo cáo module.
- **Chuỗi cung ứng.** Khung tham chiếu là [SLSA v1.2](https://slsa.dev/blog/2025/11/announce-slsa-v1.2) (11/2025). Build
  Track L1–L3 giữ nguyên; bản này thêm **Source Track** (lịch sử, provenance và kiểm soát review của mã nguồn).
  - *Hiện trạng:* image có SBOM và provenance của BuildKit, có Trivy. **Artifact attestation của GitHub bị tắt cho repo
    private** (`container.yml:125-129`), vì tính năng này
    [cần GitHub Enterprise Cloud](https://docs.github.com/de/actions/how-tos/security-for-github-actions/using-artifact-attestations/using-artifact-attestations-to-establish-provenance-for-builds).
    Repo private vì vậy đang thiếu chữ ký kiểm chứng được.
  - *Đích:* ký image bằng **cosign v3**. Từ v3, định dạng bundle của Sigstore là
    [mặc định](https://blog.sigstore.dev/cosign-3-0-available/). Với repo private, nên ký bằng khoá trong KMS/OpenBao thay
    vì keyless trên Rekor công khai, vì log minh bạch công khai sẽ lộ tên repo và workflow. Admission dùng **Kyverno**
    `ImageValidatingPolicy`: bản 1.16 có ở v1beta1, theo
    [tài liệu hiện tại](https://kyverno.io/docs/policy-types/overview/) đã stable ở `policies.kyverno.io/v1` từ 1.18 (một
    trang khác ghi 1.19, chưa đối chiếu được). Policy kiểm chữ ký, bắt buộc dùng digest, non-root. Chạy chế độ `Audit`
    trước, sau một quý mới chuyển sang `Enforce`.
  - *Mức SLSA mục tiêu:* Build L2 (provenance có chữ ký, build trên runner hosted) trong Q1/2027. Hướng tới L3 bằng cách
    tách bước ký vào reusable workflow mà caller không sửa được. Source Track L2 dựa trên branch protection và ruleset của
    org (F4).

### 2.6 Secrets và chi phí

- **Secrets.** Dùng External Secrets Operator với API `v1` (chart đã hỗ trợ ExternalSecret). Phải ghim **≥ 1.2.0** vì
  CVE-2026-22822 cho phép đọc secret khác namespace qua hàm template `getSecretKey`
  ([tóm tắt](https://dbugs.ptsecurity.com/researchers/Budimanjojo), chưa đọc advisory gốc). Backend là secret manager của
  cloud. Khi on-prem thì dùng [OpenBao](https://openbao.org/community/release-notes/2-5-0/) (fork mã nguồn mở của Vault,
  dòng 2.5.x). Xoay vòng: mật khẩu database qua CloudNativePG/OpenBao, client secret qua `platform-auth-admin` (đã có chức
  năng xoay secret).
- **FinOps.** Dùng [OpenCost](https://www.cncf.io/blog/2024/10/31/opencost-advances-to-the-cncf-incubator/) (CNCF
  incubating từ 10/2024). Mọi tài nguyên phải có nhãn `vyforge.io/project`, `vyforge.io/env`, `vyforge.io/tier`; chart
  `app` sẽ bắt buộc các nhãn này. Báo cáo showback theo dự án khách hàng, vì với mô hình outsource chi phí phải tính được
  theo từng hợp đồng. Dữ liệu billing của cloud xuất theo chuẩn
  [FOCUS 1.2](https://focus.finops.org/wp-content/uploads/2025/05/FOCUS-spec-v1_2.pdf) khi nhà cung cấp hỗ trợ. Preview
  environment phải có TTL và quota.

## 3. Năng lực còn thiếu ngoài code module

### 3.1 Quản trị version, release và deprecation

Hiện tại `platform-java` có 8 minor trong 6 ngày (v0.1.0 → v0.8.0, 21–26/10/2026). Ở giai đoạn 0.x thì điều này chấp nhận
được. Nhưng dự án khách hàng cần một lời hứa ổn định.

- **Semver nghiêm.** `platform-java` và `platform-web` lên **1.0.0 khi pilot tier 1 chạy production** (Q2/2027). Sau 1.0:
  đổi API public hoặc đổi giá trị mặc định có ảnh hưởng hành vi bảo mật thì phải tăng major.
- **Gắn với nhịp của Spring Boot.** Spring Boot ra minor vào **tháng 5 và tháng 11**, mỗi minor được hỗ trợ OSS
  [tối thiểu 12 tháng](https://github.com/spring-projects/spring-boot/wiki/Supported-Versions). 4.2.0-M1 đã ra
  [20/08/2026](https://spring.io/blog/2026/08/20/spring-boot-4-2-0-M1-available-now/), nên 4.2 GA dự kiến 11/2026. Một
  nguồn thứ cấp ghi 4.1 hết OSS ngày 31/07/2027 (chưa xác minh trên spring.io). Đề xuất:
  - Mỗi minor của Boot ứng với một **dòng release** của `platform-java`.
  - Hỗ trợ đồng thời **2 dòng**: dòng mới nhất nhận tính năng, dòng trước chỉ nhận bản vá bảo mật cho đến khi Boot tương
    ứng hết OSS.
  - **Không có LTS dài hơn Boot OSS**, vì đội không đủ người để backport.
- **Ma trận tương thích**, công bố trong README mỗi repo và trong ADR:

  | platform-java | Spring Boot | Spring Cloud (gateway) | Java | Hỗ trợ đến |
  |---|---|---|---|---|
  | 0.9.x → 1.x | 4.1.x | 2025.1.x | 25 | Hết OSS của Boot 4.1 |
  | 2.x (dự kiến) | 4.2.x | 2026.0.x | 25 | Hết OSS của Boot 4.2 |

  | platform-web | Next.js | React | Node | pnpm |
  |---|---|---|---|---|
  | 0.6.x → 1.x | ≥ 16.4.1 | 19.x | 24 LTS (Active LTS đến 20/10/2026, EOL 30/04/2028) | 11 |
  | sau Q1/2027 | 16.x / 17.x | 19.x | 24 hoặc 26 LTS ([LTS từ khoảng 28/10/2026](https://endoflife.date/nodejs), EOL 04/2029) | 11/12 |

  Từ Node 27 trở đi, Node chuyển sang [phát hành mỗi năm một bản](https://nodejs.org/en/about/releases/). Rà soát hằng quý
  cần tính đến thay đổi này.
- **Deprecation.**
  - Java: `@Deprecated(forRemoval = true, since = "x.y")`. Frontend: `@deprecated` trong JSDoc.
  - API bị đánh dấu sống ít nhất **một minor**, kèm ghi chú trong CHANGELOG và hướng dẫn chuyển đổi.
  - Thay đổi lặp lại trên nhiều repo thì kèm recipe (OpenRewrite cho Java, codemod cho TS).
  - Hiện chưa repo nào có `CHANGELOG.md`. Cần sinh tự động từ conventional commit (đã bắt buộc bằng `pr-title-lint`).
- **Service** (`auth-server`, `iam-server`, `gateway`, `auth-admin`) cũng theo semver, nhưng version gắn với **API và
  migration**. Bump major khi migration không đảo ngược được hoặc khi đổi contract (`contracts.md` của đợt này là một ví
  dụ). Cần ra tag đầu tiên ngay trong Q4/2026.

### 3.2 Nâng cấp hàng loạt cho dự án con

| Lớp | Cơ chế | Việc cần làm |
|---|---|---|
| Artifact có version (parent, starter, `@vyforge/*`, chart, workflow `@v1`) | Renovate với preset của org (ADR-0004) | Thêm `packageRules` gom `com.vyforge.platform:*` và `@vyforge/*` thành một PR "platform", không chờ 3 ngày cho package nội bộ. Hai template hiện **không có `renovate.json`** ở gốc repo template (chỉ có trong `template/`), nên chính template không tự nâng |
| Khung Copier | Renovate có [manager `copier`](https://docs.renovatebot.com/modules/manager/copier/) đọc `.copier-answers.yml` và chạy `copier update --skip-answered --defaults --vcs-ref <tag>` | Template của vyforge cần `--trust` (README). Vì vậy phải dùng Renovate **self-host** (≥ 44.3.1, ADR-0004) với `allowScripts`. Lỗi đã biết: xung đột không làm check fail ([#31600](https://docs.renovatebot.com/modules/manager/copier/)). Phương án thay thế là workflow định kỳ `copier-update.yml` trong `.github` (F3), tự mở PR và fail khi có file `.rej` |
| Thay đổi code | OpenRewrite (Java), codemod (TS) | Mỗi bản major của `platform-*` kèm recipe |
| Theo dõi độ lệch | Script hoặc workflow đọc `.copier-answers.yml` (`_commit`) và version parent/`@vyforge/*` của mọi repo trong org | Bảng "fleet": repo nào đang chậm mấy bản. Dữ liệu này cũng dùng cho chỉ số adoption ở 3.3 |

Mục tiêu đo được: **90% repo dự án dùng dòng `platform-*` còn được hỗ trợ; bản vá bảo mật lan tới mọi repo trong 7 ngày**.
Con số 7 ngày khớp với ADR-0008.

### 3.3 Đo lường

- **DORA.** Báo cáo [2025 State of AI-assisted Software Development](https://dora.dev/research/2025/dora-report/) đổi tên
  báo cáo cũ và thêm chỉ số thứ năm là **rework rate**. Các chỉ số chia làm hai nhóm: *throughput* gồm lead time,
  deployment frequency, thời gian khôi phục; *instability* gồm change failure rate và rework rate. Báo cáo thay 4 hạng
  hiệu suất bằng 7 archetype
  ([tóm tắt](https://redmonk.com/rstephens/2025/12/18/dora2025/), nguồn thứ cấp). Kết luận chính: AI **khuếch đại** điểm
  mạnh và điểm yếu sẵn có; mức dùng AI tăng throughput nhưng đi kèm độ ổn định giảm. Điều này liên quan trực tiếp vì một
  phần code platform được sinh bằng công cụ AI. Nguồn dữ liệu: GitHub (PR, deploy qua `container.yml`), Argo CD/Kargo
  (deploy và rollback), issue có nhãn `incident`.
- **SPACE.** Khung [SPACE](https://www.microsoft.com/en-us/research/?p=733804) của Forsgren và cộng sự (ACM Queue 19(1),
  2021) có 5 chiều: satisfaction, performance, activity, communication, efficiency/flow. Áp dụng bằng một khảo sát ngắn mỗi
  quý (khoảng 10 câu) cho developer dự án. Không dùng activity như số commit để chấm cá nhân.
- **Mức dùng golden path:**
  - % repo dự án mới tạo bằng `copier copy` (có `.copier-answers.yml`)
  - % repo theo kịp template ≤ 1 minor
  - % repo dùng dòng `platform-*` còn được hỗ trợ
  - **time-to-first-deploy** của dự án mới (mục tiêu dưới 1 ngày, theo lộ trình trong research cũ)
  - số ngoại lệ có ADR (ví dụ Gradle, MongoDB)
- **Mức trưởng thành.** Chấm lại bảng 1.2 mỗi nửa năm.

### 3.4 Tài liệu, dev portal, hỗ trợ, SLO

- **Tài liệu trước, portal sau.** Theo tinh thần thinnest viable platform: dựng một site tài liệu tĩnh từ `.github/docs`
  cộng README các repo. Bổ sung: "bắt đầu dự án trong 1 ngày", runbook cho từng service nền tảng, trang ma trận tương
  thích. [Backstage](https://backstage.io/docs/releases/v1.49.0/) đến 1.49 mới có release candidate của New Frontend
  System 1.0. Chi phí vận hành Backstage chỉ đáng bỏ ra khi có từ khoảng 15–20 repo dự án và cần self-service tạo repo/môi
  trường. Quyết định lại ở Q4/2027.
- **Hỗ trợ.**
  - Kênh: GitHub Discussions của `.github` cho câu hỏi; issue template "platform bug" và "yêu cầu tính năng".
  - Office hours mỗi tuần với các dự án pilot.
  - Mức phản hồi: trong giờ hành chính, P1 dưới 4 giờ, P2 dưới 2 ngày làm việc.
- **On-call.** Chưa thể có on-call 24/7 khi chỉ có một người. Hai việc phải làm: (1) ghi rõ điều này trong hợp đồng với
  khách hàng; (2) thiết kế để lỗi platform không làm sập app. IAM đã có stale cache 10 phút; JWKS được cache ở resource
  server. Khi có người thứ 2 và thứ 3 thì luân phiên on-call trong giờ làm việc.
- **SLO cho service nền tảng.** Theo [SRE Workbook](https://oreilly.com/library/view/the-site-reliability/9781492029496):
  định nghĩa SLI, SLO và error budget policy (ví dụ hết budget thì dừng tính năng, ưu tiên sửa lỗi). Đề xuất ban đầu:

  | Service | SLI | SLO (28 ngày) |
  |---|---|---|
  | auth-server | Tỉ lệ thành công của `/oauth2/token` và JWKS; p95 đăng nhập | 99,9%; p95 < 500 ms |
  | iam-server | Tỉ lệ thành công của `POST /internal/resolve`; p95 | 99,5% (app có stale cache); p95 < 100 ms |
  | gateway | Tỉ lệ request không bị lỗi 5xx do chính gateway; p99 overhead | 99,9%; p99 < 20 ms |

  Để có SLO thì cần metric. Báo cáo `1-authz-starter` §5 đã chỉ ra starter authz chưa có metric nào: allow/deny, cache
  hit/miss, độ trễ remote.

### 3.5 DR/BCP, tuân thủ, pentest

- **DR/BCP.** Mỗi tier có RPO/RTO riêng. Đề xuất: tier 1 RPO 15 phút / RTO 4 giờ; service nền tảng RPO 5 phút / RTO 1
  giờ. Các thành phần:
  - **Git** là nguồn sự thật để dựng lại cluster (Argo CD).
  - **Backup PostgreSQL và PITR** ở vùng hoặc nhà cung cấp khác.
  - **Secret gốc** (master key JWK, khoá ký image) có bản sao offline.
  - **Diễn tập** mỗi 6 tháng, ghi biên bản.
  - **Phụ thuộc GitHub.** GitHub Packages và GHCR là điểm lỗi đơn của toàn bộ build. Nên có registry mirror hoặc cache
    (bộ nhớ đệm Maven/npm/OCI) cho khách hàng on-prem.
- **Tuân thủ pháp luật Việt Nam.** **Luật Bảo vệ dữ liệu cá nhân số 91/2025/QH15** thông qua ngày 26/06/2025, **có hiệu
  lực từ 01/01/2026** ([Thư viện Pháp luật](https://thuvienphapluat.vn/van-ban/Bo-may-hanh-chinh/Law-91-2025-QH15-Personal-Data-Protection-665440.aspx)).
  **Nghị định 356/2025/NĐ-CP** ngày 31/12/2025 hướng dẫn luật này và **thay thế Nghị định 13/2023/NĐ-CP** từ 01/01/2026
  ([Công báo](https://congbao.chinhphu.vn/van-ban/nghi-dinh-so-356-2025-nd-cp-468371.htm),
  [LuatVietnam](https://luatvietnam.vn/dan-su/diem-moi-cua-nghi-dinh-356-2025-so-voi-nghi-dinh-13-2023-ve-bao-ve-du-lieu-ca-nhan-568-106269-article.html)).
  Vì vậy brief và các tài liệu khác nhắc "Nghị định 13/2023" cần cập nhật sang Luật 91 và Nghị định 356. Các điểm ảnh
  hưởng trực tiếp tới platform (theo bản tóm tắt của LuatVietnam, cần đối chiếu nguyên văn):
  - **Lưu bằng chứng đồng ý** kiểm chứng được về thời điểm và nội dung; cấm đồng ý mặc định (Điều 6). → `auth-server` và
    template cần một module consent.
  - **Thời hạn xử lý yêu cầu của chủ thể dữ liệu** (Điều 5): rút đồng ý trong 2 ngày làm việc, cung cấp hoặc chỉnh sửa
    trong 10 ngày, xoá trong 20 ngày. → Cần API xuất và xoá tài khoản kèm dữ liệu liên quan trong mọi app. Đây là lý do
    danh bạ và `sub` phải thống nhất (ADR-0011).
  - **Xử lý dữ liệu lớn phải có MFA và phân quyền truy cập** (Điều 9). Platform đã có MFA và IAM.
  - **Thông báo vi phạm**: 72 giờ với ngành tài chính, ngân hàng (Điều 8). Hồ sơ đánh giá tác động và hồ sơ chuyển dữ liệu
    ra nước ngoài có biểu mẫu kèm nghị định. Nhân sự bảo vệ dữ liệu phải có trình độ cao đẳng trở lên, 2 năm kinh nghiệm
    và đã qua đào tạo (khoản 2 Điều 13).
  - Platform cung cấp công cụ (consent, export/delete, masking, audit). Trách nhiệm pháp lý thuộc bên kiểm soát dữ liệu,
    thường là khách hàng. Cần ADR phân định rõ.
- **ISO 27001 / SOC 2.** Chứng chỉ ISO/IEC 27001:2013 đã hết hiệu lực từ
  [31/10/2025](https://dnv.com/assurance/Management-Systems/new-iso/transition/transition-to-iso-iec-27001-2022), nên nếu
  làm thì làm theo bản **2022**. Chỉ nên làm khi có khách hàng yêu cầu trong hợp đồng: khách ngân hàng hoặc tài chính
  thường đòi ISO 27001, khách Mỹ thường đòi SOC 2. Trước đó vẫn nên áp các control rẻ: quản lý truy cập theo team, log
  tập trung, quản lý thay đổi qua PR, quét lỗ hổng. Phần lớn các việc này đã có hoặc nằm trong lộ trình, nên khi cần
  chứng chỉ thì khoảng cách không còn lớn.
- **Quy trình pentest.**
  - Phạm vi theo [OWASP ASVS 5.0](https://owasp.org/projects/asvs?tab=main) (05/2025, 17 chương, khoảng 350 yêu cầu):
    **Level 3** cho `platform-auth-server`, **Level 2** cho iam-server, gateway, auth-admin và template.
  - Thực hiện trước lần đầu lên production, sau đó mỗi năm và sau mỗi thay đổi lớn ở luồng đăng nhập (ADR-0008). Thay
    đổi của tab 3 (passkey, SAML, LDAP, Zalo) là một thay đổi lớn như vậy.
  - Đơn vị bên ngoài thực hiện. Phát hiện được theo dõi bằng issue riêng tư (GitHub Security Advisory). SLA sửa: critical
    7 ngày, high 30 ngày.
  - ASVS 5.0 ghi rõ chỉ kiểm thử hộp đen là không đủ, nên pentest phải được truy cập mã nguồn.

## 4. Lộ trình theo quý (Q4/2026 – Q4/2027)

| Quý | Mục tiêu | Đầu ra cụ thể | Tiêu chí hoàn thành | Phụ thuộc |
|---|---|---|---|---|
| **Q4/2026** (đến 31/12) | Khép các việc đang làm; ra bản đầu tiên của các service | • `platform-java` 0.9.0: sửa authz (afterCommit invalidation, fail-open của `NotResource`, self-grant, `tenant.required`) — nhánh `fix/authz-hardening`<br>• `platform-auth-server`: tenant và danh bạ (ADR-0011, V020–V029), IdP mở rộng (ADR-0012, V030–V039)<br>• `platform-auth-admin`: UI tenant/danh bạ, IdP, khu "Phân quyền" (IAM)<br>• Tag **v0.1.0** cho auth-server, iam-server, gateway, auth-admin<br>• Template: chuyển sang auth-server, `platform-*` 0.9/`@vyforge/ui` 0.4, `iam_mode`<br>• Compose E2E dùng chung (auth + gateway + iam + 2 backend + BFF)<br>• ADR-0013 (version/release), ADR-0014 (nâng cấp hàng loạt)<br>• Đánh giá Boot 4.2 GA và Node 26 LTS | Merge đủ 4 nhánh của đợt này; E2E xanh trong CI; app mới sinh từ template đăng nhập được bằng auth-server mà không cần sửa tay | Contract chung (`contracts.md`); merge 0.9.0 trước template |
| **Q1/2027** | Có môi trường chạy thật | • Repo `template-infra` + `vyforge-platform-infra` (Argo CD 3.5, CloudNativePG 1.30 + Barman plugin, ESO ≥ 1.2, OTel Collector + LGTM, Kyverno ở chế độ Audit)<br>• Bật `gitops-bump` trong `cd.yml` của các service<br>• Ký cosign trong `container.yml@v1`<br>• Workflow `copier-update.yml` hoặc Renovate self-host có manager `copier`<br>• Ký hợp đồng pentest auth-server (ASVS 5.0 L3)<br>• Khởi động pilot tier 1<br>• ADR-0015 … ADR-0020 | Platform service chạy ở dev và staging qua GitOps; restore PITR thử thành công; một PR `copier update` tự động được merge | Chọn nơi chạy (ADR-0015); ngân sách cụm; người thứ 2 |
| **Q2/2027** | Pilot tier 1 lên production | • Kết quả pentest đã xử lý → `platform-auth-server` **1.0.0**<br>• `platform-java`/`platform-web` **1.0.0** cùng ma trận tương thích<br>• Kargo promote dev → staging → prod<br>• SLO và dashboard cho auth/iam/gateway; runbook<br>• Diễn tập DR lần 1<br>• Bộ công cụ tuân thủ dữ liệu cá nhân: consent, export/delete tài khoản (ADR-0025)<br>• Preview environment cho frontend | Pilot tier 1 chạy production ≥ 4 tuần không có sự cố P1 do platform; lần đầu đo DORA | Pentest không còn lỗi critical/high (nếu còn, pilot chạy Keycloak như ADR-0008); khách hàng pilot |
| **Q3/2027** | Pilot tier 2, mobile, đo lường | • Pilot tier 2: nhiều backend qua gateway, IAM `remote`, Kafka (Strimzi, KRaft)<br>• Gateway rate limit qua Redis; lọc danh sách theo policy<br>• `template-expo` (ADR-0027) và mở rộng auth-server cho native client<br>• Dashboard DORA, khảo sát SPACE lần 1, bảng fleet<br>• Kyverno chuyển sang Enforce; SLSA Build L2 có chữ ký trên mọi image<br>• Site tài liệu platform<br>• Nâng toàn bộ lên Boot 4.2 (`platform-java` 2.x) và Spring Cloud 2026.0 | Pilot tier 2 lên production; một app mobile nội bộ đăng nhập qua auth-server; ≥ 80% repo dự án dùng dòng `platform-*` còn được hỗ trợ | Kết quả Q2; nhân sự mobile |
| **Q4/2027** | Củng cố và chấm lại mức trưởng thành | • Chấm lại bảng 1.2; mục tiêu: mức 2 ở mọi chiều, mức 3 ở Interfaces<br>• Pentest định kỳ năm đầu<br>• Quyết định Backstage (ADR-0029)<br>• Quyết định ISO 27001 theo nhu cầu khách hàng<br>• FinOps: showback theo dự án<br>• Lộ trình 2028 | Báo cáo trưởng thành có số liệu; mỗi service nền tảng có ≥ 2 owner | Có ít nhất 3–4 dự án dùng platform |

Việc lặp lại mỗi quý:
- Rà soát bảng chuẩn công nghệ: Boot, Next, Node, K8s, Argo CD, Keycloak dự phòng.
- Nâng K8s theo nhịp 2 lần/năm.
- Chấm lại F1–F20 trong kiểm kê.

## 5. ADR nên viết tiếp

ADR-0011 (tenant và danh bạ) và ADR-0012 (IdP mở rộng) đang được viết ở nhánh khác. Đề xuất các ADR tiếp theo:

| ADR | Tiêu đề | Mô tả một dòng |
|---|---|---|
| 0013 | Version, release và deprecation của `platform-*` | Semver sau 1.0, dòng release theo minor của Boot, hỗ trợ 2 dòng, ma trận tương thích, quy tắc deprecate, CHANGELOG tự sinh |
| 0014 | Nâng cấp hàng loạt dự án con | Renovate với nhóm "platform", manager `copier` (self-host) hoặc workflow `copier-update`, OpenRewrite/codemod, bảng fleet |
| 0015 | Runtime đích | K8s managed trước, K3s cho on-prem; chính sách nâng 2 minor mỗi năm; phân tách cụm theo môi trường hoặc khách hàng |
| 0016 | GitOps và môi trường | Argo CD 3.x, repo `<du-an>-infra`, dev/staging/prod cùng digest, Kargo để promote |
| 0017 | Preview environment theo PR | ApplicationSet PR generator, nhãn `preview`, database tạm, TTL và quota |
| 0018 | PostgreSQL trên K8s | CloudNativePG, Barman Cloud Plugin, PITR, RPO/RTO theo tier, diễn tập restore mỗi quý |
| 0019 | Migration không downtime | Expand/contract với Flyway, quy ước dải số migration cho nhánh song song, kiểm tra trong CI |
| 0020 | Secrets | ESO `v1` (≥ 1.2.0) + secret manager của cloud hoặc OpenBao; xoay vòng; không có secret trong Git (B18) |
| 0021 | Chuỗi cung ứng | Mục tiêu SLSA v1.2 Build L2→L3 và Source L2; cosign v3 dùng khoá KMS cho repo private; Kyverno `ImageValidatingPolicy` Audit→Enforce |
| 0022 | Observability | OTel Collector + LGTM; quy ước metric, log, trace; dashboard và alert đi kèm chart |
| 0023 | SLO, hỗ trợ và on-call cho service nền tảng | SLI/SLO cho auth, iam, gateway; error budget policy; kênh hỗ trợ và mức phản hồi |
| 0024 | DR/BCP | RPO/RTO, sao lưu secret gốc, mirror registry, diễn tập 6 tháng một lần |
| 0025 | Bảo vệ dữ liệu cá nhân | Luật 91/2025/QH15 + NĐ 356/2025/NĐ-CP: phân định trách nhiệm platform/khách hàng, consent, export/delete, masking, retention |
| 0026 | Review bảo mật và pentest | ASVS 5.0 L3 cho auth-server, L2 cho phần còn lại; tần suất; SLA sửa lỗi; dùng GitHub Security Advisory |
| 0027 | Golden path mobile | Expo SDK ≥ 55, OIDC native app (PKCE, public client), lưu token trên thiết bị, gọi qua gateway |
| 0028 | Đo lường platform | DORA (5 chỉ số), khảo sát SPACE mỗi quý, chỉ số adoption, chấm CNCF maturity 6 tháng một lần |
| 0029 | Tài liệu và dev portal | Site tài liệu tĩnh trước; tiêu chí để chuyển sang Backstage |
| 0030 | FinOps | OpenCost, nhãn chi phí bắt buộc trong chart, showback theo dự án, FOCUS |
| 0031 | Kafka trên K8s | Strimzi, KRaft, chỉ dùng cho tier ≥ 2, quy ước topic/DLT |
| 0032 | Rate limit phân tán ở gateway | Bucket trong Redis thay cho bucket theo từng instance (F20) |
| 0033 | Lọc danh sách theo policy | Dịch điều kiện policy thành điều kiện truy vấn (phần "các ADR sau" của ADR-0010) |
| 0034 | Ranh giới repo và lưu trữ repo rỗng | Xử lý `nextjs-ui`, `spring-boot-starter` (archive hoặc xoá); tiêu chí khi nào tách hoặc gộp repo |
| 0035 | Owner và mức hỗ trợ của sản phẩm nền tảng | Mỗi service có ≥ 2 owner; mức hỗ trợ cam kết với dự án; quy trình nhận đóng góp ngược |

## 6. Rủi ro lớn và cách giảm

| # | Rủi ro | Dấu hiệu hiện tại | Cách giảm |
|---|---|---|---|
| R1 | **Bus factor = 1** | Org có 1 thành viên; mọi repo do một người commit; ADR-0008 đòi ≥ 2 owner nhưng chưa đạt | Tuyển hoặc điều động thêm người thứ 2 và thứ 3 trước Q2/2027 (điều kiện để pilot lên production). Runbook và ADR viết đủ để người khác tiếp quản. Pair review bắt buộc với auth-server. Ưu tiên dịch vụ managed để giảm việc vận hành |
| R2 | **Auth-server tự viết** | Chưa pentest (`platform-auth-server/README.md:8`); phạm vi đang mở rộng (passkey, SAML, LDAP, Zalo) làm tăng bề mặt tấn công | Giữ Keycloak là phương án B (ADR-0008); app chỉ phụ thuộc OIDC chuẩn nên đổi được. Pentest ASVS L3 trước 1.0. Bật các tính năng IdP mở rộng theo cờ, mặc định tắt. Theo dõi advisory của Spring Security/SAS để vá trong 7 ngày. Nếu không đủ 2 owner trước 07/2027 thì dự án khách hàng dùng Keycloak |
| R3 | **Quá nhiều repo cho đội nhỏ** | 12 repo, 2 repo rỗng; template lệch version so với thư viện; 4 service chưa tag | Không thêm repo mới khi chưa có người dùng. Archive repo rỗng (ADR-0034). Nhóm Renovate "platform" và bảng fleet. Giữ `platform-java` và `platform-web` là monorepo của từng ngôn ngữ. Cân nhắc gộp `platform-iam-server` vào cùng repo/release train với starter, vì repo này chỉ chứa cấu hình và đóng gói (`platform-iam-server/README.md:6`) |
| R4 | **Xây platform trước khi có khách hàng** | Chưa có dự án nào dùng (F19) | Pilot tier 1 phải bắt đầu trong Q1/2027. Tính năng mới chỉ làm khi pilot yêu cầu; danh sách "khi có dự án cần" trong ADR giữ nguyên tinh thần đó |
| R5 | **Tốc độ code vượt tốc độ review** | `platform-java` 8 minor trong 6 ngày; có commit do công cụ AI sinh. DORA 2025: AI tăng throughput nhưng giảm ổn định | Lên 1.0 rồi thì giữ semver nghiêm. Đo rework rate và change failure rate. Bắt buộc test tái hiện cho mỗi lỗi (như brief tab 1). Review bảo mật cho thay đổi ở auth và authz |
| R6 | **Phụ thuộc GitHub** | Packages, GHCR, Actions; attestation không dùng được cho repo private nếu không có gói Enterprise Cloud; Maven package cần PAT classic | Ký bằng cosign thay cho GitHub attestation. Mirror registry cho on-prem. Định kỳ export repo. Đánh giá chi phí gói Enterprise Cloud nếu khách hàng yêu cầu provenance |
| R7 | **Nhịp nâng cấp của hệ sinh thái** | Boot 6 tháng/minor; K8s hỗ trợ khoảng 14 tháng; Keycloak không có LTS; Node chuyển sang mỗi năm một bản từ v27 | Gom vào lịch rà soát hằng quý. Ma trận tương thích. Không hỗ trợ dài hơn upstream |
| R8 | **Pháp lý dữ liệu cá nhân** | Luật 91/2025 và NĐ 356/2025 có hiệu lực từ 01/01/2026; tài liệu nội bộ còn nhắc NĐ 13/2023 | ADR-0025. Đưa consent và export/delete vào golden path. Xin tư vấn pháp lý khi làm dự án tài chính hoặc y tế |
| R9 | **Độ phức tạp vận hành K8s** | Cùng lúc thêm Argo CD, Kargo, CNPG, ESO, Kyverno, LGTM, Strimzi | Đưa vào theo quý, mỗi thứ có lý do. Dự án tier 0 có thể deploy đơn giản (một VM + compose) như ngoại lệ có ADR. Dùng bản managed khi có thể |

## 7. Sơ đồ quy hoạch (danh sách cạnh)

Quy ước: `A -> B` nghĩa là A dùng, phụ thuộc hoặc deploy B. `[mới]` là thành phần đề xuất, chưa có. `[đang làm]` là thành
phần đang làm ở nhánh khác của đợt này.

```
# Tầng 1 — Golden path (developer dùng trực tiếp)
template-spring-boot -> platform-java
template-spring-boot -> platform-auth-server            # thay Keycloak [kế hoạch Q4/2026]
template-spring-boot -> .github:java-ci
template-spring-boot -> .github:container
template-nextjs -> platform-web
template-nextjs -> platform-auth-server
template-nextjs -> .github:node-ci
template-nextjs -> .github:container
template-expo [mới] -> platform-web:@vyforge/i18n
template-expo [mới] -> platform-web:@vyforge/api-client(types)
template-expo [mới] -> platform-auth-server              # OIDC native app
template-expo [mới] -> platform-gateway
template-infra [mới] -> platform-helm-charts:app
template-infra [mới] -> argo-cd [mới]
template-infra [mới] -> kargo [mới]
docs-site [mới] -> .github:docs
docs-site [mới] -> backstage [mới, quyết định Q4/2027]

# Tầng 2 — Thư viện và artifact có version
platform-java -> spring-boot@4.1 (2.x -> 4.2)
platform-web -> next@16 / react@19 / node@24|26
platform-helm-charts:app -> kubernetes@1.35|1.36
platform-gateway -> spring-cloud@2025.1 (-> 2026.0)
.github:renovate-config -> renovate-copier-manager [mới]
.github:copier-update [mới] -> template-spring-boot
.github:copier-update [mới] -> template-nextjs
.github:copier-update [mới] -> template-expo [mới]
.github:copier-update [mới] -> template-infra [mới]
fleet-report [mới] -> .copier-answers.yml (mọi repo dự án)

# Tầng 3 — Danh tính và quyền
platform-auth-server -> postgresql(auth)
platform-auth-server -> tenant-directory [đang làm, ADR-0011]
platform-auth-server -> idp-extensions [đang làm, ADR-0012]
platform-auth-server -> consent-export-delete [mới, ADR-0025]
platform-auth-admin -> platform-auth-server
platform-auth-admin -> platform-iam-server               # khu Phân quyền [đang làm]
platform-iam-server -> platform-java:platform-starter-authz
platform-iam-server -> postgresql(iam)
platform-iam-server -> redis
platform-gateway -> platform-auth-server                 # JWKS
platform-gateway -> platform-iam-server                  # route /api/v1/iam/**
platform-gateway -> redis                                # rate limit phân tán [mới, ADR-0032]
app(remote) -> platform-iam-server
app -> platform-auth-server                              # token exchange
keycloak (dự phòng) -.-> platform-auth-server            # thay thế được qua OIDC

# Tầng 4 — Runtime và GitOps
argo-cd [mới] -> <du-an>-infra [mới]
argo-cd:applicationset-pr [mới] -> preview-env [mới]
kargo [mới] -> argo-cd [mới]
.github:gitops-bump -> <du-an>-infra [mới]
.github:container -> ghcr
<du-an>-infra [mới] -> platform-helm-charts:app
kubernetes -> kyverno [mới]
kubernetes -> external-secrets [mới]
kubernetes -> cloudnative-pg [mới]
kubernetes -> strimzi-kafka [mới, tier ≥ 2]
kubernetes -> otel-collector [mới]
kubernetes -> opencost [mới]

# Tầng 5 — Dữ liệu
cloudnative-pg [mới] -> postgresql@18
cloudnative-pg [mới] -> barman-cloud-plugin [mới]
barman-cloud-plugin [mới] -> object-storage(S3)
app -> flyway (expand/contract, ADR-0019)
platform-java:platform-starter-messaging -> kafka@4 (KRaft)
platform-java:platform-starter-messaging -> redis

# Tầng 6 — Observability và vận hành
platform-java:platform-starter-observability -> otel-collector [mới]
otel-collector [mới] -> loki [mới]
otel-collector [mới] -> tempo [mới]
otel-collector [mới] -> mimir [mới]
grafana [mới] -> loki / tempo / mimir
slo-dashboards [mới] -> grafana [mới]
slo-dashboards [mới] -> platform-auth-server / platform-iam-server / platform-gateway
dora-metrics [mới] -> github-api
dora-metrics [mới] -> argo-cd / kargo

# Tầng 7 — Bảo mật chuỗi cung ứng và secrets
.github:container -> trivy
.github:container -> sbom + provenance (BuildKit)
.github:container -> cosign@3 [mới]
cosign@3 [mới] -> kms|openbao [mới]
kyverno:ImageValidatingPolicy [mới] -> cosign@3 [mới]
external-secrets@>=1.2 [mới] -> cloud-secret-manager|openbao [mới]
platform-helm-charts:app -> external-secrets

# Tầng 8 — Quản trị, tuân thủ, con người
adr-0013-versioning [mới] -> platform-java / platform-web / services
adr-0023-slo-oncall [mới] -> platform-auth-server / platform-iam-server / platform-gateway
adr-0025-pdp [mới] -> platform-auth-server / template-spring-boot / template-nextjs
adr-0026-pentest [mới] -> platform-auth-server
adr-0035-owners [mới] -> CODEOWNERS (mọi repo platform)
module-thong-dung (báo cáo tab 5) -> platform-java / platform-web   # notification, storage, job, flag…
```

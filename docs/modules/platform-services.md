# platform-services — server dùng chung nhúng các starter

## 1. Mục đích

Một ứng dụng Spring Boot dựng sẵn gom các module dùng chung (notification, files, payment, workflow, forms, ai…) vào một
tiến trình, để nhiều app nghiệp vụ cùng dùng thay vì mỗi app nhúng lại từng starter. Người vận hành chọn module chạy
bằng `SERVICES_MODULES`, giao với giấy phép; mỗi tenant chỉ thấy module mà gói dịch vụ cho phép. Admin microfrontend
(`/m/<module>`) của platform-admin gọi API của server này. Thiết kế: `docs/design/platform-services.md` (đã duyệt
27/10/2026).

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| App Java | `com.vyforge.platform:platform-services` (module trong repo platform-java) | platform-java 0.16.1; `Dockerfile` build từ gốc repo, `compose.yaml` (postgres, valkey, kafka) cho dev |
| Starter nhúng | notification, files, webhooks, features, payment, realtime, ai, api-access, calendar, cms-content, collaboration, ekyc, forms, master-data, organization, privacy (privacy-dsr), search, sequence, settings, tenant-lifecycle, workflow | luôn bật: ops, licensing; nền: web, security, jpa, authz, audit, scheduling, messaging, ratelimit, privacy, observability |
| **Không** có trong server | export, custom-fields, webhooks-inbound (chỉ nhúng vào app — D10); automation, billing, einvoice, esign (chưa có trong `pom.xml` và danh mục — cần xác nhận kế hoạch) | |
| Frontend | platform-admin (shell + zone `/m/<module>`) | xem `platform-admin.md` |

## 3. Tính năng chính

- **Bật/tắt module** (`ModuleToggles`): `vyforge.services.modules` (`SERVICES_MODULES`) — danh sách tên, `*` là mọi
  module, rỗng là không module nào; tên lạ làm server dừng lúc khởi động. Kết quả giao với giấy phép: trạng thái
  `ENABLED` / `DISABLED` / `UNLICENSED` (module không có giấy phép không chạy, module khác vẫn chạy). `<prefix>.enabled=false`
  tắt riêng một module.
- **Entitlement theo tenant** (`ModuleEntitlementInterceptor`): khi features chạy trong server, tenant không có feature
  `module.<key>` nhận `404 module.not_enabled`; feature `admin.<key>` điều khiển menu quản trị. Mặc định bật cho mọi
  module đang chạy; gói dịch vụ hoặc override của tenant tắt bớt. Không áp cho đường công khai (IPN, callback),
  `/admin/info`, `/internal/**`. Không đọc được entitlement thì `503`, không bao giờ mở quyền.
- **Danh mục action IAM** (`ModuleCatalogRegistrars`): ở `vyforge.authz.mode=remote`, đăng ký danh mục action từng
  module lên platform-iam-server (`PUT /api/v1/iam/internal/catalogs/<module>`, cần scope `iam.internal.admin`).
- **API cho admin shell**: `GET /api/v1/services/admin/modules` (module chạy/tắt/thiếu giấy phép);
  `GET /api/v1/<route>/admin/info` (hợp đồng chung cho module chưa có endpoint riêng).
- **OpenAPI theo module**: nhóm `/v3/api-docs/<module>` + nhóm `platform`.
- **Hạ tầng dùng chung**: một database cho mọi module của deployment (bảng có tiền tố riêng, lịch sử migration riêng
  `flyway_schema_history_<module>`; bảng outbox, `processed_messages`, `audit_log`, `idempotency_keys` chung), Valkey
  (kênh `vyforge:iam:changed`, `vyforge:features:changed`, stream realtime), Kafka tuỳ chọn.
- **Health**: `servicesModules` trong nhóm readiness (cùng `db`); Valkey/Kafka lỗi thì module tự xuống cấp.
- Mỗi module báo `mode/deployment = services` qua `/admin/info` để admin MFE biết module chạy ở server dùng chung.

## 4. Cách dùng nhanh

```bash
# build image từ gốc repo platform-java
docker build -f platform-services/Dockerfile -t platform-services .
```

```bash
# biến môi trường tối thiểu
DATABASE_URL=jdbc:postgresql://db:5432/services
REDIS_URL=redis://valkey:6379
OIDC_ISSUER_URI=https://auth.example.vn        # token phải có aud platform-services
OIDC_CLIENT_SECRET=...                         # client credentials của server (gọi iam-server)
VYFORGE_CRYPTO_ACTIVE_KEY=k1 VYFORGE_CRYPTO_KEYS_K1=<base64 32 byte>
SERVICES_MODULES=notification,files,webhooks,features,payment,workflow
VYFORGE_AUTHZ_REMOTE_URL=http://platform-iam-server:8085
```

App nghiệp vụ dùng module ở server: đặt `vyforge.<module>.mode=remote` + `vyforge.<module>.remote.url=http://platform-services`
(với module có client remote: forms, cms-content, ai, ekyc, collaboration một phần) hoặc gọi REST `/api/v1/<module>/**`
qua gateway; nhận event qua Kafka khi `KAFKA_ENABLED=true`.

## 5. Cấu hình quan trọng

| Khoá / biến | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.services.modules` / `SERVICES_MODULES` | `notification,files,webhooks,features` | Module chạy trong deployment |
| `vyforge.services.tenants` / `SERVICES_TENANTS` | `default` | Tenant cho job định kỳ theo tenant |
| `vyforge.tenant.required` / `SERVICES_TENANT_REQUIRED` | `true` | |
| `vyforge.authz.mode` / `VYFORGE_AUTHZ_MODE` | `remote` | `local` khi bản triển khai không có iam-server (bảng `iam_*` trong DB này) |
| `vyforge.authz.remote.url` | `http://localhost:8085` | platform-iam-server |
| `vyforge.authz.admin-roles` / `SERVICES_ADMIN_ROLES` | `IAM_ADMIN` | |
| `server.port` / `management.server.port` | 8087 / 8088 | |
| `KAFKA_ENABLED` | `false` | Bật listener và externalization event (bắt buộc khi app remote cần event) |
| `OIDC_AUDIENCE` | `platform-services` | Audience bắt buộc của token |
| `FILES_STORAGE_TYPE` / `FILES_STORAGE_ROOT` / `FILES_SIGNING_KEY` | `local` / `./data/files` / — | Production dùng S3-compatible |
| `TENANT_LIFECYCLE_SCOPE` | `auth.tenant.provision` | Scope tạo/khoá tenant ở auth-server |
| `vyforge.crypto.active-key` + `VYFORGE_CRYPTO_KEYS_<KID>` | `k1` | Khoá AES-256 cho secret của mọi module — mất khoá là mất mọi secret |

## 6. Tích hợp với module khác

- **licensing**: giấy phép quyết định module nào được chạy (`UNLICENSED`); module chưa có descriptor coi là có phí.
- **features**: entitlement `module.<key>`/`admin.<key>` theo tenant.
- **platform-iam-server**: authz remote, đăng ký danh mục action theo module.
- **platform-auth-server**: phát token (người dùng qua BFF/admin MFE, app qua client credentials), danh bạ tenant
  (tenant-lifecycle).
- **platform-gateway**: định tuyến `/api/v1/<module>/**`, chặn `/internal/**`.
- **platform-admin**: shell gọi `/api/v1/services/admin/modules`, zone gọi `/api/v1/<module>/admin/**`.

## 7. Chế độ triển khai, IAM, sự kiện

- Ba cách đóng gói một module (thiết kế): nhúng starter vào app; gộp trong `platform-services`; hoặc chạy thành server
  riêng (cùng image, `SERVICES_MODULES` khác, database riêng như `services_payment`).
- Kubernetes: chart chung `charts/app` của platform-helm-charts (chưa có chart riêng `platform-services` — cần xác
  nhận); probe `/actuator/health/liveness|readiness`.
- IAM: không có action riêng (API của server chỉ cần đăng nhập); action của từng module như trong tài liệu module.
- Sự kiện: event của từng module ra Kafka khi `KAFKA_ENABLED=true` (vd `payment-events`).

## 8. Trạng thái & còn mở

- Đã có: chọn module + giấy phép, entitlement theo tenant, đăng ký danh mục IAM, OpenAPI theo module, admin API, image
  Docker, compose dev.
- Chưa có trong server: automation, billing, einvoice, esign (mỗi module này cũng chưa có chế độ remote).
- Chế độ remote phía app chưa đồng đều: workflow, automation, payment, billing chỉ embedded/services, chưa có client
  Java remote; collaboration remote chỉ có `ActivityRecorder`.
- Không có README trong thư mục `platform-services`; tài liệu này dựa trên code (`ServicesModules`, `ModuleToggles`,
  `application.yml`) và thiết kế.

# ADR-0027: Giấy phép module cho bản cài on-prem

- **Trạng thái:** Chấp nhận — 28/10/2026. Thiết kế chi tiết: [licensing.md](../design/licensing.md) (D1–D12).
  Liên quan [ADR-0003](0003-modular-monolith.md), [ADR-0014](0014-neutral-ci-cd.md),
  [ADR-0020](0020-shared-application-modules.md).
- **Phát hành:** `platform-core` (`core.license`, `core.signing`), `platform-starter-licensing`, `tools/license-cli`
  (platform-java 0.13.0); gắn điều kiện vào starter trả phí (L6), `LicensingNotice`, cảnh báo
  `DEPLOYMENT_URL_MISMATCH` trên nhánh `feat/licensing-followup`, chờ merge; `@vyforge/licensing` 0.1.0, admin MFE
  `/m/licensing`; lớp giấy phép của `@vyforge/admin-shell` 0.3.0 và PrometheusRule `licensingAlerts` của
  `platform-observability` chờ merge.

## Bối cảnh

Bản cài on-prem và air-gapped (cơ quan nhà nước, ngân hàng) cần giới hạn module, số tenant, số người dùng và dòng phát
hành theo hợp đồng, kiểm hoàn toàn offline, không tắt dịch vụ đang chạy khi hết hạn. Việc này chạm mọi starter trả phí,
`platform-services`, shell `platform-admin` và hạ tầng, nên ghi thành ADR riêng thay vì một mục của ADR-0020.

## Quyết định

1. **Lớp mỏng tự xây, file JSON ký Ed25519 trên JCS (RFC 8785)**, khoá hai tầng: root nhúng trong `platform-core` và
   `@vyforge/licensing`, khoá phát hành theo năm trong OpenBao Transit, chứng chỉ do root ký đi kèm từng file. Không
   phone-home, không license server; đối soát bằng báo cáo sử dụng khách tự gửi.
2. **Kiểm ở `platform-core`, trước DataSource:** `LicenseBootstrap` đọc file (env `VYFORGE_LICENSE_TEXT`, file, classpath)
   một lần cho mỗi `Environment`; ba chế độ `LICENSED`/`DEV`/`UNLICENSED`. Thiếu `deploymentId` hoặc không khớp →
   `INVALID`; URL của bản triển khai (`vyforge.license.deployment-url`, env `VYFORGE_DEPLOYMENT_URL`) khác URL trong giấy
   phép chỉ là cảnh báo `DEPLOYMENT_URL_MISMATCH` (proxy, đổi tên miền không được làm hỏng giấy phép).
3. **Starter trả phí gắn `@ConditionalOnLicensedModule("<key>")`** lên auto-config chính và khai báo
   `META-INF/vyforge/modules/<key>.json` (`licensed: true`, `since`, `apiPrefix` khi khác `/api/v1/<key>`). Không có
   giấy phép thì bean, migration, job, REST của module không được tạo; `FailureAnalyzer` báo rõ khi app bắt buộc bean đó.
   - Trả phí: webhooks, payment, files, notification, features (chế độ nhúng), export, realtime, einvoice, billing (gồm
     cầu nối), ai, forms, organization, sequence, master-data, collaboration, ekyc, calendar, search (cả adapter
     OpenSearch/Meilisearch), api-access, workflow, custom-fields, cms-content, tenant-lifecycle, automation, esign
     (cả `-dss`), privacy (`platform-starter-privacy-dsr`). Khoá module là tên starter bỏ tiền tố, trừ privacy.
   - Miễn phí (`licensed: false`): licensing, settings, ops-console, webhooks-inbound, admin-kit và phần nền (core, web,
     jpa, security, audit, scheduling…).
   - Không gắn: client chế độ remote (`*RemoteAutoConfiguration`), cầu nối tích hợp của organization, participant của
     tenant-lifecycle: chúng gọi sang service đã được cấp phép hoặc là điểm nối của module khác.
4. **Test chạy chế độ dev mặc định:** `platform-test` cài `LauncherSessionListener` đặt `vyforge.license.mode=dev` khi
   chưa có system property hay env, nên test của starter trả phí không cần giấy phép. Test kiểm `UNLICENSED` đặt
   `enforce` tường minh (như `platform-starter-licensing`). Hạn mức dev (5 tenant, 50 người dùng) áp cả trong test.
5. **`platform-services`:** `ModuleToggles` giao cấu hình bật module với giấy phép (module không có descriptor coi là trả
   phí); service nhúng `platform-starter-licensing`.
6. **Hết hạn không tắt dịch vụ:** hành vi sau hạn là chính sách ghi trong giấy phép (`banner`, `block-upgrade`,
   `block-new-tenants`, `block-create-over-limit`, `restrict-admin-writes`); chặn nâng cấp theo ngày phát hành dòng minor
   so với `support_until`; gia hạn đọc lại file mỗi 60 giây không cần khởi động lại.
7. **Thông báo licensing ra ngoài:** audit của starter (`licensing:license:*`) đồng thời phát `LicensingNotice`
   (schemaVersion, type, deploymentId, serial, details, at) trong transaction; `vyforge.licensing.events.externalize=true`
   thì Spring Modulith đẩy ra topic `vyforge.licensing`, khoá theo `deploymentId`. Event trong tiến trình
   (`LicenseChanged`…) của core giữ nguyên.
8. **Shell `platform-admin` tự kiểm file** bằng `@vyforge/licensing/server`; `@vyforge/admin-shell` không phụ thuộc
   package đó mà nhận callback: manifest khai `license` (mặc định = `name`, `false` = miễn phí), registry lọc lớp 0 ở mọi
   chế độ trước quyền và entitlement, trạng thái `unlicensed` ở trang Module, `license` trong phản hồi registry chỉ cho
   người vận hành, `AdminFrame` có slot `banner` cho `LicenseBanner`.
9. **Vận hành:** metric `vyforge_license_*` ở core (mọi tiến trình); PrometheusRule `licensingAlerts` (sắp hết hạn, ân
   hạn, hết hạn, giấy phép không dùng được ngoài chế độ dev, chạy chế độ dev quá lâu, hạn mức ≥ 90 %, file mới bị từ chối,
   đồng hồ lùi).

## Phương án đã loại

| Phương án | Lý do loại |
|---|---|
| Keygen, license3j làm phụ thuộc runtime | FCL-1.0 (không OSI) và cần server; license3j định dạng nhị phân riêng |
| License server / phone-home | Bản air-gapped không có Internet; tạo điểm chết chung |
| Tắt dịch vụ khi hết hạn | Dịch vụ công, ngân hàng không được dừng vì thủ tục gia hạn chậm |
| Kiểm giấy phép trong `ModuleToggles` của `platform-services` thay cho điều kiện trên starter | App tự dựng (không qua platform-services) sẽ chạy module trả phí mà không cần giấy phép |
| Gắn điều kiện cả client remote | Client chỉ gọi service đã được cấp phép; chặn nó làm app nhỏ phải mang giấy phép |
| Coi webhooks-inbound là trả phí | Là thư viện nhận webhook mà payment, einvoice, esign, ekyc dùng; trả phí theo module dùng nó |
| URL không khớp → `INVALID` | Proxy, đổi tên miền, truy cập qua IP nội bộ làm hỏng giấy phép đang dùng; deploymentId đã đủ ràng buộc |
| `@vyforge/admin-shell` phụ thuộc `@vyforge/licensing` | Shell dùng được ở app không bán theo giấy phép; callback đủ cho lớp 0 |
| Profile `test` bắt buộc cho mọi test của starter | Phải sửa hàng trăm test và `ApplicationContextRunner`; listener của platform-test áp một lần |

## Hệ quả

- **Cho tới buổi tạo khoá root (Q7), bản build không tin giấy phép nào ngoài test:** bản triển khai không ở chế độ dev
  sẽ tắt toàn bộ module trả phí. Môi trường dev/test phải đặt `vyforge.license.mode=dev` (hoặc profile `dev`/`local`/
  `test`); bản cài khách hàng chỉ phát hành sau khi đã nhúng root.
- App dùng module trả phí ở chế độ `UNLICENSED` mà bắt buộc bean của module (constructor injection) sẽ không khởi động,
  với thông báo của `LicenseFailureAnalyzer`; dùng `ObjectProvider` nếu module là tuỳ chọn.
- Module mới phải có descriptor; thiếu descriptor thì `ModuleToggles` coi là trả phí.

## Lệch so với thiết kế

- Danh sách miễn phí thêm webhooks-inbound và ops-console (Q5 chỉ liệt kê licensing, ops-console, settings, admin-kit).
- `restrict-admin-writes` là servlet filter của starter (order -90), không phải interceptor của `-web`.
- `@vyforge/admin-shell` nhận callback `isLicensed` thay vì tự gọi `loadLicense` + `licenseAllows`.
- Hạn mức `tenants`/`users` (L7) chưa đo: tenant và người dùng ở `platform-auth-server`, không ở `platform-iam-server`
  như thiết kế ghi.

## Còn mở

- Buổi tạo khoá root (Shamir 3/5), thêm vào `roots.json` (Java) và `trust.ts` (TS), khoá phát hành trong OpenBao Transit
  `transit-license`.
- L7: `LicenseUsageProvider` (`tenants`, `users`, `active_users`) và `requireWithinLimit` khi tạo tenant/người dùng ở
  `platform-auth-server`, sau khi repo đó lên platform-parent có `core.license`.
- Shell `platform-admin` truyền `license` vào `resolveRegistryView` và `LicenseBanner` vào `AdminFrame`; licensing-admin
  khai `license: false` (sau khi phát hành `@vyforge/admin-shell` 0.3.0).
- L11 repo `license-ledger` (PR bốn mắt, CI ký, `ledger.jsonl` có chuỗi băm); Secret `vyforge-license` và ConfigMap
  `vyforge-deployment` trong chart/overlay; hướng dẫn cấp và cài giấy phép.

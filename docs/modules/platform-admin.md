# platform-admin — khu quản trị (shell + admin microfrontend)

## 1. Mục đích

Một khu quản trị duy nhất cho người vận hành platform và tenant admin, ghép từ nhiều microfrontend (Next.js
Multi-Zones): shell ở `/` lo tài khoản, tenant, IdP, phân quyền, đăng nhập và registry; mỗi module có một zone riêng
`/m/<module>` build và deploy độc lập, gắn vào shell lúc chạy. Mọi zone dùng chung phiên đăng nhập, khung giao diện, CSP
có nonce. Thiết kế: `docs/design/platform-services.md` §4 (đã duyệt 27/10/2026).

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Repo | `platform-admin` (pnpm workspace) | |
| Shell | `apps/shell` (`@vyforge/platform-admin-shell` 0.1.0), image `platform-admin` | BFF Next.js 16 của platform-auth-server + platform-iam-server |
| Zone module | `apps/<module>-admin`, image `platform-admin-<module>` | hiện có: ai, api-access, automation, billing, calendar, cms-content, collaboration, custom-fields, ekyc, esign, forms, licensing, master-data, ops-console, organization, privacy, search, sequence, settings, tenant-lifecycle, trash, workflow |
| Zone mẫu | `apps/example-admin` (`/m/example`) | sinh từ template, dùng trong E2E, không phát hành |
| Template | `templates/admin-zone` + `scripts/create-admin-zone.mjs` | |
| Package dùng | `@vyforge/admin-shell`, `@vyforge/admin-kit`, `@vyforge/ui`, `@vyforge/api-client`, `@vyforge/auth`, `@vyforge/iam`, `@vyforge/i18n`, `@vyforge/<module>/admin` | |

Chưa có zone cho: notification, files, webhooks, features, payment, realtime, export, einvoice (cần xác nhận — tên trong
README là ví dụ).

## 3. Tính năng chính

**Shell** (chỉ role `admin` và đăng nhập có MFA — claim `amr` chứa `mfa`; khu `/iam/**` mở thêm cho `IAM_ADMIN`):

| Trang | Nội dung |
|---|---|
| `/` | Tổng quan tài khoản, MFA, đăng nhập 24 giờ, biểu đồ 14 ngày |
| `/users`, `/roles`, `/clients` | Tài khoản (mời, khoá, MFA, phiên, ứng dụng đã cấp quyền), role, client OAuth2 (xoay secret) |
| `/sign-ins`, `/activity` | Lịch sử đăng nhập, nhật ký quản trị (`@Audited`) |
| `/settings` | Thương hiệu (tên sản phẩm, màu, email hỗ trợ) |
| `/tenants` | Chỉ role `platform-admin` (ADR-0011): tạo/khoá/xoá tenant, thành viên, gắn client |
| `/iam` | Quản trị platform-iam-server: policy (JSON có kiểm), group, gán quyền |
| `/modules` | Trạng thái registry các zone (ok, stale, lỗi, thiếu giấy phép) |

**Cơ chế microfrontend**:

- Ingress chuyển `/m/<module>/**` tới zone; dev/không ingress thì shell chuyển tiếp theo `ADMIN_MODULES`.
- Shell đọc `ADMIN_MODULES`, tải `GET /m/<module>/api/admin-manifest` của từng zone (kiểm schema + `contract`), dựng menu;
  zone tạm lỗi thì giữ manifest cũ (`stale`). Ẩn module theo giấy phép và theo entitlement `admin.<module>` (khi có
  `FEATURES_API_BASE_URL`).
- Phiên chung: Better Auth cùng `BETTER_AUTH_URL` (URL shell), `BETTER_AUTH_SECRET`, OIDC client; zone không mount
  `/api/auth/*`. Đăng xuất ở shell làm zone mất phiên. Chuyển module là tải lại trang.
- Hai chế độ: `operator` (tenant vận hành + role vận hành + MFA) và `tenant`; menu/trang có `audience`.
- Mỗi trang và Server Action của zone gọi `requireModuleAccess(manifest, path)` / `requireOperator` /
  `requireStepUp` trước (B13) — chỉ chặn sớm, backend vẫn kiểm từng thao tác.

## 4. Cách dùng nhanh — chạy local và tạo zone mới

```bash
cp apps/shell/.env.example apps/shell/.env.local     # BETTER_AUTH_SECRET=$(openssl rand -base64 32)
pnpm install && pnpm dev                             # shell: http://localhost:3001
```

**Tạo khu quản trị cho một module:**

```bash
pnpm create-admin-zone invoice --title-vi "Hoá đơn" --title-en "Invoices"   # [--port 3110]
# → apps/invoice-admin từ templates/admin-zone (manifest, trang mẫu, kiểm quyền, next.config, proxy, i18n)
```

1. Sửa `apps/<module>-admin/admin.manifest.ts`: `minPermission` (action IAM thật), `entitlement`, `api.base` +
   `api.requires` (khoảng version backend), `menu` (có `permission`, `audience`).

   ```ts
   export default defineAdminManifest({
     contract: 1, name: 'invoice', version: pkg.version,
     title: { vi: 'Hoá đơn', en: 'Invoices' }, icon: 'blocks',
     audiences: ['operator', 'tenant'],
     minPermission: { anyOf: [{ action: 'invoice:settings:Get' }] },
     entitlement: 'admin.invoice',
     api: { base: '/api/v1/invoice', requires: '>=0.13.0 <1.0.0' },
     menu: [{ id: 'overview', path: '/', title: { vi: 'Tổng quan', en: 'Overview' } }],
   });
   ```
2. Viết trang trong `src/app/[locale]/`, mỗi trang gọi `requireModuleAccess(manifest, '/...')` rồi ghép component từ
   `@vyforge/<module>/admin` (+ `@vyforge/admin-kit`).
3. Chạy: `pnpm --filter @vyforge/<module>-admin dev`, thêm `ADMIN_MODULES=[{"name":"<module>","url":"http://localhost:<port>"}]`
   vào `.env.local` của shell, mở `http://localhost:3001/m/<module>`.
4. Image: `docker build --build-arg APP=<module>-admin -t platform-admin-<module> .`; CI thêm bước
   `vyforge-ci image --name platform-admin-<module> --build-arg APP=<module>-admin`.
5. Sửa template thì tạo lại `apps/example-admin` (test `scripts/create-admin-zone.test.mjs` kiểm hai bên khớp).

## 5. Cấu hình quan trọng

| Biến | Ở | Ý nghĩa |
|---|---|---|
| `BETTER_AUTH_URL` / `BETTER_AUTH_SECRET` | shell (zone dùng `ADMIN_SHELL_URL` + cùng secret) | URL công khai của shell; khoá cookie ≥ 32 ký tự |
| `OIDC_ISSUER`, `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`, `OIDC_ROLES_CLAIM` | shell + zone | Client confidential `platform-auth-admin` (audience thêm `platform-iam-server`) |
| `API_BASE_URL` | shell: auth-server; zone: gateway | API quản trị |
| `IAM_API_BASE_URL` | shell + zone | platform-iam-server (`/api/v1/iam/me/evaluate`) |
| `ADMIN_MODULES` | shell | `[{"name":"files","url":"http://platform-admin-files:3000"}]` |
| `ADMIN_OPERATOR_TENANT` | shell | Tenant người vận hành, mặc định `default` |
| `FEATURES_API_BASE_URL` | shell (tuỳ chọn) | Ẩn module tenant không được hưởng |
| `ADMIN_SHELL_INTERNAL_URL` | zone (tuỳ chọn) | Lấy menu `/api/admin/registry` |
| `<MODULE>_API_BASE` | zone | Vd `WORKFLOW_API_BASE` (mặc định `/api/v1/workflow`) |

## 6. Tích hợp

- platform-auth-server: đăng nhập OIDC + PKCE, API `/api/v1/admin/**`, Directory API chọn người dùng.
- platform-iam-server: khu Phân quyền, đánh giá quyền menu/trang.
- platform-services / app nhúng starter: API `/api/v1/<module>/**` qua gateway, `GET /api/v1/services/admin/modules`,
  `/admin/info` của từng module.
- licensing: manifest `license` (mặc định theo `name`, `false` = zone miễn phí như licensing, settings).

## 7. Triển khai

- CI qua `vyforge-ci`: `node-verify` → `image` (Trivy, SBOM, provenance, cosign) → `gitops-bump` vào `platform-infra`
  (Argo CD). Không trỏ cứng về GitHub (ADR-0014).
- E2E nhiều zone: Traefik (Docker) cổng 3180, OIDC/IAM giả (`e2e/oidc-mock.mjs`).

## 8. Trạng thái & còn mở

- Đã có: shell đầy đủ, registry + giấy phép + entitlement, 22 zone module, template + script tạo zone, E2E nhiều zone.
- README repo còn ghi `vendor/vyforge-admin-shell-0.1.0.tgz`, nhưng thư mục `vendor/` đã không còn trên `main` (README cũ —
  nên sửa). Shell đang dùng `@vyforge/admin-shell` 0.2.0, package mới nhất 0.3.0.
- Một số zone tạo trước khi package `@vyforge/<module>` lên registry (vd workflow) — lockfile cập nhật sau khi publish
  (cần xác nhận).
- Chưa có zone cho các module P1 đầu tiên (notification, files, webhooks, features, payment, realtime, export) và
  einvoice (cần xác nhận kế hoạch).

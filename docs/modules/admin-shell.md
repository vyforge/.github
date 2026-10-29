# @vyforge/admin-shell — khung chung của shell và các admin microfrontend

## 1. Mục đích

Gói chung để shell `platform-admin` và mọi zone `/m/<module>` hoạt động như một khu quản trị: hợp đồng manifest giữa zone
và shell, registry module (giấy phép, entitlement), phiên đăng nhập dùng chung, kiểm quyền đầu trang, cấu hình Next.js
cho zone, CSP có nonce, khung giao diện (menu, trạng thái module). Zone chỉ cần khai manifest và ghép màn hình của module.
Thiết kế: `docs/design/platform-services.md` §4–5.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Package web | `@vyforge/admin-shell` 0.3.0 (repo platform-web) | không có README package; tài liệu này dựa trên mã nguồn |
| Subpath | `.` (manifest, registry, mode, CSP), `/config`, `/proxy`, `/server`, `/ui`, `/manifest.schema.json` | |
| Dùng ở | `platform-admin/apps/shell`, mọi `apps/<module>-admin`, `templates/admin-zone` | shell hiện pin 0.2.0 |

## 3. Tính năng chính

- **Manifest** (`defineAdminManifest`, `parseAdminManifest`, `adminManifestSchema` zod + JSON Schema): `contract`
  (hiện `1`; `SUPPORTED_CONTRACTS`), `name` (`/^[a-z][a-z0-9-]{1,30}$/` = path `/m/<name>`), `version` (semver),
  `title` vi/en, `icon`, `order`, `audiences` (`operator`/`tenant`), `minPermission.anyOf` (action
  `<service>:<resource>:<Verb>`), `entitlement`, `license` (khoá module hoặc `false` = miễn phí), `api.base` +
  `api.requires` (khoảng semver backend), `menu` (cây 2 tầng, id không trùng, `permission`, `audience`), `health`.
- **Registry** (`createModuleRegistry` ở shell): tải `GET <url>/m/<name>/api/admin-manifest` của mọi zone trong
  `ADMIN_MODULES` (`parseAdminModules`), TTL 60 giây, timeout 2 giây, giữ manifest hợp lệ cuối khi zone lỗi (`stale`);
  `applyLicense`, `resolveRegistryView` (lọc theo giấy phép và feature `admin.<module>`), `createRegistryClient` (zone
  lấy menu từ shell), `createEntitlementsClient`.
- **Chế độ** (`adminModeOf`, `tenantOf`, `rolesOf`, `hasMfa`, `tokenClaims`): `operator` khi token thuộc tenant vận hành
  + role vận hành + MFA; còn lại `tenant`. Chỉ đọc claim để chọn giao diện, backend vẫn kiểm.
- **Phiên** (`createAdminAuth` ở `/server`): Better Auth cùng `baseURL` (URL shell), cùng secret, cùng OIDC client — zone
  đọc cookie của shell, không mount `/api/auth/*`; `getSession`, `requireUser`, `getAccessToken`, `hasRole`, `loginUrl`.
- **Kiểm quyền** (`createAdminAccess`): `requireModuleAccess(manifest, path)` (chế độ trong `audiences` + thoả
  `minPermission`), `requireOperator`, `requireMfa` (thao tác tiền/secret), `can` gom theo request
  (`/iam/me/evaluate`); thiếu quyền mặc định `notFound()`.
- **Cấu hình zone** (`createAdminZone` ở `/config`): `basePath: /m/<module>`, output standalone, header bảo mật,
  `deploymentId`, `serverActions.allowedOrigins`.
- **Proxy** (`createAdminProxy` ở `/proxy`): CSP nonce theo request + định tuyến ngôn ngữ next-intl; shell chuyển tiếp
  `/m/<name>/**` khi không có ingress; hook `frameSources` cho zone cần iframe (vd live preview cms). Không phân quyền ở
  đây (B13).
- **CSP** (`contentSecurityPolicy`, `createNonce`, `frameSourceOrigins`, `SECURITY_HEADERS`): không `unsafe-eval` ở
  production.
- **UI** (`/ui`): `AdminFrame` (khung, menu, banner giấy phép), `ModuleNav`, `ModuleStatusTable`, `ZoneLink`,
  `adminShellMessages`.

## 4. Cách dùng nhanh

```ts
// next.config.ts của zone
import { createAdminZone } from '@vyforge/admin-shell/config';
export default withNextIntl(createAdminZone({ module: 'files' }));
```

```ts
// src/shared/auth/session.ts của zone
export const { getSession, requireUser, getAccessToken, hasRole } = createAdminAuth({
  config: () => ({ shellUrl: env().ADMIN_SHELL_URL, secret: env().BETTER_AUTH_SECRET /* , OIDC … */ }),
  basePath: '/m/files',
});
```

```tsx
// trang
const access = await requireModuleAccess(manifest, '/');
```

Zone mới nên tạo bằng `pnpm create-admin-zone <module>` trong platform-admin (đã nối sẵn mọi thứ trên) — xem
`platform-admin.md`.

## 5. Cấu hình quan trọng

| Biến / tuỳ chọn | Mặc định | Ý nghĩa |
|---|---|---|
| `ADMIN_MODULES` (shell) | rỗng = không module | JSON `[{name,url}]`; sai định dạng thì ném lỗi liệt kê dòng sai |
| `AdminModeOptions.operatorTenant` / `operatorRole` | `default` / `admin` | Xác định chế độ operator |
| `rolesClaim` / `tenantClaim` | `roles` / `tenant_id` | |
| `ModuleRegistryOptions.ttlMs` / timeout | 60s / 2s | |
| `ADMIN_SHELL_URL`, `BETTER_AUTH_SECRET` (zone) | — | Phiên dùng chung |

## 6. Tích hợp

- `@vyforge/licensing/server` (`loadLicense`) cho lọc giấy phép và banner; `@vyforge/features` cho entitlement.
- `@vyforge/iam` (`createPermissionHelpers`) cho `can`.
- Backend: `GET /api/v1/services/admin/modules`, `/admin/info` của từng module.

## 7. Triển khai

Không có backend riêng; mỗi zone là một app Next.js standalone (image `platform-admin-<module>`), ingress định tuyến theo
path.

## 8. Trạng thái & còn mở

- Hợp đồng `contract: 1` ổn định; đổi không tương thích thì thêm version mới, giữ bản cũ ít nhất một minor.
- Chưa có README trong package (nên bổ sung). Shell và template đang pin 0.2.0 trong khi package 0.3.0 (cần xác nhận
  khác biệt).

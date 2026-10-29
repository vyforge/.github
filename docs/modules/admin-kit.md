# @vyforge/admin-kit — khối dựng màn hình quản trị

## 1. Mục đích

Bộ khối dựng màn hình quản trị dùng chung cho mọi admin microfrontend (`<module>-admin`) và trang `/admin` của app nghiệp
vụ: bảng danh sách lọc/sắp/phân trang phía server, form có lỗi theo field, trang chi tiết, thao tác có kiểm quyền, xác
nhận thao tác nguy hiểm, thao tác hàng loạt, lịch sử thay đổi, theo dõi việc nền, che/hiện dữ liệu cá nhân theo Luật
91/2025. Không phải framework admin: dữ liệu đi một chiều **URL → Server Component → API → props**, token chỉ ở server,
Server Action kiểm quyền ở đầu (B13). Thiết kế: `docs/design/admin-kit.md`.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Package web | `@vyforge/admin-kit` 0.2.0 (platform-web) | Storybook cổng 6007 |
| Subpath | `.`, `/query`, `/table`, `/form`, `/detail`, `/widgets`, `/server`, `/messages`, `/testing` | |
| Dùng ở | các zone platform-admin, trang `/admin` của app sinh từ template-nextjs | |
| Backend | hợp đồng danh sách `com.vyforge.platform.web.list` (platform-java ≥ 0.13) | |

## 3. Tính năng chính

- **Định nghĩa resource** (`defineAdminResource`): `key`, `service`, `pagination` (`offset`/`cursor`), bộ lọc
  (`filter.multiSelect`, `dateRange`, `text`… có `audience`, `sensitive`), sắp xếp cho phép, tìm kiếm, cột (`col.id`,
  `enum` có tone, `text` có `href`, `number`, `datetime`, `boolean`, `requires` capability), thao tác (`action.server`
  theo dòng có `rowPermission`/`when`, `action.bulk` theo id hoặc theo bộ lọc có xác nhận gõ số lượng + lý do,
  `action.export`). `extendResource`, `registerCell`, `registerFilter` để mở rộng.
- **`/query`**: `defineListQuery`, `createListLoader`, `toApiQuery`, `toExportFilter`, đồng bộ URL (nuqs).
- **`/table`**: `DataTable`, `FilterBar`, `ColumnPicker`, `BulkBar`, `RowActions`, `SavedViewMenu`, skeleton.
- **`/form`**: `useResourceForm` + `ResourceForm` (React Hook Form + Standard Schema), `SchemaResourceForm` (bọc
  `SchemaForm` của `@vyforge/forms`), `applyProblem` (lỗi Problem Details về đúng field theo JSON Pointer), `409
  *.version_conflict` → banner "Tải lại", `SecretField`, `ReferenceSelect`, `MoneyInput`.
- **`/detail`**: `DetailView`, `DetailSection`, `FieldValue`, `DetailActions`.
- **`/widgets`**: `ConfirmDangerDialog`, `AuditTimeline`, `TaskMonitor.{Backlog,List,Progress}`, `StatusBadge`,
  `CopyableId`.
- **`/server`**: `ResourceListPage`, `ResourceDetailPage`, `ResourceFormPage`, `PermissionGate`, `resolveResource`,
  `loadAllowed`, `createResourceAction`, `createRowAction`, `createBulkAction`, `createTaskActions`,
  `createAuditLoader`, `createModuleAdminClient` (`{base}/admin/tasks/**`, `{base}/admin/audit`), view đã lưu, bộ lọc
  nhạy cảm, `createRevealAction`.
- **Dữ liệu cá nhân**: cột `personalData` ẩn mặc định và che ngay ở server (`091****678`); `'sensitive'` không gửi giá
  trị xuống; nút "Hiện" (`reveal: {}`, action `<service>:<resource>:RevealPersonalData`, backend ghi audit mỗi lần hiện);
  bộ lọc `sensitive` không lên URL, giữ trong cache BFF 15 phút (`?sq=<id>`).
- **Quyền**: cột/lọc/thao tác có `audience`/`permission`/`requires` được lọc ở server; quyền theo dòng từ
  `allowedActions` của API, không có thì hỏi `can()` theo lô (≤ 200).
- **`/testing`**: `offsetPage`, `cursorPage`, `taskInfo`, `auditEntry`, `moduleAdminHandlers` (MSW), `renderWithUrl`.

## 4. Cách dùng nhanh

```bash
pnpm add @vyforge/admin-kit
```

```ts
// features/deliveries/resource.ts
export const deliveries = defineAdminResource<Delivery>({
  key: 'webhooks.delivery',
  service: 'webhooks',
  pagination: 'cursor',
  query: {
    filters: { status: filter.multiSelect({ options: ['PENDING', 'FAILED'], label: 'status' }) },
    sort: { allowed: ['createdAt'], default: [['createdAt', 'desc']] },
  },
  columns: [col.id('id'), col.enum('status'), col.datetime('createdAt', { sortable: true })],
  actions: { row: [action.server('retry', { rowPermission: 'webhooks:delivery:Retry' })] },
});
```

```tsx
// app/[locale]/deliveries/page.tsx
const access = await requireModuleAccess(manifest, '/deliveries');
return (
  <ResourceListPage resource={deliveries} access={access} info={await adminInfo()} searchParams={searchParams}
    load={(_q, api) => unwrap(webhooks().GET('/api/v1/webhooks/admin/deliveries', { params: { query: api } }))}
    actions={{ retry: retryDeliveries }} title={t('title')} t={t} locale={locale} linkAs={Link} />
);
```

App bọc `NuqsAdapter` ở layout (zone dùng `AdminFrame` của `@vyforge/admin-shell` ≥ 0.2 đã có sẵn).

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `ADMIN_KIT_SENSITIVE_QUERY_TTL` | 15 phút | Thời gian giữ bộ lọc nhạy cảm trong cache BFF |
| Hợp đồng danh sách | `page` từ 0, `size` ≤ 200, `sort=field,dir`, `<f>From`/`<f>To`, `cursor`, `include=allowedActions` | Backend `com.vyforge.platform.web.list` |
| Hợp đồng hàng loạt | `POST …/bulk/{op}` `{ids}` hoặc `{filter, expectedCount}` → `200 BulkResult` / `202 {taskId}` | `Idempotency-Key` sinh khi mở hộp xác nhận |
| Hợp đồng hiện PII | `POST {base}/admin/<plural>/{id}/reveal` `{field}` → `{field, value}` | |

## 6. Tích hợp

- `@vyforge/ui` (component nền), `@vyforge/forms` (`SchemaForm`), `@vyforge/export` (nút xuất), `@vyforge/admin-shell`
  (`requireModuleAccess`, `AdminFrame`), `@vyforge/api-client` (Problem Details, `ActionResult`).
- Backend: `TaskAdminEndpoint` (`/admin/tasks/**`) và `ModuleAuditEndpoint` (`/admin/audit`) của platform-java.

## 7. Triển khai

Thư viện, không có runtime riêng. Phát hành qua registry npm `@vyforge` khi merge platform-web (nâng version cùng branch).

## 8. Trạng thái & còn mở

- Đã có: đủ subpath, hiện PII có audit, view đã lưu, bộ lọc nhạy cảm, Storybook + test render mọi story.
- Ca hợp đồng URL ↔ API dùng chung ở `src/query/contract-cases.json` (cùng ca với Java).

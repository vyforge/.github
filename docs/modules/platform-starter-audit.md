# platform-starter-audit

## 1. Mục đích

Ghi nhật ký nghiệp vụ theo một chuẩn chung: ai, làm gì, trên bản ghi nào, kết quả, request id, IP. Bản ghi thành công đi
cùng transaction nghiệp vụ. Bản ghi thất bại ghi bằng transaction riêng, nên vẫn còn dù nghiệp vụ rollback. Dữ liệu cá
nhân trong `details` được che trước khi ghi. Đây là nguồn truy vết cho tenant admin, người vận hành và yêu cầu pháp lý.

## 2. Thành phần

| Loại | Tên |
|---|---|
| Java | `com.vyforge.platform:platform-starter-audit` (kéo `platform-starter-privacy`; `-web`, `-authz` tuỳ chọn) |
| Package web | `@vyforge/iam` 0.5.0 (`createAuditLogClient()`, `AuditLogTable`); `AuditTimeline` của `@vyforge/admin-kit` cho khu từng module |
| Zone admin | Trang `/iam/audit` của shell; khu admin của từng module (`{base}/admin/audit`) |
| App liên quan | mọi backend, `platform-starter-ops` (bắt buộc) |

## 3. Tính năng chính

- **Ghi log**:
  - `@Audited(action, resource, details)` dùng SpEL, có sẵn `#result`.
  - `AuditLog.success(...)` và `failure(...)` ghi trong code.
  - `forResource(tenant, vrn, n)` đọc lịch sử của một bản ghi.
- **Người thực hiện**: `sub` của token; job hay consumer ghi `system`. Tenant lấy từ `TenantContext`, chỉ khi không có
  mới đọc claim. Thay được bằng bean `AuditContextProvider`.
- **Che PII**: `details` ghi qua `LogSafeJson`. Field `@PersonalData` bị che hoặc hash; email, số điện thoại, CCCD nằm
  trong chuỗi cũng bị che.
- **Tìm kiếm**: `search(tenant, AuditQuery(actor, action, resource, outcome, from, to), offset, limit)` và `count`.
  `action`/`resource` kết thúc bằng `*` thì lọc theo tiền tố.
- **REST** (khi có authz): `GET /api/v1/audit-log?actor=&action=&resource=&outcome=&from=&to=&page=&size=`, quyền
  `audit:entry:List`, chỉ trong tenant của người gọi.
- **`ModuleAuditEndpoint.forService("x")`**: dựng sẵn `{base}/admin/audit` cho khu admin của module. Endpoint này chỉ
  trả resource `vrn:x:*`.
- **Audit theo field**: `AuditChanges.diff(before, after)` (định dạng `vyforge.audit.changes/1`). Field `@PersonalData`
  hay secret được ghi là `masked`.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-audit</artifactId>
</dependency>
```

Tạo bảng theo `META-INF/vyforge/sql/audit_log.sql` (trong migration của app). Chỉ cấp quyền INSERT/SELECT trên bảng cho
user của app.

```java
@Audited(action = "crm:customer:Create", resource = "'vrn:crm:customer/' + #result.id()", details = "{'email': #email}")
public Customer create(String email) { ... }

auditLog.success("crm:order:Approve", Resources.of("crm", "order", id), Map.of("note", note));
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.audit.api-enabled` | `true` | Bật API `/api/v1/audit-log` (cần `platform-starter-authz`) |
| `vyforge.audit.tenant-claim` / `.default-tenant` | `tenant_id` / `default` | Chỉ dùng khi không có `TenantContext` |
| `vyforge.privacy.json.mask-text` | `true` | Che PII trong mọi chuỗi của `details` |
| `vyforge.privacy.hash-key` | ngẫu nhiên mỗi lần chạy | Khoá HMAC cho `Redaction.HASH`. Phải đặt để hash so được giữa các replica |

## 6. Tích hợp với module khác

- `platform-starter-privacy` cung cấp `LogSafeJson` và bộ che PII.
- `platform-starter-authz` cung cấp kiểm quyền cho REST.
- `platform-starter-web` (bulk) ghi một dòng audit `<service>:<resource>:Bulk<Op>` cho cả lô.
- `platform-starter-ops` ghi mọi thao tác vận hành (kèm lý do) vào đây.
- Phần lớn starter nghiệp vụ có `-audit` là phụ thuộc bắt buộc hoặc tuỳ chọn.

## 7. Chế độ triển khai, quyền, sự kiện

- Thư viện nhúng. Bảng `audit_log` nằm trong database của app (`platform-services` có sẵn bảng này).
- Quyền: `audit:entry:List`.
- Không phát sự kiện.

## 8. Trạng thái & còn mở

- Ổn định.
- Không có retention hay partition dựng sẵn cho `audit_log`; bảng tăng dần theo thời gian (cần xác nhận kế hoạch
  partition).
- Vẫn chỉ nên đưa vào `details` những gì cần để truy vết.

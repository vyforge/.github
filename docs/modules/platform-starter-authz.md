# platform-starter-authz

## 1. Mục đích

Phân quyền chi tiết theo mô hình AWS IAM (ADR-0009). Token chỉ mang `sub`, role thô và `tenant_id`. Quyền chi tiết
(action trên resource, có điều kiện theo thuộc tính) nằm trong policy JSON do admin tự cấu hình qua API, không cần
deploy. Cùng một starter chạy theo hai chế độ:

- `local`: policy nằm trong database của app.
- `remote`: nạp policy từ `platform-iam-server` khi nhiều service dùng chung người dùng (ADR-0010).

## 2. Thành phần

| Loại | Tên |
|---|---|
| Java | `com.vyforge.platform:platform-starter-authz` (kéo `-web`, `-security`) |
| Package web | `@vyforge/iam` 0.5.0: client, `PolicyEditor`, `PolicySimulator`, `GroupEditor`, `AttachmentManager`, `AuditLogTable`, quản lý thành viên tenant |
| Zone admin | Trang `/iam/*` nằm trong `apps/shell` của platform-admin (policies, groups, users, simulator, actions, audit), không có zone riêng |
| App liên quan | `platform-iam-server` (chế độ `remote`), mọi backend |

## 3. Tính năng chính

- **Policy JSON**:
  - `Allow`/`Deny`, wildcard `*` và `?`, `NotAction`, `NotResource`.
  - `Condition`: `String*`, `Numeric*`, `Date*`, `Bool`, `IpAddress`, `Null`; hậu tố `IfExists`; tiền tố
    `ForAnyValue:`/`ForAllValues:`.
  - Biến `${principal.*}`; biến nhiều giá trị mở thành OR.
  - Deny thắng mọi Allow.
- **Gắn policy**: vào group (thành viên trực tiếp, hoặc qua `tokenRole` = role trong token), vào user, hoặc policy
  riêng (inline) của user.
- **Kiểm quyền trong code**:
  - `@RequiresPermission(action, resource SpEL)`
  - `@PreAuthorize("hasPermission(...)")`
  - bean `Authz`: `check`, `allowed`, `holds`
  - `Resources.of(EntityRef)`
  - Thiếu quyền → 403 Problem Details.
- **Danh mục action**: `PermissionAction` + `ActionCatalog`, dạng `<service>:<resource>:<Verb>`.
- **Thuộc tính cho `Condition`**: `ResourceAttributesResolver` (`resource.*`) và `PrincipalAttributesProvider`
  (`principal.*`).
- **Permission boundary** (từ 0.15):
  - `PermissionBoundaryProvider` trả các `PermissionBoundary` (tập policy) theo claim của token.
  - Quyền hiệu lực = policy ∧ mọi boundary. Deny ở đâu cũng thắng. Boundary không cấp thêm quyền.
  - Áp cho `check`, annotation, `hasPermission`, việc nền và Policy Simulator.
  - api-access cài sẵn `ScopeBoundaryProvider` theo scope của token, key và installation.
- **Chống leo thang đặc quyền**:
  - Không tự sửa quyền của chính mình (`iam.self_modification`).
  - Chỉ cấp được action mình đang có (`iam.grant_not_held`).
  - Gắn hay gỡ policy cần quyền trên cả policy lẫn đích.
- **API quản trị `/api/v1/iam`**:
  - `actions`
  - `policies` (`PUT` cần `version`), `policies/validate`
  - `groups`, members, gắn policy
  - `users/{sub}`, `inline-policies`
  - `simulate` (thử được policy, claim và boundary chưa lưu)
  - `audit-log`
  - `me`, `me/evaluate` (frontend hỏi tối đa 100 cặp action/resource cho `can()`)
- **Cache** theo `(tenant, sub, roles)`, TTL 60 s:
  - Thay đổi qua API xoá cache sau khi commit.
  - Có Redis thì mọi replica nghe kênh `vyforge:iam:changed`.
  - Health `iamInvalidation`.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-authz</artifactId>
</dependency>
```

```yaml
# chế độ local: bảng iam_* qua Flyway location của starter
spring.flyway.locations: classpath:db/migration,classpath:META-INF/vyforge/db/iam
```

```java
public enum OrderActions implements PermissionAction {
    APPROVE("crm:order:Approve", "Duyệt đơn hàng");
    // constructor + action() + description()
}
@Bean ActionCatalog orderActions() { return ActionCatalog.of(OrderActions.values()); }

@RequiresPermission(action = "crm:order:Approve", resource = "'vrn:crm:order/' + #id")
public void approve(UUID id) { ... }
```

Chế độ remote: `vyforge.authz.mode=remote`, `remote.url`, và client registration `vyforge-iam` (client credentials, scope
`iam.internal`). Test: `TestAuthz.builder().role("user", "<policy JSON>").boundaries(...)` của `platform-test`.

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.authz.mode` | `local` | `remote`: nạp policy từ `platform-iam-server` |
| `vyforge.authz.remote.url` / `.service` / `.stale-ttl` | — / `spring.application.name` / `10m` | IAM server lỗi thì dùng bản cache cũ tối đa `stale-ttl`; quá hạn thì 503 `iam.unavailable` |
| `vyforge.authz.remote.register-catalog` | `true` | Đăng ký `ActionCatalog` lên IAM server lúc khởi động |
| `vyforge.authz.admin-roles` | `IAM_ADMIN` | Role được toàn quyền `iam:*` để khởi tạo |
| `vyforge.authz.cache-ttl` | `60s` | `0s` tắt cache |
| `vyforge.authz.api-enabled` | `true` | Bật API `/api/v1/iam/**` |
| `vyforge.authz.invalidation.redis.enabled` | `true` | Xoá cache mọi replica qua Redis |
| `vyforge.authz.internal-api.enabled` | `false` | Chỉ `platform-iam-server` bật |

## 6. Tích hợp với module khác

- Hầu hết starter nghiệp vụ khai `ActionCatalog` và gọi `Authz`. `platform-starter-web` dùng `Authz` cho
  `AllowedActions`; `platform-starter-audit` dùng nó cho `GET /api/v1/audit-log`.
- `platform-starter-api-access` cài `ScopeBoundaryProvider`. Bật/tắt bằng
  `vyforge.api-access.authz.permission-boundary`.
- `platform-starter-realtime` kiểm quyền subscribe topic qua `Authz` và kiểm lại khi IAM đổi.
- `platform-gateway` không đánh giá IAM. Frontend gọi `/api/v1/<service>/iam/me/evaluate` của service sở hữu action.

## 7. Chế độ triển khai, quyền, sự kiện

- **local**: bảng `iam_*` nằm trong database của app.
- **remote**: không cần database. Service đánh giá trong app với policy nạp từ IAM server. API quản trị chỉ có trên IAM
  server; `/me/evaluate` vẫn chạy trong service.
- Quyền: `iam:*` (policy, group, user, simulate…), `iam:audit:List`.
- Sự kiện: kênh Redis `vyforge:iam:changed` (chỉ mang tenant). Metric `vyforge.authz.decisions{outcome}`,
  `vyforge.authz.cache{result}`, `vyforge.authz.remote.requests`, `vyforge.authz.boundary.denied`.

## 8. Trạng thái & còn mở

- Ổn định. Permission boundary có từ 0.15. Ở chế độ remote, boundary của provider cục bộ áp như ở chế độ local.
- `request.ip` cần `server.forward-headers-strategy` và chỉ tin proxy của mình.
- `Resources.Ref` đã `@Deprecated` từ 0.13 (thay bằng `EntityRef`); bản 1.0 sẽ bỏ.
- Chế độ local mà thiếu JDBC thì app dừng lúc khởi động.

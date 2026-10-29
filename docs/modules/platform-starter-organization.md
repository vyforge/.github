# platform-starter-organization — cơ cấu tổ chức dùng chung

## 1. Mục đích

Hầu hết ứng dụng nghiệp vụ cần biết "người này thuộc đơn vị nào, quản lý trực tiếp là ai, chi nhánh nào" để định tuyến
duyệt, đánh số theo chi nhánh, phân quyền dữ liệu theo nhánh. Organization cung cấp cây đơn vị **có hiệu lực theo thời
gian**, chức danh và phân công, thay đổi theo **đợt** có ngày hiệu lực và số quyết định (xem trước tác động, duyệt, huỷ,
hoàn tác, sáp nhập/chia tách), tra cứu theo thời điểm và thuộc tính IAM để viết policy theo cơ cấu (ADR-0022).

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Starter | `platform-starter-organization` | Từ 0.14.0; có phí (`licensed: true`) |
| Package web | `@vyforge/organization` 0.1.0 | Client API, chọn đơn vị, sơ đồ tổ chức, component quản trị |
| Zone admin | `apps/organization-admin` 0.1.0 (`/m/organization`) | |
| Service | `platform-services` | Bật module `organization`; app khác dùng `remote` |
| Phụ thuộc ngoài | `platform-auth-server` | Danh bạ (`/api/v1/directory/users:lookup`) và event gỡ thành viên |

Thiết kế: [docs/design/organization.md](../design/organization.md), ADR-0022.

## 3. Tính năng chính

- **Cây đơn vị** (công ty, chi nhánh, phòng ban, nhóm…) có hiệu lực theo thời gian, nhiều cây (`main`, `region`…), cờ
  `BRANCH_LIKE`, `LEGAL_ENTITY`; mẫu khởi tạo `enterprise`, `public-sector`, `education`, `healthcare`, `retail-chain`.
- **Chức danh, phân công**: chính, kiêm nhiệm, phụ trách, biệt phái; trưởng/phó; quản lý chéo (`reports_to`); thành viên
  không có tài khoản (`PERSON:<id>`).
- **Tra cứu** (`OrgDirectory`) theo thời điểm `at`: `managerOf`, `headOf`, `holders`, `unitsOf`, `primaryUnitOf`,
  `descendants`, `ancestorWithFlag`, `ancestorOfType`; `resolveManager` trả kèm giải thích từng bước. Thời điểm hiện tại
  đọc từ snapshot bất biến trong bộ nhớ, tự hết hạn ở mốc hiệu lực kế tiếp.
- **Lọc dữ liệu theo nhánh**: `OrgScopes.subtreeCondition("o.unit_id", rootId, at)` (SQL, cùng DB) hoặc `subtreeIds`.
- **Đợt thay đổi** (`OrgAdmin`): `draft` → `validate` (vi phạm + tác động, không ghi) → `submit` (→ `SCHEDULED`) →
  có hiệu lực; `cancel` (khôi phục), `revert` (đợt ngược, `409` nếu đợt sau đã chạm cùng đối tượng). Thao tác:
  `CreateUnit`, `SplitUnit`, sáp nhập, `Transfer`… "Sửa nhanh" là đợt một thao tác. Các đợt áp tuần tự (advisory lock),
  ràng buộc exclusion của PostgreSQL là lớp chặn cuối.
- **Thuộc tính IAM**: `principal.orgUnits`, `principal.branch`, `principal.managedUnits`, `resource.unitPath`; biến đa trị
  mở thành OR (authz 0.13).
- REST `/api/v1/organization`: `/me`, `/units/**` (children, ancestors, descendants, members, history, successors),
  `/chart`, `/people/{type}/{id}/assignments|managers`, `/jobs`, `/unit-types`, `/search`; quản trị `/admin/**`
  (change-sets với `:validate|:submit|:approve|:reject|:return|:cancel|:revert`, units, assignments, presets, settings,
  reconcile, `tree:verify`); nội bộ `/internal/snapshot|revision`.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-organization</artifactId>
</dependency>
```

```bash
pnpm add @vyforge/organization
```

Cần PostgreSQL có extension `btree_gist` (hoặc `vyforge.organization.integrity=application`, chọn trước lần migrate đầu).

```yaml
vyforge.organization.directory.url: http://platform-auth-server:9000
spring.security.oauth2.client.registration.organization-directory:
  provider: vyforge
  client-id: platform-services
  client-secret: ${PLATFORM_SERVICES_SECRET}
  authorization-grant-type: client_credentials
  scope: directory.read
```

```java
String approverFor(String userId, long amount) {
    Instant at = Instant.now();
    if (amount > 50_000_000) {
        String branch = org.primaryUnitOf(userId, at)
                .flatMap(u -> org.ancestorWithFlag(u.id(), UnitFlag.BRANCH_LIKE, at)).orElseThrow().id();
        return org.holders("GDCN", branch, at).getFirst();
    }
    return org.managerOf(userId, 1, at).orElseThrow();
}
```

Policy theo nhánh:

```json
{"Statement": {"Effect": "Allow", "Action": "crm:order:Get", "Resource": "vrn:crm:order/*",
  "Condition": {"ForAnyValue:StringEquals": {"resource.orgUnitPath": "${principal.branch}"}}}}
```

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.organization.mode` | `embedded` | `remote`: snapshot từ `platform-services` (client `vyforge-organization`, scope `organization.internal`) |
| `vyforge.organization.integrity` | `database` | `application`: không cần `btree_gist` (migration `organization-application V1`) |
| `vyforge.organization.defaults.preset` | `enterprise` | Mẫu áp cho tenant lần đầu dùng |
| `vyforge.organization.admin-roles` | `ADMIN`, `TENANT_ADMIN` | Role được ghi khi không có `-authz` |
| `vyforge.organization.directory.url`, `.registration-id` | —, `organization-directory` | Danh bạ auth-server |
| `vyforge.organization.auth-events.enabled` | `false` | Nhận event gỡ thành viên từ auth-server (cần webhooks-inbound) |
| `vyforge.organization.retention.anonymizeEndedAssignmentsAfter` | `P24M` | Ẩn danh hoá phân công đã kết thúc |
| `vyforge.organization.retention.pseudonym-key` | — | Khoá HMAC cho `erased:<hmac>` |
| `vyforge.organization.schema.migrate` | `true` | `false` khi DBA tự chạy `META-INF/vyforge/sql/organization.sql` |

## 6. Tích hợp với module khác

Phụ thuộc tuỳ chọn: settings, export, features, search, forms-core, workflow, webhooks-inbound.

- **authz** (khuyến nghị): kiểm quyền `organization:*`, thuộc tính `principal.*`; không có thì đọc chỉ cần đăng nhập, ghi cần
  role trong `admin-roles`.
- **auth-server**: `AuthServerMemberDirectory` (mặc định khi không có bean `MemberDirectory`); event
  `auth.tenant.member_removed` / `auth.account.disabled` qua **webhooks-inbound** đánh dấu phân công mồ côi ngay.
- **workflow**: bean `OrgChangeApproval` để duyệt đợt thay đổi bằng quy trình; workflow dùng `OrgDirectory` định tuyến.
- **settings**: tầng `ORG_UNIT` theo đơn vị chính của người dùng. **sequence**: phạm vi `branch`/`legalEntity`.
  **calendar**: lịch theo đơn vị (`OrgUnitStep`). **search**: token `o:<unitId>` (`OrgSearchTokens`).
- **export**: xuất/nhập đơn vị, phân công, chức danh, đợt thay đổi (`OrgExports`, `OrgImports`). **forms**: nguồn lựa
  chọn đơn vị (`OrgChoiceSources`). **features**: giới hạn theo entitlement (`OrgEntitlements`). **privacy-dsr**:
  `OrganizationPersonalData`.
- SPI: `MemberDirectory`, `PersonDirectory`, `OrgChangeApproval`, `ResourceAttributesResolver` (phía app).

## 7. Chế độ triển khai, quyền, sự kiện

- **embedded** hoặc **remote**: app remote có `OrgDirectory` và thuộc tính IAM trên snapshot tải về, `OrgAdmin` remote
  (0.16.0); `subtreeCondition` trả 501 vì không cùng DB.
- **Quyền IAM**: `organization:unit:List|Get|Create|Update|Move|Merge|Close`,
  `organization:assignment:List|Get|Create|Update|End`, `organization:changeset:List|Create|Submit|Approve|Apply|Cancel|Revert|Backdate`,
  `organization:job:List|Manage`, `organization:type:Manage`, `organization:settings:Get|Update`,
  `organization:reconcile:Manage`.
- **Sự kiện** (`OrganizationEvent`, chỉ mang id): `OrgUnitCreated`, `OrgUnitUpdated`, `OrgUnitMoved`, `OrgUnitClosed`,
  `OrgUnitReopened`, `OrgUnitSplit`, `OrgUnitsMerged`, `UnitHeadChanged`, `MemberAssigned`, `MemberTransferred`,
  `MemberAssignmentEnded`, `MemberOrphaned`, `OrgChangeSetScheduled`, `OrgChangeSetEffective`.

## 8. Trạng thái & còn mở

- **Đã làm**: 0.14.0 bản đầu; 0.16.0 thêm `integrity=application`, `OrgAdmin` remote, tìm người, benchmark JMH; nối danh
  bạ và event của auth-server.
- **Chờ xác nhận** (thiết kế 10.3): Q3 pháp chế xác nhận thời hạn ẩn danh hoá 24 tháng (Luật 91/2025 Điều 25 khoản 2);
  Q4/Q5 cam kết của phiên workflow và search.
- **Giới hạn đã biết**: chỉ lưu `member_id`, tên/email lấy từ danh bạ lúc hiển thị; danh bạ lỗi thì không đối soát.
  `integrity` phải chọn trước lần migrate đầu tiên.

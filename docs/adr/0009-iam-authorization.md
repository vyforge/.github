# ADR-0009: Phân quyền nghiệp vụ theo mô hình IAM (policy JSON) trong app

- **Trạng thái:** Chấp nhận — 22/10/2026. Cụ thể hoá ý "phân quyền nghiệp vụ nằm trong app" của [ADR-0008](0008-platform-auth-server.md).

## Bối cảnh

Role thô trong token (`ADMIN`, `SALES`) không đủ cho các yêu cầu thường gặp: quyền theo từng thao tác, theo bản ghi
(chi nhánh, người tạo, hạn mức), chặn tường minh một thao tác cho một nhóm, và để admin của khách hàng tự cấu hình mà
không cần deploy. Viết `if` theo role trong code thì mỗi dự án làm một kiểu, khó kiểm toán. AWS IAM là mô hình đã được
kiểm chứng và quen thuộc: policy JSON gồm `Effect`/`Action`/`Resource`/`Condition`, Deny tường minh luôn thắng.

## Quyết định

- **Mô hình IAM**, hiện thực trong `com.vyforge.platform:platform-starter-authz`; dữ liệu nằm trong database của app (PostgreSQL), khoá theo `tenant_id` + `sub`.
  - **Policy**: JSON `{"Statement":[{"Sid","Effect":"Allow|Deny","Action|NotAction","Resource|NotResource","Condition"}]}`.
  - **Action**: `<service>:<resource>:<Verb>`, vd. `crm:order:Approve`. Mỗi module khai báo action bằng enum (`PermissionAction`) — danh mục action là contract như mã lỗi.
  - **Resource**: `vrn:<service>:<resource>/<id>`, vd. `vrn:crm:order/0192…`. Wildcard `*`, `?`.
  - **Condition**: toán tử String/Numeric/Date/Bool/IpAddress/Null (có hậu tố `IfExists`, tiền tố `ForAnyValue:`/`ForAllValues:`) trên khoá `principal.*`, `resource.*`, `request.*`; biến `${principal.sub}`, `${principal.<thuộc tính>}` trong `Resource` và giá trị điều kiện.
  - **Đánh giá**: Deny tường minh > Allow > mặc định từ chối.
  - **Gắn policy**: policy dùng chung (managed) gắn vào group hoặc user; policy riêng (inline) của một user. Role trong token map sang group (`token_role`) để dùng song song với IdP.
- **Multi-tenant từ đầu**: mọi bảng IAM có `tenant_id`; tenant lấy từ claim của token (mặc định `tenant_id`, dự án một tenant dùng `default`).
- **Thuộc tính resource** cho Condition do app cung cấp qua `ResourceAttributesResolver` theo loại resource, chỉ được gọi khi statement cần khoá `resource.*`.
- **Kiểm tra trong code**: `@RequiresPermission(action = "...", resource = "...")`, `hasPermission(...)` trong `@PreAuthorize`, hoặc bean `Authz`; từ chối → 403 Problem Details. Frontend gọi `POST /api/v1/iam/me/evaluate` (kiểm tra nhiều quyền một lần) qua helper `can()`/`requirePermission()` của `@vyforge/auth` — chỉ để ẩn/hiện giao diện, backend vẫn là nơi quyết định.
- **Quản trị**: REST API `/api/v1/iam/**` (policy, group, gắn policy, danh mục action, Policy Simulator) được bảo vệ bởi chính IAM (action `iam:*`); role bootstrap `vyforge.authz.admin-roles` (mặc định `IAM_ADMIN`) có quyền `iam:*`. Thao tác quản trị ghi audit log. Component giao diện dùng chung nằm trong `@vyforge/iam` (platform-web).
- Token vẫn chỉ mang `sub`, `aud`, `scope`, role thô và `tenant_id` (ADR-0008); không nhét permission vào token.
- **Ranh giới quyền** (permission boundary, bổ sung 28/10/2026): bean `PermissionBoundaryProvider` trả các
  `PermissionBoundary` (tập policy) theo claim của token, vd. scope của token app ([ADR-0026](0026-api-access.md) quyết
  định 10). Quyền = policy ∧ mọi ranh giới; Deny ở đâu cũng thắng; ranh giới không cấp thêm quyền. Policy Simulator hiện
  kết quả từng ranh giới.

### Chưa làm (khi có dự án cần)

Policy cấp tổ chức kiểu SCP, session policy, phân quyền theo tag, điều kiện thời gian thực từ dịch vụ ngoài.

## Hệ quả

- Admin của khách hàng tự cấu hình quyền, không cần deploy; mọi thay đổi có audit log.
- Policy được cache theo `(tenant, sub)` (mặc định 60 giây, xoá ngay trên replica nhận thay đổi); thay đổi có thể chậm tối đa bằng TTL ở replica khác.
- Mỗi lần kiểm tra quyền có chi phí (đọc policy từ cache, có thể gọi resolver thuộc tính); danh sách dữ liệu lớn phải lọc theo điều kiện ở truy vấn, không kiểm tra từng bản ghi.
- Policy viết sai có thể mở quyền quá rộng: API từ chối policy sai cú pháp hoặc dùng action không có trong danh mục; dùng Policy Simulator trước khi gắn.
- `@PreAuthorize("hasRole(...)")` vẫn dùng được cho quyền thô; dự án mới dùng IAM cho quyền nghiệp vụ.

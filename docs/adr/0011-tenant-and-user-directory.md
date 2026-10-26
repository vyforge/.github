# ADR-0011: Tenant ở platform-auth-server, claim `tenant_id` và danh bạ người dùng

- **Trạng thái:** Chấp nhận — 26/10/2026. Làm tiếp phần "Chưa làm" của [ADR-0010](0010-api-gateway-central-iam.md)
  (tenant trong token, danh bạ người dùng). Mở rộng [ADR-0008](0008-platform-auth-server.md) (auth server) và
  [ADR-0009](0009-iam-authorization.md) (IAM khoá theo `tenant_id`).

## Bối cảnh

ADR-0009 cho mọi bảng IAM có cột `tenant_id` và lấy tenant từ claim của token. Tuy vậy platform-auth-server chưa phát
claim này, nên mọi app thực chất đang chạy một tenant (`default`). Có ba nhu cầu chưa đáp ứng được:

- Một nền tảng phục vụ nhiều khách hàng: dữ liệu và quyền của khách hàng này không được lộ sang khách hàng khác.
- Một người dùng có thể làm việc cho nhiều khách hàng, với role khác nhau ở mỗi nơi.
- Màn hình quản trị IAM và các service cần tra người dùng theo `sub`, email hoặc tên (để gắn policy, hiện tên thay vì
  UUID) mà không đọc database của auth server.

## Quyết định

### 1. Mô hình tenant

- **Tenant** (`tenant_tenants`) gồm:
  - `slug`: khoá chính, khớp `^[a-z0-9][a-z0-9-]{1,62}$`, không đổi được sau khi tạo vì nó chính là giá trị claim.
  - `name`.
  - `status`: `ACTIVE` hoặc `LOCKED`.
  - `version`, cùng thời điểm và người tạo/sửa.
- **Tài khoản vẫn dùng chung toàn nền tảng.** Mỗi tài khoản có một email, một mật khẩu và một cấu hình MFA. `sub` không
  đổi. Một tài khoản **thuộc được nhiều tenant**.
- **Membership** (`tenant_memberships`) gắn tài khoản với tenant. **Role được gán theo từng membership**
  (`tenant_membership_roles`), nên cùng một người có thể là `admin` ở tenant A và `user` ở tenant B. Tên role lấy từ
  danh mục role chung (`account_roles`). Danh mục không tách theo tenant.
- **Tenant `default` là tenant gốc của nền tảng** và có quy tắc riêng:
  - Mọi tài khoản mặc nhiên là thành viên của `default`.
  - Role ở `default` chính là role của tài khoản hiện có (`account_user_roles`), quản lý ở màn hình Users như trước.
  - Không khoá và không xoá được `default`.
  - Lý do: giữ nguyên hành vi cho mọi app và dữ liệu hiện có (xem mục 6), và không phải đổi luồng đăng ký, mời hay đăng
    nhập ngoài.
- Role `platform-admin` chỉ có nghĩa ở tenant `default` và không gán được trong tenant khác.

### 2. Chọn tenant khi đăng nhập

Tenant của một lần đăng nhập được xác định theo thứ tự sau:

1. **Theo client:** client gắn với một tenant (`tenant_clients`, mỗi client tối đa một tenant) luôn đăng nhập vào tenant
   đó. Nếu tham số `tenant` khác với tenant của client thì request bị từ chối.
2. **Theo tham số:** client không gắn tenant (dùng chung, như admin console hay app nền tảng) nhận tham số mở rộng
   `tenant=<slug>` ở `/oauth2/authorize`. BFF truyền tham số này, ví dụ khi người dùng chọn tổ chức ở app.
3. **Mặc định:** không có tham số thì dùng `default`.

Ràng buộc kèm theo:

- Người dùng phải là thành viên của tenant và tenant phải `ACTIVE`.
  - `/oauth2/authorize` kiểm tra ngay sau khi đăng nhập. Không đạt thì trả trang lỗi 403, không chuyển hướng về app.
  - Mỗi lần cấp token đều kiểm tra lại, kể cả khi refresh (xem mục 4).
- Tenant đã chọn được lưu cùng authorization request, nên refresh token luôn giữ tenant ban đầu.
- Chưa làm màn hình chọn tenant ở auth server (xem mục "Chưa làm").

### 3. Claim `tenant_id`

- Access token và ID token (cả userinfo) có claim `tenant_id` (string, là slug).
- `roles` là role của membership trong tenant đó, không phải hợp role của mọi tenant.
- **Client credentials:**
  - Tenant là tenant của client nếu client có gắn, không thì là `default`.
  - Tenant bị khoá thì trả `unauthorized_client`.
- **Token exchange (ADR-0010 §4):**
  - Token mới giữ nguyên `tenant_id` của `subject_token` (token cũ không có claim thì coi là `default`).
  - `roles` được tính lại theo membership của tenant đó.
  - Client đang đổi mà gắn với tenant khác thì trả `invalid_grant`.
- `platform-starter-authz` đọc claim `tenant_id`. Tuỳ chọn `vyforge.authz.tenant.required=true` từ chối (403) token
  không có claim.

### 4. Khoá tenant

- Tenant `LOCKED` không được cấp token mới dưới mọi grant:
  - `/oauth2/authorize`: trả 403.
  - `authorization_code`, `refresh_token`, `token_exchange`: trả `invalid_grant`.
  - `client_credentials`: trả `unauthorized_client`.
- Access token đã cấp vẫn còn hiệu lực tới khi hết hạn (tối đa `access-token-ttl`, 10 phút). Không thu hồi phiên trình
  duyệt, vì tài khoản vẫn dùng được ở tenant khác.
- Mở khoá thì mọi thứ hoạt động lại ngay, không cần cấp lại client hay membership.

### 5. Quản trị

- **Admin xuyên tenant** chỉ dành cho role `platform-admin` trong tenant `default`, hoặc token client credentials có scope
  `auth.admin` của tenant `default`. Cả hai đều phải đã qua MFA như ADR-0008 (MFA không áp dụng cho client credentials).
- API `/api/v1/admin/tenants`:
  - CRUD tenant, `POST …/{slug}/lock` và `…/unlock`.
  - `…/{slug}/members`: thêm theo email, sửa role, gỡ thành viên.
  - `…/{slug}/clients/{clientId}`: gắn hoặc gỡ client.
  - Mọi thao tác ghi nhật ký quản trị (`@Audited`, action `auth:tenant:*`).
- Các API quản trị hiện có (`/api/v1/admin/**`, role `admin`) **chỉ nhận token của tenant `default`**. Lý do: role giờ là
  role theo tenant, nên `admin` của một tenant khách hàng không được quyền quản trị toàn auth server.
- Chưa có quản trị uỷ quyền cho từng tenant: admin của tenant X chưa tự quản lý được tài khoản và membership của X.
- Admin console có bộ chọn tenant đang thao tác ở các trang tenant.

### 6. Danh bạ người dùng

API này dành cho service và màn hình quản trị IAM:

| Endpoint | Nội dung |
|---|---|
| `GET /api/v1/directory/users?q=&page=&size=` | `PageResponse<User>` của platform-starter-web (`content`, `page`, `size`, `totalElements`, `totalPages`). `q` tìm theo email hoặc tên, không phân biệt hoa thường |
| `GET /api/v1/directory/users/{sub}` | `User`. Trả 404 khi không có hoặc không thuộc tenant của token |

`User` = `{sub, username, email, displayName, status: ACTIVE|LOCKED|DISABLED, tenantId}`:

- `username` là email đăng nhập, vì auth server không có username riêng.
- `status` là `DISABLED` nếu tài khoản bị vô hiệu, `LOCKED` nếu đang bị khoá tạm vì đăng nhập sai, còn lại là `ACTIVE`.

Ràng buộc:

- **Quyền đọc:** token có `aud` chứa `platform-auth-server`, kèm một trong hai:
  - scope `directory.read` (client credentials của service);
  - token người dùng có role `admin` hoặc `IAM_ADMIN` trong tenant của token.
- **Cô lập tenant:** kết quả chỉ gồm thành viên của tenant trong token. Không có tham số nào để đọc chéo tenant. Danh bạ
  của `default` gồm mọi tài khoản (vì membership mặc nhiên).
- **Chống liệt kê tràn lan:**
  - `size` tối đa 50.
  - Rate limit theo người gọi (`sub` hoặc `client_id`), mặc định 60 request/phút/replica
    (`vyforge.auth.directory.requests-per-minute`). Vượt thì trả 429 kèm `Retry-After`.
  - Không trả mật khẩu, trạng thái MFA hay role.

### 7. Migrate dữ liệu cũ

Migration Flyway V020–V029 của module `tenant`:

- Tạo tenant `default` (`ACTIVE`).
- Mọi tài khoản và role hiện có tự động thuộc `default` theo quy tắc ở mục 1, không cần chép dữ liệu.
- Thêm role `platform-admin` vào danh mục và gán cho các tài khoản đang có role `admin`. Trước đây `admin` đã có toàn quyền
  auth server, nên quyền thực tế không mở rộng thêm.
- Client hiện có không gắn tenant, nên vẫn đăng nhập vào `default` như cũ.
- Token phát trước khi nâng cấp không có `tenant_id`. App vẫn coi là `default` cho tới khi bật `tenant.required`.

## Hệ quả

- Một auth server phục vụ được nhiều khách hàng. Người dùng có một tài khoản duy nhất và role riêng ở từng tenant. IAM
  (ADR-0009/0010) cô lập dữ liệu nhờ claim `tenant_id` thật.
- Thu hồi quyền theo tenant (khoá tenant, gỡ thành viên) có hiệu lực từ lần cấp token kế tiếp, tối đa 10 phút với access
  token đã cấp.
- `default` có ngữ nghĩa đặc biệt: danh bạ của nó thấy mọi tài khoản. App dành cho khách hàng nên dùng tenant riêng thay vì
  `default`.
- Client gắn tenant là cách an toàn nhất cho app của một khách hàng. Client dùng chung phải tự truyền tham số `tenant`.
- Danh bạ của tenant có rất nhiều thành viên lọc theo danh sách ID membership. Đủ dùng tới vài chục nghìn thành viên mỗi
  tenant. Lớn hơn thì cần read model riêng.

### Chưa làm

- Màn hình chọn tenant ở auth server khi người dùng thuộc nhiều tenant mà app không truyền `tenant`.
- Quản trị uỷ quyền theo tenant (admin của tenant tự quản lý thành viên).
- Danh mục role riêng theo tenant.
- Thương hiệu theo tenant.
- Gắn tenant cho client khai báo trong cấu hình (`vyforge.auth.clients.*.tenant`).
- Tự huỷ phiên hoặc refresh token khi khoá tenant.

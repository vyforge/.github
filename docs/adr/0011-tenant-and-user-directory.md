# ADR-0011: Tenant ở platform-auth-server, claim `tenant_id` và danh bạ người dùng

- **Trạng thái:** Chấp nhận — 26/10/2026. Làm tiếp phần "Chưa làm" của [ADR-0010](0010-api-gateway-central-iam.md)
  (tenant trong token, danh bạ người dùng). Mở rộng [ADR-0008](0008-platform-auth-server.md) (auth server) và
  [ADR-0009](0009-iam-authorization.md) (IAM khoá theo `tenant_id`).
- **Cập nhật 26/10/2026:** thêm quản trị uỷ quyền theo tenant (mục 8), trước đây ở "Chưa làm".

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
- Role `platform-admin` chỉ có nghĩa ở tenant `default` và không gán được trong tenant khác. Ở `default`, chỉ người đã có
  `platform-admin` (hoặc scope `auth.admin`) mới gán hoặc gỡ được role này; `admin` không tự nâng quyền, cũng không cấp
  cho người khác (`account.role_requires_platform_admin`, 403).

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
- **Tenant cố định theo phiên.** Tenant chọn ở lần cấp token đầu (claim `tenant_id` của access token) được lưu trong
  authorization ở server. Refresh token đọc lại đúng tenant đó, không chọn lại theo binding hay tham số hiện tại, và trả
  `invalid_grant` (người dùng đăng nhập lại) khi:
  - tenant không còn tồn tại hoặc đang bị khoá;
  - client đã được gắn vào một tenant khác với tenant đã lưu (ví dụ phiên bắt đầu khi client còn dùng chung);
  - client đã bị gỡ khỏi tenant (xem mục 5).
- Chưa làm màn hình chọn tenant ở auth server (xem mục "Chưa làm").

### 3. Claim `tenant_id`

- Access token và ID token (cả userinfo) có claim `tenant_id` (string, là slug).
- `roles` là role của membership trong tenant đó, không phải hợp role của mọi tenant.
- **Client credentials:**
  - Tenant là tenant của client nếu client có gắn, không thì là `default`.
  - Tenant bị khoá thì trả `unauthorized_client`.
  - Client từng gắn tenant rồi bị gỡ **không** rơi về `default`: trả `unauthorized_client` cho tới khi được gắn lại.
- **Token exchange (ADR-0010 §4):**
  - Token mới giữ nguyên `tenant_id` của `subject_token` (token cũ không có claim thì coi là `default`).
  - `roles` được tính lại theo membership của tenant đó.
  - Client đang đổi mà gắn với tenant khác thì trả `invalid_grant`.
  - Không đổi được sang audience của chính auth server (API quản trị `platform-auth-server`, danh bạ
    `platform-auth-directory`), kể cả khi audience đó nằm trong `audiences` của client: trả `invalid_target`.
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
- **Token uỷ quyền** (có claim `act`, từ token exchange) bị API quản trị và danh bạ từ chối (401): service không quản trị
  auth server hay đọc danh bạ thay mặt người dùng. Không cấp audience `platform-auth-server` cho client có grant
  `token_exchange`.
- API `/api/v1/admin/tenants`:
  - CRUD tenant, `POST …/{slug}/lock` và `…/unlock`. **Xoá** chỉ được khi tenant đã khoá
    (`tenant.delete_requires_lock`) và không còn client gắn vào (`tenant.has_clients`); FK `tenant_clients.tenant_id` là
    `ON DELETE RESTRICT`.
  - `…/{slug}/members`: thêm theo email, sửa role, gỡ thành viên.
  - `…/{slug}/clients/{clientId}`: gắn hoặc gỡ client. **Gỡ** client thu hồi mọi authorization đã cấp qua client đó
    (event `TenantClientUnbound`) và đánh dấu client "đã gỡ" (`tenant_released_clients`): mọi grant bị từ chối cho tới
    khi gắn lại vào một tenant, có thể là `default`. Client của khách hàng không bao giờ âm thầm thành client dùng chung.
  - Mọi thao tác ghi nhật ký quản trị (`@Audited`, action `auth:tenant:*`).
- Các API quản trị hiện có (`/api/v1/admin/**`, role `admin`) **chỉ nhận token của tenant `default`**. Lý do: role giờ là
  role theo tenant, nên `admin` của một tenant khách hàng không được quyền quản trị toàn auth server.
- Admin của từng tenant tự quản lý thành viên của tenant mình qua API riêng, xem mục 8.
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

- **Audience riêng:** token có `aud` chứa `platform-auth-directory` (`vyforge.auth.directory.audience`). Service đọc danh
  bạ chỉ cần audience này, không cần audience của API quản trị. Token có `platform-auth-server` (admin console) cũng được
  nhận. Token exchange không đổi sang được audience này, và token có claim `act` bị từ chối.
- **Quyền đọc**, một trong hai:
  - scope `directory.read` (client credentials của service);
  - token người dùng có role `admin` hoặc `IAM_ADMIN` trong tenant của token **và đã qua MFA** (`amr` chứa `mfa`, cùng
    cấu hình `vyforge.auth.admin-api.require-mfa` với API quản trị).
- **Cô lập tenant:** kết quả chỉ gồm thành viên của tenant trong token. Không có tham số nào để đọc chéo tenant. Danh bạ
  của `default` gồm mọi tài khoản (vì membership mặc nhiên).
- **Chống liệt kê tràn lan:**
  - `size` tối đa 50.
  - Rate limit theo người gọi (`sub` hoặc `client_id`), mặc định 60 request/phút/replica
    (`vyforge.auth.directory.requests-per-minute`). Vượt thì trả 429 kèm `Retry-After`.
  - Không trả mật khẩu, trạng thái MFA hay role.

### 7. Migrate dữ liệu cũ

Migration Flyway của module `tenant` dùng dải V020–V029 (hiện V020–V025). **Nhánh này merge và deploy trước nhánh dùng
V030+** (`feat/idp-extensions`). Deploy sai thứ tự thì auth server dừng lúc khởi động với lỗi nêu rõ migration bị bỏ
qua; chạy bù bằng một lần deploy với `AUTH_FLYWAY_OUT_OF_ORDER=true` (các migration này chỉ tạo bảng và thêm dữ liệu).

Nội dung:

- Tạo tenant `default` (`ACTIVE`).
- Mọi tài khoản và role hiện có tự động thuộc `default` theo quy tắc ở mục 1, không cần chép dữ liệu.
- Thêm role `platform-admin` vào danh mục và gán cho các tài khoản đang có role `admin`. Trước đây `admin` đã có toàn quyền
  auth server, nên quyền thực tế không mở rộng thêm.
- Client hiện có không gắn tenant, nên vẫn đăng nhập vào `default` như cũ.
- V025 (role `TENANT_ADMIN`, mục 8) ra sau V030–V032. Database đã chạy V030+ cần đúng một lần deploy với
  `AUTH_FLYWAY_OUT_OF_ORDER=true`. Migration này chỉ thêm một dòng vào `account_roles` (`ON CONFLICT DO NOTHING`).
- Token phát trước khi nâng cấp không có `tenant_id`. App vẫn coi là `default` cho tới khi bật `tenant.required`.

### 8. Quản trị uỷ quyền theo tenant

Người của khách hàng tự quản lý thành viên và phân quyền trong tenant của mình, không cần người vận hành platform.

**Role.** Role `TENANT_ADMIN` có sẵn trong danh mục chung (migration V025), nên tenant nào cũng dùng được. Role này
gán theo membership như mọi role khác: platform admin gán qua `/api/v1/admin/tenants/{slug}/members`, hoặc một tenant
admin gán cho người khác. Role không có nghĩa ở tenant `default`. Để đặt được tên này, tên role nay được viết toàn chữ
thường (`report-viewer`) hoặc toàn chữ hoa (`TENANT_ADMIN`, `IAM_ADMIN`). Không được trộn hoa thường, để không có hai
role chỉ khác nhau ở cách viết.

**API ở auth server** `/api/v1/tenant-admin/**`. Không có tham số chọn tenant: tenant luôn là claim `tenant_id` của
token.

| Endpoint | Nội dung |
|---|---|
| `GET …/tenant` | Tenant đang quản lý và các role người gọi cấp được |
| `GET …/members?q=&page=&size=` | Thành viên của tenant (`PageResponse`) |
| `POST …/members` `{email, displayName?, roles}` | Email đã có tài khoản thì thêm luôn. Chưa có thì tạo tài khoản (không có role ở `default`) và gửi email mời đặt mật khẩu bằng mailer sẵn có (`invited: true`). Đã là thành viên thì 409 `tenant.member_exists` |
| `PUT …/members/{accountId}` `{roles}` | Thay role trong tenant |
| `DELETE …/members/{accountId}` | Gỡ khỏi tenant |

Token được nhận khi đủ các điều kiện:

- `aud` chứa audience riêng `platform-auth-tenant-admin` (`vyforge.auth.tenant-admin.audience`). Không nhận audience
  của API quản trị hay danh bạ. Token exchange không đổi sang được audience này (`invalid_target`, như mục 3).
- Token người dùng có role `TENANT_ADMIN` và đã qua MFA, cùng cờ `vyforge.auth.admin-api.require-mfa` với API quản trị.
  Không nhận client credentials, vì tenant admin luôn là một người.
- Token có claim `act` bị từ chối (401), như mục 5.
- Role trong token chỉ là lớp chặn đầu. Mỗi request kiểm lại từ database: tenant khác `default` và đang `ACTIVE`, người
  gọi còn là thành viên có `TENANT_ADMIN`, tài khoản chưa bị vô hiệu. Không đạt thì 403 `tenant.admin_required`. Nhờ
  vậy gỡ quyền, khoá tenant hay vô hiệu tài khoản có hiệu lực ngay, không đợi token hết hạn.

**Chống leo thang:**

- **Chỉ cấp role chính mình đang có trong tenant** (403 `tenant.role_not_held`). Hệ quả:
  - không tự thêm role cho mình;
  - không gán được `platform-admin` (vốn không có ở tenant khác `default`);
  - không gán được role quản trị mà mình không có, như `admin` hay `IAM_ADMIN` (role toàn quyền IAM).
- Role mà thành viên đã có từ trước (do platform admin gán) thì giữ nguyên hoặc gỡ được, nhưng không cấp lại được.
- **Tenant luôn còn ít nhất một tenant admin dùng được.** Gỡ role hay gỡ thành viên mà làm tenant không còn tenant
  admin nào có tài khoản đang bật thì trả 409 `tenant.last_admin`. Mọi thao tác ghi khoá dòng tenant
  (`SELECT … FOR UPDATE`), nên hai tenant admin gỡ nhau cùng lúc thì một người bị từ chối. Platform admin không bị luật
  này ràng buộc.
- Không chạm được tenant khác: thành viên tenant khác trả 404 như không tồn tại. Token ghi tenant mà người gọi không
  quản trị thì trả 403.

**Audit và giới hạn:** mọi thao tác ghi nhật ký quản trị (`@Audited`, action `auth:tenant-admin:*`, resource
`tenant/<slug>`), kể cả lần bị từ chối. Rate limit theo người gọi như danh bạ.

**IAM (platform-iam-server).** Có hai cách, chọn cách thứ hai:

- **Cách 1 — thêm `TENANT_ADMIN` vào `vyforge.authz.admin-roles`.** Không chọn. Role trong `admin-roles` có toàn quyền
  `iam:*`, và `IamAdminService` bỏ qua luật chống leo thang với người có toàn quyền. Khi đó tenant admin tự gắn được
  policy bất kỳ cho mình, kể cả action nghiệp vụ.
- **Cách 2 — policy dựng sẵn `vyforge:TenantAdministrator`.** Đã chọn. Người có role `TENANT_ADMIN` ở tenant khác
  `default` nhận policy này trong tenant của token. Policy cho mọi action `iam:*` trên `*`, trừ
  `iam:user:PutInlinePolicy` và `DeleteInlinePolicy`. Mọi bảng IAM khoá theo `tenant_id`, nên quyền không vượt ra ngoài
  tenant. Vì thiếu hai action đó, người giữ policy không có toàn quyền và luật chống leo thang vẫn áp dụng:
  - không tự sửa quyền của mình (gắn policy cho mình, vào group, group có `tokenRole` trùng role mình);
  - chỉ cấp action mình đang có (`iam.grant_not_held`). Muốn tenant admin uỷ quyền tiếp action nghiệp vụ (ví dụ
    `crm:*`), một admin toàn quyền phải gắn policy đó cho tenant admin trước;
  - mọi quyền cấp đi đều là policy có tên, thấy được trong danh sách và audit log. Policy riêng (inline) để dành cho
    admin toàn quyền.

  Policy nằm trong code của IAM server, không nằm trong database, giống `vyforge:IamAdministrator` của `admin-roles`.
  Lý do: không ai sửa hay xoá được, tenant mới có ngay mà không cần seed theo từng tenant, và không phải đồng bộ danh sách
  tenant giữa hai server. API nội bộ (`/internal/resolve`) không trả policy này, vì nó chỉ gồm action `iam:*` mà chỉ IAM
  server kiểm tra. IAM server từ chối khởi động nếu `admin-roles` chứa role tenant admin (`vyforge.iam.tenant-admin-role`).

**Màn hình.** Tenant admin dùng trang `/admin/iam` có sẵn trong app nghiệp vụ (`@vyforge/iam`, template-nextjs khi
`use_iam`), cộng thêm màn hình thành viên gọi API ở trên. Không dùng `platform-admin`.

## Hệ quả

- Một auth server phục vụ được nhiều khách hàng. Người dùng có một tài khoản duy nhất và role riêng ở từng tenant. IAM
  (ADR-0009/0010) cô lập dữ liệu nhờ claim `tenant_id` thật.
- Thu hồi quyền theo tenant (khoá tenant, gỡ thành viên, gỡ client) có hiệu lực từ lần cấp token kế tiếp, tối đa 10 phút
  với access token đã cấp. Phiên không bao giờ âm thầm đổi tenant: đổi binding hay xoá tenant buộc đăng nhập lại.
- Xoá tenant là thao tác nhiều bước có chủ đích: gỡ client, khoá, rồi xoá.
- `default` có ngữ nghĩa đặc biệt: danh bạ của nó thấy mọi tài khoản. App dành cho khách hàng nên dùng tenant riêng thay vì
  `default`.
- Client gắn tenant là cách an toàn nhất cho app của một khách hàng. Client dùng chung phải tự truyền tham số `tenant`.
- Khách hàng tự mời, phân quyền và gỡ thành viên trong tenant của mình. Quyền tối đa của tenant admin do platform
  admin quyết định: tenant admin chỉ chia lại role và quyền IAM mà chính họ đang có.
- Audience riêng không ngăn được service đang nhận token của người dùng gọi lại API quản trị uỷ quyền, vì access token
  mang mọi audience khai báo cho client. Ví dụ: BFF có cả `platform-gateway` và `platform-auth-tenant-admin` thì backend
  phía sau gateway cũng nhận token có audience đó. Lớp chặn còn lại là MFA, kiểm tra lại từ database và audit. App nào
  mở màn hình tenant admin nên dùng client riêng cho khu quản trị, chỉ có audience `platform-auth-tenant-admin`.
- Tenant admin phải đăng nhập có MFA. App cần bắt người dùng xác thực lại có MFA (step-up) trước khi vào khu quản trị.
- Một tenant admin có thể biết một email đã có tài khoản trên nền tảng hay chưa (`invited`). Mỗi lần thêm đều ghi
  audit và bị rate limit.
- Danh bạ của tenant có rất nhiều thành viên lọc theo danh sách ID membership. Đủ dùng tới vài chục nghìn thành viên mỗi
  tenant. Lớn hơn thì cần read model riêng.

### Chưa làm

- Màn hình chọn tenant ở auth server khi người dùng thuộc nhiều tenant mà app không truyền `tenant`.
- Đọc audit của quản trị uỷ quyền. Audit được ghi vào `audit_log` theo tenant của token (ví dụ `acme`). Tuy vậy trang
  Activity của admin console (`/api/v1/admin/activity`) chỉ đọc tenant `default`, và chưa có API đọc cho tenant admin.
  Cần thêm cả hai: platform admin lọc theo tenant, tenant admin đọc tenant của mình.
- Màn hình thành viên cho tenant admin trong `@vyforge/iam` và template-nextjs.
- Danh mục role riêng theo tenant.
- Thương hiệu theo tenant.
- Gắn tenant cho client khai báo trong cấu hình (`vyforge.auth.clients.*.tenant`).
- Tự huỷ phiên hoặc refresh token khi khoá tenant.

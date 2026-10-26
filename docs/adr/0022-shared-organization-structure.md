# ADR-0022: Cơ cấu tổ chức dùng chung

- **Trạng thái:** Chấp nhận — 27/10/2026. Thiết kế chi tiết: [organization.md](../design/organization.md). Liên quan
  [ADR-0009](0009-iam-authorization.md), [ADR-0011](0011-tenant-and-user-directory.md),
  [ADR-0015](0015-tenant-data-isolation-platform-core.md).

## Bối cảnh

Ít nhất tám module cần biết "ai thuộc đơn vị nào, ai là quản lý của ai": workflow (người duyệt), collaboration (giao
việc cho đơn vị), search (tài liệu chia cho phòng ban), forms, cms-content, features (giới hạn số chi nhánh), helpdesk,
booking. Không module nào trong số đó nên sở hữu cơ cấu tổ chức. Doanh nghiệp và cơ quan nhà nước Việt Nam đổi cơ cấu
theo quyết định có số hiệu và ngày hiệu lực (sắp xếp bộ máy 2025–2026: gộp phòng, sở), nên cơ cấu phải có lịch sử và
nhập trước được thay đổi tương lai.

## Quyết định

1. **Module riêng `organization`** (`platform-starter-organization`, `@vyforge/organization`, admin MFE
   `/m/organization`), action `organization:*`, bảng `org_*`. Danh tính người vẫn ở auth-server: module chỉ lưu `sub`
   (và id `PERSON` do app cấp cho người không có tài khoản), không lưu tên, email.
2. **Hiệu lực theo thời gian (valid time)** cho phiên bản đơn vị, phân công, quan hệ (`tstzrange` + exclusion
   constraint, cần `btree_gist`). Mọi API đọc nhận `at`. Không làm bitemporal: trục "hệ thống biết gì lúc nào" lấy từ
   nhật ký đợt thay đổi bất biến và audit.
3. **Lưu cây:** danh sách kề có hiệu lực là nguồn sự thật; **closure table có hiệu lực** (`org_unit_tree`) là chỉ mục
   dẫn xuất, tính lại trong cùng transaction cho các nhánh bị chạm, có job kiểm và dựng lại. Không dùng `ltree` hay nested
   set.
4. **Đường đọc nóng không chạm DB:** snapshot bất biến theo tenant, tự hết hạn ở mốc hiệu lực kế tiếp, xoá qua event sau
   commit, Valkey (`vyforge:organization:changed`) và poll revision.
5. **Đợt thay đổi là đường ghi duy nhất:** nháp → (duyệt: none / four-eyes / workflow) → lên lịch → có hiệu lực; có số
   quyết định, xem trước tác động, huỷ (khôi phục đúng dòng cũ từ nhật ký), hoàn tác (đợt ngược). Sáp nhập/chia tách ghi
   quan hệ kế thừa (`successorsOf`). Các đợt của một tenant áp tuần tự (advisory lock).
6. **Quản lý trực tiếp suy ra từ cây**, không lưu như một trường độc lập: `reports_to` khai báo → trưởng đơn vị chính
   (gồm người phụ trách) → đơn vị cha khi trống hoặc chính người đó là trưởng. Một chỗ duy nhất (`ManagerResolver`), có
   lời giải thích từng bước cho workflow.
7. **Phân quyền theo cơ cấu qua IAM có sẵn:** thuộc tính `principal.orgUnits|primaryUnit|branch|company|managedUnits…`
   và `resource.unitPath`/`orgUnitPath` (mã luôn chữ thường). `-authz` 0.13 mở biến nhiều giá trị trong giá trị điều kiện
   thành OR (tối đa 1.024 giá trị, vượt thì coi như thiếu biến), để viết "bản ghi thuộc nhánh của bất kỳ đơn vị nào tôi
   quản lý". Quản trị cơ cấu uỷ quyền theo nhánh; đợt thay đổi kiểm quyền trên từng thao tác.
8. **Không phụ thuộc cứng module khác**; ba cách chạy như mọi module (nhúng, `platform-services`, server riêng). App ở
   `mode=remote` tải snapshot về bộ nhớ nên đường đọc vẫn không có I/O.

## Hệ quả

- Workflow, collaboration, search… dùng `OrgDirectory` qua `@ConditionalOnBean`; chữ ký giữ đúng port của workflow.
- Cần extension `btree_gist` (trusted). Ràng buộc chồng lấn nằm ở DB, validate ở code chỉ để báo lỗi dễ hiểu.
- Ghi cấu trúc đắt hơn đọc (nạp lịch sử của tenant, tính lại closure); chấp nhận vì ghi vài lần mỗi tháng.
- Retention ẩn danh hoá phân công đã kết thúc (mặc định 24 tháng, Luật 91/2025 Điều 25 khoản 2) — thời hạn cần pháp chế
  xác nhận.
- Để sau: định biên/vị trí việc làm, đồng bộ HRM, đồng bộ đơn vị thành group IAM, bản sao closure cho app `remote`.

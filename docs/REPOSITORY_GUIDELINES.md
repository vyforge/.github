# Quy chuẩn quản lý repository

Vyforge triển khai nhiều dự án song song. Tài liệu này giúp tổ chức source code thống nhất, dễ tìm và phân quyền đúng.

## 1. Đặt tên repo

```
<du-an>-<thanh-phan>
```

- Chữ thường, ngăn cách bằng `-`, không dấu, không khoảng trắng.
- `<du-an>`: mã dự án / khách hàng ngắn gọn (vd. `crm`, `hrm`, `acme-shop`).
- `<thanh-phan>`: `backend`, `frontend`, `mobile`, `infra`, `docs`, `<ten-service>-service`...

| Ví dụ | Ý nghĩa |
|---|---|
| `crm-backend` | Backend dự án CRM |
| `crm-frontend` | Web dự án CRM |
| `acme-shop-order-service` | Microservice Order của dự án Acme Shop |
| `acme-shop-infra` | Terraform/Helm của Acme Shop |
| `platform-*` | Thư viện, tool dùng chung nhiều dự án (vd. `platform-auth-sdk`) |
| `template-*` | Repo mẫu để tạo dự án mới (bật **Template repository**) |

## 2. Gắn nhãn (topics)

Mỗi repo gắn topics để lọc theo dự án:

- `project-<du-an>` (vd. `project-crm`)
- Loại: `backend`, `frontend`, `mobile`, `infra`, `library`
- Công nghệ: `spring-boot`, `nextjs`, ...
- Trạng thái nếu ngừng phát triển: **Archive** repo thay vì xoá.

Tìm toàn bộ repo của một dự án: `https://github.com/orgs/vyforge/repositories?q=topic:project-crm`

## 3. Team & phân quyền

Tạo team theo dự án, quyền gán cho **team**, không gán cho cá nhân.

```
vyforge
├── engineering            (toàn bộ kỹ sư — Read repo platform-*)
├── devops                 (Admin hạ tầng, CI/CD)
├── project-crm            (team cha của dự án CRM)
│   ├── project-crm-leads  → Maintain
│   └── project-crm-devs   → Write
└── project-acme-shop
    ├── project-acme-shop-leads
    └── project-acme-shop-devs
```

| Vai trò | Quyền repo |
|---|---|
| Trưởng dự án / Tech lead | Maintain |
| Developer | Write |
| QA / BA / khách hàng xem code | Read / Triage |
| DevOps | Admin |

Cài đặt org khuyến nghị (**Settings → Member privileges**):

- Base permission: **No permission** (chỉ thấy repo được cấp qua team).
- Repository creation: chỉ Owner (hoặc bật nhưng chỉ Private).
- Bắt buộc **2FA** cho toàn bộ thành viên.
- Repo mặc định **Private**; chỉ công khai khi được phê duyệt.

## 4. Bảo vệ nhánh (Rulesets)

Tạo **Organization ruleset** (Settings → Repository → Rulesets) áp dụng cho mọi repo, nhánh `main` và `develop`:

- [x] Require a pull request before merging — tối thiểu 1 approval
- [x] Dismiss stale approvals khi có commit mới
- [x] Require review from Code Owners
- [x] Require status checks to pass
- [x] Block force pushes
- [x] Restrict deletions

## 5. Khởi tạo dự án mới — checklist

1. [ ] Chọn mã dự án `<du-an>`.
2. [ ] Tạo team `project-<du-an>`, `-leads`, `-devs` và thêm thành viên.
3. [ ] Tạo repo từ `template-*` phù hợp (Private) theo quy tắc đặt tên.
4. [ ] Gắn topics `project-<du-an>` + loại + công nghệ.
5. [ ] Cấp quyền repo cho các team.
6. [ ] Thêm `CODEOWNERS` (vd. `* @vyforge/project-<du-an>-leads`).
7. [ ] Thêm `README.md` (giới thiệu, cách chạy local, môi trường, người phụ trách) và `.gitignore`.
8. [ ] Thêm CI từ **Actions → New workflow → By Vyforge**.
9. [ ] Khai báo secrets / environments (`dev`, `staging`, `production`) — không commit vào code.
10. [ ] Tạo GitHub Project (board) cho dự án nếu dùng GitHub để quản lý task.

## 6. Kết thúc dự án

- Archive các repo của dự án, giữ topic `project-<du-an>`.
- Thu hồi quyền của thành viên ngoài / khách hàng.
- Thu hồi secrets, deploy keys liên quan.

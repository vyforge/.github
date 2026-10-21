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

## 5. Chuẩn công nghệ (cập nhật 10/2026)

Dự án mới bắt buộc dùng phiên bản dưới đây (template đã cấu hình sẵn); dự án cũ lên kế hoạch nâng cấp. Kiến trúc chi tiết: [ARCHITECTURE.md](ARCHITECTURE.md).

| Thành phần | Phiên bản | Ghi chú |
|---|---|---|
| Java | **25 (LTS)** | Không dùng Java 21 cho dự án mới |
| Spring Boot | **4.1.x** | 3.5 đã hết hỗ trợ OSS 30/06/2026; 4.0 hết hỗ trợ 31/12/2026 |
| Spring Modulith | **2.1.x** | Modular monolith, xem ADR-0003 |
| Maven | **3.9.x** (qua `mvnw`) | Maven 4 chưa GA (ADR-0001) |
| Node.js | **24 (LTS)** | Chuyển lên 26 sau khi 26 thành LTS (28/10/2026) |
| pnpm | **11.x** | Ghim qua `packageManager`; chặn package phát hành < 24 giờ |
| Next.js | **≥ 16.4.1** | 16.0–16.4.0 có lỗ hổng RCE; Next.js 15 hết hỗ trợ 21/10/2026 |
| React | **19.x** | Theo Next.js 16 |
| TypeScript | **6.0.x** | Chưa lên 7: `typescript-eslint` chỉ hỗ trợ < 6.1 |
| ESLint | **9.x** | Chưa lên 10: plugin của `eslint-config-next` chưa hỗ trợ |

Rà soát bảng này mỗi quý. Mọi repo bật **Renovate** với preset của org (`github>vyforge/.github:renovate-config`, ADR-0004).

## 6. Khởi tạo dự án mới — checklist

1. [ ] Chọn mã dự án `<du-an>` và tier (0/1/2 — xem ARCHITECTURE.md).
2. [ ] Tạo team `project-<du-an>`, `-leads`, `-devs` và thêm thành viên.
3. [ ] Sinh code bằng Copier, rồi tạo repo Private theo quy tắc đặt tên và push lên:
   - Backend: `copier copy --trust gh:vyforge/template-spring-boot <du-an>-backend`
   - Frontend: `copier copy --trust gh:vyforge/template-nextjs <du-an>-frontend`
4. [ ] Giữ file `.copier-answers.yml` để sau này `copier update`.
5. [ ] Gắn topics `project-<du-an>` + loại + công nghệ.
6. [ ] Cấp quyền repo cho các team (`CODEOWNERS` đã sinh sẵn, trỏ tới `@vyforge/project-<du-an>-leads`).
7. [ ] Cài Renovate app cho repo (template đã có `renovate.json`).
8. [ ] CI đã có sẵn (`.github/workflows/ci.yml` gọi reusable workflow của org) — kiểm tra chạy xanh ở PR đầu tiên.
9. [ ] Khai báo secrets / environments (`dev`, `staging`, `production`) — không commit vào code.
10. [ ] Tạo GitHub Project (board) cho dự án nếu dùng GitHub để quản lý task.

## 7. Kết thúc dự án

- Archive các repo của dự án, giữ topic `project-<du-an>`.
- Thu hồi quyền của thành viên ngoài / khách hàng.
- Thu hồi secrets, deploy keys liên quan.

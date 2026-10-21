# vyforge/.github

Repo cấu hình chung cho toàn bộ tổ chức **Vyforge** trên GitHub. Các file ở đây được GitHub tự động áp dụng cho **mọi repo** trong org nếu repo đó chưa có file riêng.

| Thành phần | Vị trí | Tác dụng |
|---|---|---|
| Trang giới thiệu org | `profile/README.md` | Hiển thị tại github.com/vyforge |
| Issue templates | `.github/ISSUE_TEMPLATE/` | Mẫu Bug / Feature / Task cho mọi repo |
| PR template | `.github/pull_request_template.md` | Checklist khi mở Pull Request |
| Quy trình đóng góp | `CONTRIBUTING.md` | Branch, commit, review, merge |
| Bảo mật | `SECURITY.md` | Cách báo cáo lỗ hổng |
| Hỗ trợ | `SUPPORT.md` | Kênh hỏi đáp nội bộ |
| Quy tắc ứng xử | `CODE_OF_CONDUCT.md` | Chuẩn mực làm việc |
| Workflow templates | `workflow-templates/` | CI mẫu chọn được ở tab **Actions → New workflow** |
| Quy chuẩn quản lý repo | `docs/REPOSITORY_GUIDELINES.md` | Đặt tên repo, team, phân quyền, branch protection, khởi tạo dự án mới |

> Lưu ý: `CODEOWNERS`, `LICENSE`, branch protection và secrets **không** được kế thừa — phải cấu hình ở từng repo (xem `docs/REPOSITORY_GUIDELINES.md`).

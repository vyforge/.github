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
| Chuẩn kiến trúc | `docs/ARCHITECTURE.md` | Kiến trúc backend/frontend, tier, quy tắc bắt buộc/khuyến nghị |
| Quyết định kiến trúc | `docs/adr/` | ADR cho các lựa chọn dùng chung (Maven, Modulith, PostgreSQL, auth server, Renovate…) |
| Reusable workflows | `.github/workflows/` | CI/CD dùng chung, gọi bằng `uses: vyforge/.github/.github/workflows/<file>@v1`: `java-ci`, `node-ci`, `container` (image + Trivy + SBOM/provenance), `gitops-bump`, `maven-publish`, `npm-publish`, `helm-publish` |
| Workflow templates | `workflow-templates/` | File CI mỏng gọi reusable workflow, chọn ở **Actions → New workflow** |
| Renovate preset | `renovate-config.json` | `{"extends": ["github>vyforge/.github:renovate-config"]}` |
| Thư viện platform | `platform-java`, `platform-web`, `platform-helm-charts` | Parent POM + Spring Boot starter, package `@vyforge/*`, Helm chart chung (xem `docs/ARCHITECTURE.md`) |
| Quy chuẩn quản lý repo | `docs/REPOSITORY_GUIDELINES.md` | Đặt tên repo, team, phân quyền, branch protection, khởi tạo dự án mới |

## Tạo dự án mới

```bash
pipx install copier
copier copy --trust gh:vyforge/template-spring-boot <du-an>-backend
copier copy --trust gh:vyforge/template-nextjs <du-an>-frontend
```

Chi tiết: [`template-spring-boot`](https://github.com/vyforge/template-spring-boot), [`template-nextjs`](https://github.com/vyforge/template-nextjs).

> Lưu ý: `CODEOWNERS`, `LICENSE`, branch protection và secrets **không** được kế thừa — phải cấu hình ở từng repo (xem `docs/REPOSITORY_GUIDELINES.md`).

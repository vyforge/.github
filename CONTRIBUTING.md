# Quy trình đóng góp mã nguồn

Áp dụng cho mọi repo trong Vyforge, trừ khi repo có `CONTRIBUTING.md` riêng.

## 1. Mô hình nhánh

| Nhánh | Mục đích | Ghi chú |
|---|---|---|
| `main` | Code ổn định, đang chạy production | Được bảo vệ, chỉ merge qua PR |
| `develop` | Tích hợp cho bản phát hành tiếp theo (tuỳ dự án) | Được bảo vệ |
| `feature/<ticket>-<mo-ta>` | Tính năng mới | Tách từ `develop` (hoặc `main`) |
| `fix/<ticket>-<mo-ta>` | Sửa lỗi | |
| `hotfix/<mo-ta>` | Sửa gấp trên production | Tách từ `main`, merge lại cả `develop` |
| `release/<version>` | Chuẩn bị phát hành | Tuỳ dự án |

Ví dụ: `feature/CRM-123-login-otp`, `fix/CRM-140-null-pointer-invoice`.

## 2. Commit message — Conventional Commits

```
<type>(<scope>): <mô tả ngắn>
```

`type`: `feat` · `fix` · `docs` · `style` · `refactor` · `perf` · `test` · `build` · `ci` · `chore` · `revert`

Ví dụ: `feat(auth): thêm đăng nhập bằng OTP`, `fix(invoice): xử lý hoá đơn rỗng`.

## 3. Pull Request

1. Cập nhật nhánh với nhánh đích trước khi mở PR.
2. Tiêu đề PR theo định dạng commit ở trên.
3. Điền đầy đủ PR template, liên kết ticket/issue.
4. Cần **tối thiểu 1 approve** (repo quan trọng: 2) và CI xanh.
5. Ưu tiên **Squash and merge**; xoá nhánh sau khi merge.
6. Không push trực tiếp lên `main` / `develop`.

## 4. Quy tắc chung

- Không commit secrets, mật khẩu, `.env`, file build hoặc dữ liệu khách hàng.
- Mỗi PR nên nhỏ, tập trung một mục tiêu (< 400 dòng thay đổi là lý tưởng).
- Code mới phải có test tương ứng.
- Phiên bản phát hành theo [SemVer](https://semver.org/): `vMAJOR.MINOR.PATCH`.

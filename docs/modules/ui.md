# @vyforge/ui — component giao diện nền

## 1. Mục đích

Bộ component React dùng chung cho mọi app Next.js của Vyforge, khu quản trị và package giao diện của từng module
(`@vyforge/<module>/ui`, `/admin`): nút, form, bảng, phân trang, thẻ, khung trang quản trị, hộp thoại, toast… trên
Tailwind CSS v4 và Radix, để các màn hình trông và hành xử giống nhau. Phần lớn component dùng được trong Server Component.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Package web | `@vyforge/ui` 0.5.0 (platform-web) | không có README riêng; mô tả ở README gốc platform-web |
| Subpath | `.` (component), `./theme.css` (token Tailwind v4) | |
| Phụ thuộc | `radix-ui`, `sonner`, `clsx`, `tailwind-merge`, `@vyforge/i18n`; peer `react` 19, `react-dom` 19, `tailwindcss` 4 | |
| Storybook | `pnpm --filter @vyforge/ui storybook` | story: app-shell, button, form |

## 3. Tính năng chính

- **Server Component được**: `cn`, `Button`/`buttonVariants`, `Input`, `Textarea`, `NativeSelect`, `Label`,
  `FormField`, `Field`/`FieldError`/`FormError`, `Table*`, `Pagination`/`pageItems` (0-based như `PageResponse` của
  backend, phân trang bằng link), `Badge` (tone), `Card*`, `StatCard`, `PageHeader`, `Breadcrumbs`, `DescriptionList`,
  `Avatar`/`initials`, `EmptyState`, `Alert`, `Skeleton`, `Progress`, `AppShell`/`Sidebar`/`SidebarItem`/`SidebarSection`
  (khung trang quản trị: thanh bên cố định từ `md`, ngăn kéo trên mobile).
- **Client (Radix)**: `Checkbox`, `Select*`, `Dialog*`, `Sheet*`, `DropdownMenu*`, `Tabs*`, `Tooltip`, `CopyButton`,
  `Toaster`/`toast` (sonner).
- **Đa ngôn ngữ**: `LocalizedTextField` (tab theo ngôn ngữ, input `<name>.<locale>`, đánh dấu tab trống, ngôn ngữ bắt
  buộc, đếm ký tự; đọc lại bằng `localizedFromForm` của `@vyforge/i18n`), `LocalizedText` (hiện bản dịch theo locale).
- **Icon** cơ bản (`CheckIcon`, `ChevronDownIcon`, `InfoIcon`, `XIcon`…); kiểu `LinkComponent` để truyền `Link` của
  next-intl.
- **Theme**: `theme.css` khai token màu/khoảng cách cho Tailwind v4 (nền `canvas`…), sáng/tối.

## 4. Cách dùng nhanh

```bash
pnpm add @vyforge/ui
```

```css
/* src/app/globals.css */
@import 'tailwindcss';
@import '@vyforge/ui/theme.css';
```

```tsx
import { Button, Card, CardContent, PageHeader, Pagination } from '@vyforge/ui';

export default async function CustomersPage() {
  return (
    <main className="mx-auto max-w-5xl p-4">
      <PageHeader title="Khách hàng" />
      <Card><CardContent className="pt-6">…</CardContent></Card>
      <Button>Thêm</Button>
    </main>
  );
}
```

(Cách import `theme.css` cần xác nhận theo template-nextjs — xem `src/app/globals.css` của template.)

## 5. Cấu hình quan trọng

Không có cấu hình runtime. Yêu cầu Tailwind CSS 4 trong app (quét class của package trong `node_modules/@vyforge/ui`).

## 6. Tích hợp

Nền cho `@vyforge/admin-kit`, `@vyforge/admin-shell/ui`, `@vyforge/forms/renderer` (theme rjsf), và mọi `/ui`, `/admin`
của package module; template-nextjs và platform-admin dùng trực tiếp.

## 7. Triển khai

Thư viện; phát hành qua registry `@vyforge` khi merge platform-web (nâng version trong cùng branch khi sửa).

## 8. Trạng thái & còn mở

- Chưa có README trong package (chỉ có dòng mô tả ở README gốc platform-web).
- Bộ icon tối giản; module cần icon khác tự mang (vd `lucide-react`) — cần xác nhận quy ước.

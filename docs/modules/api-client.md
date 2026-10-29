# @vyforge/api-client — gọi backend từ BFF Next.js

## 1. Mục đích

Chuẩn hoá cách app Next.js (vai trò BFF, ADR-0006) gọi backend Vyforge: client có kiểu sinh từ OpenAPI, gắn Bearer token
ở server (token không xuống trình duyệt), đọc Problem Details (RFC 9457) của platform-starter-web thành lỗi theo field,
và Server Action theo khuôn authorize → validate → gọi backend → lỗi theo field (B13). Kèm `EntityRef` cùng quy tắc với
platform-core để tham chiếu bản ghi giữa các module.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Package web | `@vyforge/api-client` 0.6.0 (platform-web) | không có README riêng; mô tả ở README gốc platform-web |
| Subpath | `.` | dùng ở file `server-only` |
| Nền | `openapi-fetch`, Standard Schema (Zod 4, Valibot, ArkType) | |

## 3. Tính năng chính

- **`createApiClient<paths>({ baseUrl, getAccessToken, headers })`**: client openapi-fetch có kiểu từ
  `openapi-typescript`, gắn Bearer token lấy ở server. `createApiFetch()` khi chưa sinh schema.
- **`unwrap(promise)`**: lấy `data` hoặc ném `ApiError` (kèm Problem Details nếu có).
- **Problem Details**: `isProblemDetail`, `ProblemDetail` (`code` ổn định để dịch, `params`, lỗi field),
  `toFormErrors(error, { translate })` → `FormErrors` (lỗi chung ở khoá `_form`), `applyFormErrors()` đẩy vào form client
  (vd `setError` của React Hook Form).
- **`createFormAction({ schema, authorize, handler, translate })`**: Server Action cho `useActionState`;
  `authorize` bắt buộc (action công khai ghi rõ `'public'`), giữ lại giá trị đã nhập khi lỗi (trừ field nhạy cảm);
  `redirect()`/`notFound()` được ném tiếp. Kèm `initialFormState`, `formDataToObject`, `issuesToFormErrors`.
- **`EntityRef`**: `{ type: 'crm:order', id }`, chuỗi chuẩn `crm:order/<id>`; `parseEntityRef`, `formatEntityRef`,
  `isEntityRef`, `entityRefOfKey` (khoá có ký tự lạ), `entityRefVrn` (`vrn:crm:order/<id>` cho `can()`/`Authz.check`),
  `isValidEntityType`, `isValidEntityId`. Ca mẫu dùng chung với Java (`__fixtures__/entity-ref-samples.json`).

## 4. Cách dùng nhanh

```bash
pnpm add @vyforge/api-client
pnpm dlx openapi-typescript http://localhost:8080/v3/api-docs -o src/shared/api/schema.d.ts   # sinh kiểu (cần xác nhận script của template)
```

```ts
// src/shared/api/client.server.ts
import 'server-only';
import { createApiClient } from '@vyforge/api-client';
import type { paths } from './schema';
export const api = () => createApiClient<paths>({ baseUrl: env().API_BASE_URL, getAccessToken });
```

```ts
// src/features/customers/api/actions.ts
'use server';
export const createCustomer = createFormAction({
  schema: z.object({ name: z.string().min(1), email: z.email() }),
  authorize: () => requireRole('SALES'),
  handler: async (input) => {
    const customer = await unwrap(api().POST('/api/v1/customers', { body: input }));
    redirect(`/customers/${customer.id}`);
  },
});
// client: const [state, action, pending] = useActionState(createCustomer, initialFormState);
```

## 5. Cấu hình quan trọng

| Tuỳ chọn | Ý nghĩa |
|---|---|
| `baseUrl` | URL backend/gateway, đọc từ env runtime ở server (B14) |
| `getAccessToken` | Lấy token ở server (vd `getAccessToken` của `@vyforge/auth`) |
| `headers` | Header thêm, vd `Accept-Language` |
| `translate` | Dịch theo `code` (vd `errorTranslator(t)` của `@vyforge/i18n`) |

## 6. Tích hợp

`@vyforge/auth` (token), `@vyforge/i18n` (`errorTranslator`), `@vyforge/admin-kit` (`applyProblem`, Server Action),
mọi `createXxxClient()` của package module dựng trên client này.

## 7. Triển khai

Thư viện; chỉ chạy ở server (BFF).

## 8. Trạng thái & còn mở

- Chưa có README trong package.

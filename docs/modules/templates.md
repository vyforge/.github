# template-spring-boot và template-nextjs — khung dự án và sub-template `resource`

## 1. Mục đích

Hai template Copier để khởi tạo dự án mới đúng chuẩn kiến trúc Vyforge trong vài phút: backend Spring Boot
(`<slug>-backend`) và frontend Next.js BFF (`<slug>-frontend`), đã nối sẵn platform-java / platform-web, auth, IAM,
tenant, CI trung lập. Sub-template `resource` sinh khung CRUD quản trị đầu-cuối của một entity (API admin ở backend +
trang `@vyforge/admin-kit` ở frontend) từ **cùng một file YAML**. Dự án đã tạo cập nhật theo template bằng
`copier update`. Chuẩn kiến trúc: `docs/ARCHITECTURE.md`.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Template backend | repo `template-spring-boot` (tag mới nhất v2.3.0) | Java 25, Spring Boot 4.1, Spring Modulith 2.1, PostgreSQL 18, Kafka, Redis/Valkey; `platform-parent` 0.14.0 |
| Template frontend | repo `template-nextjs` (tag mới nhất v2.3.0) | Next.js 16, React 19, TypeScript 6.0, Tailwind 4, next-intl, Better Auth, pnpm 11, Node 24; `@vyforge/ui` 0.5.0 |
| Sub-template | thư mục `resource/` của cả hai repo (`kind=resource`) | |
| Lệnh | `just resource <tên>` trong dự án đã sinh | |

## 3. Tính năng chính

**template-spring-boot** — câu hỏi: `project_name/slug`, `package_name` (`com.vyforge.<pkg>`), `ci_platform`
(github/forgejo/gitea/gitlab), `tier` (0 PoC, 1 dự án khách, 2 product), `use_kafka`, `use_redis`, `use_iam` +
`iam_mode` (`local`/`remote`), `use_rls` (tier 2), `use_scheduling`, `use_ratelimit`, `include_example`.
Có sẵn: modular monolith (`ModularityTests`), `TenantEntity` + `TenantIsolationTests` (ADR-0015), `@PersonalData`,
`@ClusterScheduled`, `@RateLimited`, outbox Modulith + `@Externalized` Kafka, resource server JWT, mã lỗi en/vi
(`ErrorCatalogTests`), Problem Details, `Idempotency-Key`, `openapi.json` + `OpenApiContractTests`, tier 2 hexagonal
(jMolecules) + consumer idempotent mẫu, Testcontainers, Spotless, SBOM, Dockerfile non-root, CI `vyforge-ci`
(`java-verify` → `image` → `gitops-bump`). Starter mới của 0.14 (workflow, search, AI, custom-fields…) **không bật sẵn** —
thêm dependency khi cần.

**template-nextjs** — câu hỏi: `project_name/slug`, `ci_platform`, `use_auth` (Better Auth + platform-auth-server, BFF),
`use_iam` (`can`/`requirePermission`, trang `/admin/iam`), `include_example` (feature `customers` gọi API mẫu của
backend). Có sẵn: App Router chỉ routing, code theo `features/<tên>` (`index.ts` client-safe, `server.ts`
server-only, ESLint chặn import sâu/ngược), `@vyforge/ui`, `api-client`, `i18n` (`localePrefix: 'always'`), loading/error/
not-found, Vitest + Playwright smoke, Dockerfile non-root, CI `vyforge-ci` (`node-verify` → `image` → `gitops-bump`).

**Sub-template `resource`** — YAML mô tả entity (chỉ dùng lúc sinh code, không có logic):

| Khoá YAML | Ý nghĩa |
|---|---|
| `resource`, `module`, `label` (vi/en) | Tên số ít kebab-case, module Modulith, nhãn |
| `pagination` | `offset` \| `cursor` (`large: true` → cursor) |
| `fields[]` | `name`, `type`, `max`/`min`, `required`, `unique`, `sort`, `filter` (`eq`/`like`/`in`/`range`), `search`, `readOnly`, `default`, `values` + `valueLabels` (enum), `sensitiveFilter`, `personalData`, `label` |
| `actions` | `List`, `Get`, `Create`, `Update`, `Delete` |
| `bulk` | vd `[delete]` |
| `ui.detailSections` | Nhóm field trên form/chi tiết (frontend) |

Kiểu field: `string`, `text`, `user`, `email`, `phone-vn` (email/phone có `@PersonalData`), `int`, `long`, `decimal`,
`money`, `bool`, `date`, `datetime`, `enum`, `ref` (UUID, không FK chéo module). **Chưa hỗ trợ**: `localized`,
`masterdata`, `attachments`, `extensible`, `number` (bỏ qua, có cảnh báo).

Backend sinh (trong `com.vyforge.<pkg>.<module>`): DTO, `Command`, `Filter`, `Service` (offset/keyset, optimistic lock,
event `Created/Updated/Deleted`, `AuditChanges`), `Actions` + catalog IAM, `Errors` (unique → 409 có field), entity
`TenantEntity`, repository + `Specification`, `<Resource>AdminController` ở `/api/v1/<service>/admin/<plural>`
(`GET/POST/PUT/DELETE`, `POST search`, `POST bulk/{op}`, `include=allowedActions`; có PII + IAM thì `POST {id}/reveal` +
action `RevealPersonalData`, ghi audit), migration `V<yyyyMMddHHmmss>__<module>_create_<table>.sql`, test module,
`TenantIsolationTests`, `AdminListContractTests`.

Frontend sinh: `src/features/<plural>/schemas.ts` (DTO, Zod, chuyển giờ VN ↔ `Instant`), `resource.ts`
(`defineAdminResource`), `api/` (`client`, `queries`, `actions` có kiểm quyền, `access`), form `ResourceForm` (lỗi
Problem Details về đúng ô, 409 → "Tải lại"), trang `src/app/[locale]/admin/<plural>/` (danh sách, chi tiết, `new`,
`[id]/edit`, `layout.tsx` bọc `NuqsAdapter`), ghép nhãn vào `messages/{vi,en}.json` (giữ bản dịch đã sửa), test Vitest
(không lộ PII, hợp đồng API, schema); thêm `@vyforge/admin-kit` 0.2 và `nuqs` vào `package.json`.

## 4. Cách dùng nhanh

```bash
pipx install copier                                   # >= 9.4
copier copy --trust gh:vyforge/template-spring-boot crm-backend
VYFORGE_NPM_REGISTRY=<registry> REGISTRY_TOKEN=<token> vyforge-ci npm-login    # một lần cho @vyforge/*
copier copy --trust gh:vyforge/template-nextjs crm-frontend
```

Sinh resource (cùng YAML cho hai repo):

```yaml
# resources/contract.yml
resource: contract
module: sales
label: { vi: Hợp đồng, en: Contracts }
fields:
  - { name: code, type: string, max: 30, required: true, unique: true, sort: true, filter: eq, search: true, readOnly: true }
  - { name: status, type: enum, values: [DRAFT, ACTIVE, EXPIRED], default: DRAFT, filter: in }
  - { name: value, type: money, min: 0, sort: true, filter: range }
  - { name: contactPhone, type: phone-vn, sensitiveFilter: true, filter: eq }
actions: [List, Get, Create, Update, Delete]
bulk: [delete]
ui: { detailSections: [[code, status], [value, contactPhone]] }
```

```bash
# backend
just resource contract && ./mvnw verify
# frontend
just resource contract && pnpm install && pnpm lint && pnpm typecheck && pnpm test
```

Cập nhật dự án theo template mới: `copier update --trust` (xung đột giải như merge). Chạy lại `just resource <tên>` là
`copier update` của riêng resource đó (cần cây git sạch).

## 5. Cấu hình quan trọng

| Biến / câu hỏi | Ý nghĩa |
|---|---|
| `tier` (backend) | 2 bật Kafka mặc định, hỏi RLS, module mẫu hexagonal |
| `iam_mode` | `local` (bảng `iam_*` trong app) / `remote` (platform-iam-server, token exchange, Redis xoá cache) |
| `use_rls` | App kết nối bằng role `<pkg>_app`, Flyway bằng role owner |
| `ci_platform` | Nơi sinh adapter CI mỏng gọi `vyforge-ci` (ADR-0014) |
| `VYFORGE_MAVEN_URL`, `REGISTRY_USER/TOKEN` | Registry chứa `com.vyforge.platform:*` (build backend) |
| `VYFORGE_NPM_REGISTRY`, `REGISTRY_TOKEN` | Registry `@vyforge/*` (frontend; không ghi vào repo) |

## 6. Tích hợp

- Backend dựa trên platform-java (`platform-parent`, starter), auth bằng platform-auth-server (compose có sẵn image),
  IAM local hoặc platform-iam-server.
- Frontend dựa trên platform-web (`@vyforge/ui`, `api-client`, `i18n`, `auth`, `iam`, `admin-kit`).
- Hợp đồng resource: backend theo `com.vyforge.platform.web.list` (≥ 0.13), frontend theo `@vyforge/admin-kit`.
- Zone admin của module platform dùng template khác (`templates/admin-zone` trong platform-admin), không phải
  template-nextjs.

## 7. Triển khai

Dự án sinh ra có Dockerfile, `compose.yaml` (backend), CI `vyforge-ci` theo `ci_platform`, `gitops-bump` vào
`<slug>-infra` (Argo CD). Chart dùng `charts/app` của platform-helm-charts (Actuator ở cổng riêng 8081 →
`managementPort`).

## 8. Trạng thái & còn mở

- Đã có: hai template v2.3.0, sub-template `resource` ở cả hai đầu dùng chung YAML, hiện PII có audit.
- Kiểu field chưa hỗ trợ: `localized`, `masterdata`, `attachments`, `extensible`, `number`. Đổi schema sau khi có dữ
  liệu phải viết migration tay (template không sinh `ALTER`).
- Workflow `test-template.yml` của hai repo đang tắt trong lúc chuyển sang `vyforge-ci`, chỉ chạy tay.
- Frontend: ESLint 9 (plugin Next chưa hỗ trợ 10), TypeScript 6.0 (typescript-eslint < 6.1).
- template-spring-boot pin `platform-parent` 0.14.0 trong khi platform-java đã 0.16.1 (Renovate sẽ nâng ở dự án; cần xác
  nhận có nâng template không).

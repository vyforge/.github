# CMS Content — nội dung website (headless CMS)

## 1. Mục đích

Cho mọi app một headless CMS dùng chung: trang, tin tức, banner, FAQ, thông báo, menu… có content type định nghĩa lúc
chạy, bản nháp/revision theo locale, duyệt và xuất bản (cả qua workflow), hẹn giờ, gỡ khẩn cấp, và delivery API có cache
nhiều tầng, SEO, sitemap, redirect cho site Next.js. Tự xây thay vì nhúng Payload. Thiết kế:
`docs/design/cms-content.md`.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Java starter | `com.vyforge.platform:platform-starter-cms-content` | platform-java 0.16.1; schema `cms` |
| Package web | `@vyforge/cms-content` 0.2.0 | client delivery (server), `/next` (route `[...slug]`, revalidate, draft mode, sitemap, robots), `/render` (`RichText`, `Blocks`, `JsonLd`), `/preview`, `/admin` (`EntryTable`, `EntryEditor`, `RichTextEditor` Tiptap 3) |
| Zone admin | `apps/cms-content-admin` (`/m/cms-content`) | |
| Phụ thuộc bắt buộc | PostgreSQL 15+ (`unaccent`, `pg_trgm`, ICU), `platform-starter-scheduling`, `platform-starter-audit` | thiếu một trong số đó thì module không bật |

## 3. Tính năng chính

- **Content type**: preset `page`, `post`, `banner`, `faq`, `static-page`, `menu`, `category`, `tag`, `author`,
  `notice`, `site-settings`; type khai bằng code (bean `ContentTypeDefinition`, chỉ đọc trên UI) hoặc tenant tạo bằng
  schema builder. 26 kiểu field, component, block, reference, media, SEO, đối tượng xem, khoảng hiển thị.
- **Schema có version bất biến**: đổi tên/xoá/đổi kiểu cần xác nhận (409 `cms.schema_migration_required`), dữ liệu cũ
  đọc qua upcaster, task `cms.migrate-entries`.
- **Phía ghi**: mỗi lần lưu là một revision (`AUTOSAVE`, `SAVE`, `RESTORE`, `PUBLISH`…), khoá mềm, `If-Match`, slug bỏ
  dấu tiếng Việt, so sánh/khôi phục revision.
- **Xuất bản theo locale**: `DRAFT → IN_REVIEW → APPROVED → SCHEDULED → PUBLISHED / CHANGED / UNPUBLISHED / ARCHIVED`;
  gate `none` | `simple-review` | `workflow:<định nghĩa>` | `PublicationGate` tuỳ biến; hẹn giờ; gỡ khẩn cấp
  (`Takedown`) có purge CDN đồng bộ và audit lý do.
- **Release** (`/api/v1/cms/releases`): gom tới 1.000 mục, `:lock`, `:publish`/`:schedule` trong một transaction,
  `:revert`, xem trước `perspective=release:<id>`.
- **Delivery** `/api/v1/cms/delivery/{site}/**` (public path): khoá đọc `cms_pk_…`, token preview `cms_pt.…`, JWT;
  perspective `published|draft|revision:<id>|release:<id>`; lọc/sắp xếp, tìm không dấu (FTS `vyforge.vi`), include
  reference theo tầng, `hreflang`, SEO, JSON-LD, sync API, sitemap, robots; `ETag`/304, `stale-while-revalidate`,
  `Cache-Tag`; cache Valkey/Caffeine.
- **Mẫu** (`/api/v1/cms/templates`): `entry`, `seo-title`, `structured-data`, có version.
- **Xoá cache**: task `cms.invalidate` gọi CDN (`cloudflare`, `fastly`, `bunny`, `varnish`, `nginx` hoặc `CdnPurger`)
  và route revalidate Next.js (HMAC + timestamp).
- **HTML thô** (`rawHtml`, tắt mặc định): qua OWASP HTML Sanitizer, `iframe` chỉ tới YouTube/Vimeo/Maps/Facebook/TikTok;
  cần `cms:entry:EditRawHtml`.
- **In PDF** (khi có platform-report): `EntryPrints.print(...)` theo template `cms.entry.<typeKey>` của tenant (không có
  REST, không có template mặc định).
- **SPI** (`com.vyforge.platform.cms.spi`): `FieldTypeProvider`, `FieldValidator`, `ContentLifecycleInterceptor`,
  `PublicationGate`, `MediaResolver`, `CdnPurger`, `RevalidateNotifier`, `SlugGenerator`, `AudienceResolver`,
  `DeliveryTransformer`, `PreviewUrlBuilder`, `CmsQuotaProvider`, `DomainVerifier`.
- **Quan sát**: metric `vyforge.cms.delivery.requests`, `.entries.saved`, `.publish`, `.schedule.lag`,
  `.invalidation`; health `cmsSchedule`.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-cms-content</artifactId>
</dependency>
```

```yaml
spring.flyway.locations: classpath:db/migration,classpath:META-INF/vyforge/db/scheduling
vyforge:
  crypto:                         # SecretCipher: khoá ký preview, token CDN, secret revalidate
    keys.k2026a: ${CRYPTO_KEY_2026A}
    active-key: k2026a
```

```java
@Bean
ContentTypeDefinition branchType() {
    return ContentTypeDefinition.collection("branch", "Chi nhánh", "Branch")
            .field(FieldDefinition.builder("name", FieldType.TEXT).required().localized().build())
            .field(FieldDefinition.builder("province", FieldType.TEXT).filterable().build())
            .settings(s -> s.withTitleField("name"))
            .build();
}

Optional<DeliveryEntry> notice = delivery.singleton(DeliveryContext.published("default", "vi"), "notice");
```

Site Next.js: `createCmsClient()` + `resolveCmsPage()` cho route `[...slug]`, `createRevalidateRoute()` để CMS báo
revalidate.

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.cms.enabled` | `true` | Tắt toàn bộ module |
| `vyforge.cms.mode` | `embedded` | `services` (chạy trong platform-services) hoặc `remote` (app chỉ đọc qua HTTP) |
| `vyforge.cms.presets` | rỗng = tất cả | Preset content type được bật |
| `vyforge.cms.schema.migrate` | `true` | Flyway riêng (`flyway_schema_history_cms`); tắt thì chạy `META-INF/vyforge/sql/cms.sql` |
| `vyforge.cms.admin.api-enabled` | `true` | Tắt ở replica chỉ đọc |
| `vyforge.cms.delivery.base-path` | `/api/v1/cms/delivery` | Tự thêm vào `vyforge.security.public-paths` |
| `vyforge.cms.delivery.public-endpoints` | `true` | Tự mở public path |
| `vyforge.cms.delivery.anonymous-by-host` | `false` | Đọc ẩn danh theo domain đã xác minh |
| `vyforge.cms.delivery.max-include` / `max-page-size` | 3 / 100 | |
| `vyforge.cms.delivery.cache.ttl` / `store` | `10m` / `auto` | `auto` = Valkey nếu có, không thì Caffeine |
| `vyforge.cms.delivery.http.s-maxage` / `stale-while-revalidate` | `300s` / `1d` | Header cache |
| `vyforge.cms.delivery.rate-limit.per-key` | 50 | Request/giây mỗi khoá đọc |
| `vyforge.cms.raw-html.allowed` | `false` | Cho field `rawHtml` |
| `vyforge.cms.events.externalize` | `false` | Event ra Kafka topic `vyforge.cms` |
| `vyforge.cms.remote.url` | — | Server chạy module (thường `http://platform-services`), registration mặc định `vyforge-cms` |
| `vyforge.cms.integrations.workflow.enabled` | `true` | Gate `workflow` |

## 6. Tích hợp với module khác

| Module | Tích hợp |
|---|---|
| workflow | Gate `workflow:<định nghĩa>` (mặc định `cms-publish`): gửi duyệt tạo hồ sơ subject `vrn:cms:entry/<id>`; `APPROVED` → duyệt/publish, `REJECTED` → về nháp |
| files | Field `media`, gắn file vào `cms:entry` |
| webhooks | `cms.entry.published`, `cms.entry.unpublished`, `cms.entry.taken_down`, `cms.release.published` |
| platform-report | `EntryPrints` (PDF) |
| forms | Block `form` trên trang gửi tới API công khai của forms (theo thiết kế) |
| audit (bắt buộc), scheduling (bắt buộc) | Audit lý do gỡ, task hẹn giờ/invalidate |
| Valkey/Redis, Kafka | Cache delivery dùng chung; event ra ngoài |

## 7. Chế độ triển khai, IAM, sự kiện

- **Chế độ**: `embedded` (mặc định), `services` (chạy trong platform-services), `remote` (app không có bảng `cms_*`,
  chỉ `ContentDelivery` gọi `POST /api/v1/cms/internal/delivery/**` của server bằng client credentials scope
  `cms.internal`; ghi qua admin MFE của server; không cache phía app).
- **IAM** (resource `vrn:cms:entry/<type>/<id>`): `cms:content-type:List|Get|Create|Update|Enable|Disable`,
  `cms:entry:List|Get|Create|Update|Delete|Submit|Approve|Reject|Publish|Unpublish|Schedule|CancelSchedule|Archive|Unarchive|Restore|Preview|EditInReview|EditRawHtml|Takedown|ClearTakedown`,
  `cms:release:List|Get|Create|Update|Delete|Lock|Unlock|Publish|Schedule|Revert`,
  `cms:template:List|Get|Create|Update|Publish|Archive`, `cms:redirect:*`, `cms:site:*` (gồm
  `AddDomain|VerifyDomain|RemoveDomain`), `cms:locale:Manage|Update`, `cms:delivery-key:Create|Revoke|Manage`,
  `cms:settings:Get|Update`, `cms:task:Manage`, `cms:import:Run`. Admin MFE: `GET /api/v1/cms/admin/info`.
- **Sự kiện** (`CmsEvent`): `ContentEntryCreated`, `ContentDraftSaved`, `ContentSubmittedForReview`,
  `ContentReviewWithdrawn`, `ContentApproved`, `ContentRejected`, `ContentScheduled`, `ContentScheduleSkipped`,
  `ContentPublished`, `ContentUnpublished`, `ContentTakenDown`, `ContentEntryArchived`, `ContentEntryDeleted`,
  `ContentTypeChanged`, `ContentCacheInvalidated`, `ReleasePublished`, `ReleaseFailed`, `ReleaseReverted`.
- **Giấy phép**: module `licensed` (`cms-content`, từ 0.14.0).

## 8. Trạng thái & còn mở

- Đã có: type động + preset, revision, xuất bản theo locale, gate workflow, release, delivery có cache/CDN, mẫu, HTML
  thô có sanitizer, in PDF, chế độ remote đọc, zone admin.
- Chưa có (README module): live preview trong iframe, nhập WordPress WXR, gate duyệt ở cấp release, kiểu `vn-address`
  (cần master-data), tích hợp search/realtime/notification.

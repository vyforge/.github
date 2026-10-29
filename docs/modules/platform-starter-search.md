# platform-starter-search — tìm kiếm toàn văn tiếng Việt (+ OpenSearch, Meilisearch)

## 1. Mục đích

Ứng dụng nghiệp vụ cần ô tìm kiếm "gõ gì cũng ra": có dấu, không dấu, Telex/VNI gõ dở, sai chính tả, số điện thoại, mã
đơn; kèm facet, lọc, sắp xếp, highlight, gợi ý và tìm nhiều index một lúc (⌘K). Search làm việc đó với lọc theo tenant và
**theo quyền ngay trong truy vấn**, đánh chỉ mục bất đồng bộ không mất thay đổi, reindex blue/green không ngừng tìm kiếm.
Index chỉ là bản chiếu: dữ liệu gốc ở bảng nghiệp vụ, dựng lại được từ `SearchDocumentSource` của app. Engine mặc định là
PostgreSQL có sẵn; OpenSearch và Meilisearch là artifact tuỳ chọn.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Starter | `platform-starter-search` | Từ 0.14.0; có phí (`licensed: true`); engine `postgres` có sẵn |
| Engine tuỳ chọn | `platform-search-opensearch` | OpenSearch 3.x (Apache-2.0), qua REST, routing theo tenant |
| Engine tuỳ chọn | `platform-search-meilisearch` | Meilisearch 1.5x, chỉ lõi MIT (không dùng phần Enterprise BUSL-1.1) |
| Test-jar | `platform-starter-search:tests` | `SearchEngineContractTests` cho adapter mới |
| Package web | `@vyforge/search` 0.1.0 | Client server-only, đồng bộ URL, route gợi ý BFF, `SearchBox`, `FacetPanel`, `ResultList`, `Highlight`, `CommandPalette`, component quản trị |
| Zone admin | `apps/search-admin` 0.1.0 (`/m/search`) | |
| Service | `platform-services` | Bật module `search`; app khác dùng `remote` |

Thiết kế: [docs/design/search.md](../design/search.md).

## 3. Tính năng chính

- **Khai báo index**: `SearchIndexDefinition.builder(...)` (field `NAME`, `PHONE`, `IDENTIFIER`, `KEYWORD`, `DATE`…, trọng
  số, `suggest`, `facetable`, `sortable`, `personalData`, `restricted(action)`, `rankSignal`), hoặc YAML
  `META-INF/vyforge/search/indexes/*.yml`.
- **Đồng bộ**: `SearchIndexer.index(...)` chỉ ghi `search_change` trong transaction, flush bất đồng bộ (queue `search`);
  entity JPA đơn giản dùng `@Searchable(index, mapper)` (listener Hibernate). Ingest qua Kafka (`vyforge` hoặc
  `debezium-outbox`).
- **Tiếng Việt** (`VietnameseAnalyzer`, chạy ở Java cho mọi engine): NFC, chuẩn dấu thanh, bỏ dấu, shingle, mã/số điện
  thoại/email, đảo họ tên; Telex/VNI, viết tắt hành chính, synonym, sửa lỗi gõ (`correctedQuery`).
- **Quyền 4 lớp**: tenant (engine luôn lọc); index (`search:index:Query` + `requires`, thiếu → 404); bản ghi (`ACL` token
  `u:`, `g:`, `r:`, `t:*`; `POLICY` biên dịch policy IAM thành filter); field (`restricted`: không trả, không highlight,
  không tìm được).
- **Truy vấn**: filter, facet, sort, highlight, gợi ý, tìm nhiều index (`/multi`), ghi click; vector và hybrid (RRF) khi có
  `EmbeddingProvider`.
- **Reindex blue/green** theo generation, kiểm số tài liệu sau dựng, tự dựng lại khi đổi định nghĩa.
- **Quản trị**: synonym, rule, thống kê truy vấn, xem/giải thích document, thay đổi lỗi, engine riêng của tenant.
- REST `/api/v1/search`: `indexes/{index}/query` (POST, GET), `indexes/{index}/suggest`, `multi`, `clicks`, `admin/**`.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-search</artifactId>
</dependency>
<!-- tuỳ chọn -->
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-search-opensearch</artifactId>
</dependency>
```

```bash
pnpm add @vyforge/search
```

```java
@Bean
SearchIndexDefinition customersIndex() {
    return SearchIndexDefinition.builder("customers")
            .field("fullName", FieldType.NAME, f -> f.searchable(Weight.A).suggest().stored().personalData(BASIC))
            .field("phone", FieldType.PHONE, f -> f.searchable(Weight.A).stored().personalData(BASIC))
            .field("province", FieldType.KEYWORD, f -> f.facetable().stored())
            .access(AccessMode.POLICY, a -> a.action("crm:customer:Get").attributes("branchId"))
            .requires("crm:customer:List")
            .build();
}

@Transactional
void rename(UUID id, String name) {
    Customer c = customers.rename(id, name);
    indexer.index("customers", id.toString(), c.getVersion());
}

SearchResult r = search.search(SearchRequest.of("customers")
        .q("nguyen van an")
        .filter(Filter.eq("province", "Hà Nội"))
        .facets(Facet.terms("province"))
        .page(0, 20));
```

App còn cài bean `SearchDocumentSource` (`load`, `scan`, `count`) để reindex.

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.search.mode` | `embedded` | `remote`: gọi `platform-services` |
| `vyforge.search.default-engine` | `postgres` | `opensearch`, `meilisearch` khi có artifact |
| `vyforge.search.engines.postgres.statement-timeout` | `2s` | |
| `vyforge.search.indexing.flush-window` / `batch-size` | `2s` / `500` | |
| `vyforge.search.reindex.on-schema-change` | `auto` | Tự dựng generation mới khi đổi định nghĩa |
| `vyforge.search.query.timeout` / `max-size` / `max-offset` | `2s` / `100` / `10000` | |
| `vyforge.search.query-log.enabled` / `retention` | `true` / `90d` | |
| `vyforge.search.ingest.enabled` / `topic` | `false` / `search-ingest` | Nhận document qua Kafka |
| `vyforge.search.events.externalize` / `topic` | `false` / `vyforge.search` | |
| `vyforge.search.hashing-key` | dẫn xuất từ `vyforge.crypto` | |
| `vyforge.search.privacy.enabled` | `true` | Contributor privacy-dsr |
| `vyforge.search.engines.opensearch.*` | — | `enabled`, `uris`, `username`, `password`, `index-prefix` |
| `vyforge.search.engines.meilisearch.*` | — | `enabled`, `url`, `api-key` (key giới hạn theo prefix, không dùng master key), `index-prefix` |

## 6. Tích hợp với module khác

Phụ thuộc tuỳ chọn: authz, security, audit, ratelimit.

- **authz**: bắt buộc cho `POLICY` và API quản trị; không có thì chỉ `TENANT` và `ACL`.
- **security**: `TokenExchangeClient` cho chế độ remote (đổi token theo `remote.audience`).
- **organization**: token `o:<unitId>` cho tài liệu chia theo đơn vị.
- **ai**: cài `EmbeddingProvider`, rerank cross-encoder.
- **privacy-dsr**: `SearchPersonalData` (khoá `search`); `SearchErasure.eraseForSubject` xoá document, thay đổi chờ, log
  truy vấn của chủ thể.
- **ratelimit**: gợi ý 30 truy vấn/phút/người cho `/query`, 120/phút cho `/suggest`.
- **platform-core**: `VietnameseText` (cùng kết quả với hàm SQL `vyforge.fold`).
- SPI: `SearchEngine`, `SearchDocumentSource`, `PrincipalTokensResolver`, `SynonymSource`, `EmbeddingProvider`,
  `WordSegmenter`, `RankingFunction`.

## 7. Chế độ triển khai, quyền, sự kiện

- **embedded** hoặc **remote**: `SearchService` gọi server bằng token người dùng (server kiểm quyền); `SearchIndexer` gửi
  document qua Kafka sau commit. App remote không có bảng `search_*`, không việc nền; không dùng được `indexNow` và index
  `SYNC`. `SearchContractTests` chạy cùng kịch bản ở hai chế độ.
- Engine riêng của tenant: tạo qua `POST /api/v1/search/admin/engines`.
- **Quyền IAM**: `search:index:List|Get|Query|Reindex`, `search:document:Inspect|InspectSensitive`,
  `search:synonym:List|Create|Update|Delete`, `search:rule:List|Create|Update|Delete`, `search:analytics:Get`,
  `search:engine:List|Update`, `search:subject:Erase`.
- **Sự kiện** (`SearchEvent`): `SearchGenerationStarted`, `SearchGenerationActivated`, `SearchGenerationFailed`,
  `SearchIndexDegraded`, `SearchIndexRecovered`, `SearchDocumentRejected`.
- Metric `vyforge.search.*` (không tag tenant); job `search-sweep`, `search-metrics`, `search-cleanup`, `search-query-log`.

## 8. Trạng thái & còn mở

- **Đã làm**: 0.14.0 bản đầu với ba engine; 0.15.0 sửa generation FAILED khi tài liệu lỗi được sửa trong lúc kiểm; 0.16.0
  chế độ remote, contributor privacy-dsr, dùng `VietnameseText`.
- **Giới hạn hiện tại** (README):
  - remote: chưa có `PUT /internal/catalogs/{service}`, lệnh reindex qua topic `search-control`, đồng bộ `@Searchable`;
    gửi envelope sau commit không qua outbox (app chết giữa commit và gửi thì mất thay đổi tới lần ghi sau/reindex).
  - postgres: chưa có index sort riêng theo field, chưa có BM25 (`pg_textsearch`); field `IDENTIFIER` vẫn khớp tiền tố khi
    là từ cuối câu.
  - OpenSearch: chưa có vector (hybrid lùi về keyword). Meilisearch: không có cursor, không ghi có điều kiện theo version,
    không dùng synonym/Telex, không nhân tín hiệu xếp hạng.
  - Chưa có adapter tách từ (`WordSegmenter`), `RankingFunction`.

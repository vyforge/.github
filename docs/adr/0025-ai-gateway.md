# ADR-0025: Module AI là gateway có use case, cổng dữ liệu và ngân sách, adapter Spring AI 2.0

- **Trạng thái:** Chấp nhận — 27/10/2026. Thiết kế chi tiết: [ai.md](../design/ai.md). Liên quan
  [ADR-0009](0009-iam-authorization.md), [ADR-0015](0015-tenant-data-isolation-platform-core.md),
  [ADR-0020](0020-shared-application-modules.md) (mục privacy-dsr, search).
- **Phát hành:** `platform-starter-ai` (platform-java 0.14.0), `@vyforge/ai` 0.1.0 (platform-web), admin MFE `/m/ai`
  (platform-admin).

## Bối cảnh

Nhiều module và app muốn gọi LLM: soạn thư, tóm tắt, trích xuất hoá đơn/CCCD, hỏi đáp trên tài liệu (RAG), agent gọi
tool. Gọi thẳng provider từ từng app thì không ai kiểm được dữ liệu cá nhân nào ra nước ngoài (Luật 91/2025 Điều 20),
chi phí, rủi ro theo Luật Trí tuệ nhân tạo 134/2025 (phân loại, minh bạch, báo cáo sự cố theo NĐ 142/2026), prompt
injection (OWASP LLM01) và rò dữ liệu qua RAG (LLM08). MODULE_PLAN ban đầu xếp module AI ở P2; người dùng đã chọn xây
sẵn mọi module P1/P2 trong kế hoạch.

## Quyết định

1. **Mọi lời gọi đi qua một use case đã khai báo** (`AiUseCase`): model alias, prompt, mức dữ liệu tối đa, mức rủi ro,
   chính sách log, ngân sách, tool được dùng. Không có lời gọi "chui".
2. **API công khai là của Vyforge** (`AiGateway`, `KnowledgeBases`, `AiExtraction`, `AiAgents`); adapter provider dùng
   **Spring AI 2.0.2** (openai, openai-compatible, anthropic, ollama qua SPI `ProviderFactory`), không lộ kiểu Spring AI.
   Vòng tool do module tự chạy để mọi tool đi qua `ToolPolicy`.
3. **Model alias + danh sách deployment** (PRIORITY/WEIGHTED/LOWEST_COST), fallback theo loại lỗi, cooldown, alias dự
   phòng khi vượt context; stream fallback được trước token đầu.
4. **Cổng dữ liệu trước router:** request có `DataClass` (PUBLIC < INTERNAL < PERSONAL < SENSITIVE); deployment có
   `location` (ON_PREM, VN, FOREIGN) và `maxDataClass`; chính sách xuyên biên giới DENY / REDACTED_ONLY / ALLOW kèm
   `ctiaRef`. Mặc định `DENY`, và dữ liệu mặc định của lời gọi bằng trần dữ liệu của use case.
5. **Che PII hai chiều** (pseudonymize → khôi phục, cả khi stream): detector regex VN + gợi ý từ app (khớp không dấu),
   chuẩn hoá TCVN3/VNI; adapter Presidio/LLM cục bộ là tuỳ chọn.
6. **Log mặc định chỉ metadata**; nội dung chỉ lưu khi bật `capture`, mã hoá bằng `SecretCipher`, có hạn.
7. **Ngân sách giữ chỗ trước, quyết toán sau**, trừ nguyên tử bằng `UPDATE` có điều kiện trên PostgreSQL; trần mỗi lời
   gọi, ngưỡng cảnh báo.
8. **Prompt có version bất biến + nhãn**, template Liquid (Liqp, cùng engine với notification), eval gate chặn chuyển
   nhãn `production` khi điểm dưới ngưỡng.
9. **RAG trên PostgreSQL + pgvector**: một bảng chunk, cột `vector` không cố định chiều, index HNSW từng phần theo
   embedding space; hybrid vector + FTS `unaccent` gộp RRF; lọc trước bằng `access_tags`, kiểm sau bằng
   `DocumentAccessVerifier`; trích dẫn được server kiểm.
10. **Tool calling an toàn:** `@AiTool` khai action IAM và mức rủi ro; tool chạy dưới danh tính người khởi chạy;
    DESTRUCTIVE luôn cần duyệt; ngữ cảnh đã nhiễm nội dung không tin cậy thì tool có egress phải duyệt (taint).
11. **Việc dài chạy trên `DurableTaskQueue`** (queue `ai`: ingest, reindex, eval, extract, agent); chat đồng bộ, stream SSE.
12. **Minh bạch theo Luật 134:** response mang `aiGenerated`, model, use case; sổ use case xuất CSV; sự cố có hạn 72 giờ /
    05 ngày làm việc.

## Phương án đã loại

| Phương án | Lý do loại |
|---|---|
| LangChain4j làm nền | Tương đương về tính năng; Spring AI khớp Boot 4.1, Jackson 3 và observation `gen_ai.*` sẵn có của platform |
| Chạy LiteLLM proxy làm gateway | Thêm một service Python phải trực; thư mục `enterprise/` license riêng; không gắn được tenant + IAM. Chỉ học mô hình `model_list`, cooldown, budget |
| Langfuse tự host cho prompt/trace | Thêm service và DB; chỉ học mô hình prompt version + nhãn |
| Dify | Multi-tenant cần giấy phép riêng bằng văn bản |
| Vector DB riêng (Qdrant, Milvus…) | PostgreSQL + pgvector đủ cho quy mô tier 0–2, cùng transaction và cùng quy ước tenant |
| Nhúng OCR/NER (JNI, Python) trong JVM | Engine OCR chạy sidecar sau SPI `DocumentExtractor` |
| Mặc định `REDACTED_ONLY` cho provider nước ngoài | Regex không bắt hết tên, địa chỉ tiếng Việt; vẫn là chuyển dữ liệu cá nhân xuyên biên giới |
| Parser SSE `eventsource-parser` ở web | Tự viết parser nhỏ để không thêm dependency |

## Hệ quả

- Cần extension `vector` và `unaccent`; migration thử `CREATE EXTENSION`, không có quyền thì báo lỗi rõ.
- Cooldown, cache và giới hạn đồng thời đang trong bộ nhớ từng instance; bộ đếm ngân sách chỉ dùng JDBC. Nhiều replica
  thì cooldown không chia sẻ (chấp nhận được tới khi có Valkey).
- `JdbcClient`/`TransactionTemplate` của module gói trong bean `AiDb` để không chiếm bean của app.
- REST app chỉ bật khi có Spring Security; API quản trị chỉ bật khi có `-authz`.

## Lệch so với thiết kế

- Thêm cột/bảng: `ai_budget_counter.exhausted`, bảng `ai_budget_reservation`, `ai_document.content`,
  `ai_extraction.source_text/data_class/error_code`, `ai_run.roles/transcript`, `ai_run_step.tool_call_id`,
  `ai_embedding_space.index_ddl`; mọi bảng partition có partition `default` + job chuyển dòng.
- Mã số thuế để `KEEP`, không nâng mức dữ liệu của lời gọi.
- Cấu hình platform qua `?scope=platform` + quyền trên `vrn:ai:<loại>/platform`.

## Còn mở

- `mode: remote` và contract test hai chế độ; Valkey cho cooldown và bộ đếm giữa replica.
- Sidecar OCR dựng sẵn (docling-serve, PaddleOCR, Tesseract), vision, MCP client, rerank, đồng bộ theo event của
  `-files`.
- PrometheusRule trong `platform-observability`.

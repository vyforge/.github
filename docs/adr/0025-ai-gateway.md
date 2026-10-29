# ADR-0025: Module AI là gateway có use case, cổng dữ liệu và ngân sách, adapter Spring AI 2.0

- **Trạng thái:** Chấp nhận — 27/10/2026. Thiết kế chi tiết: [ai.md](../design/ai.md). Liên quan
  [ADR-0009](0009-iam-authorization.md), [ADR-0015](0015-tenant-data-isolation-platform-core.md),
  [ADR-0020](0020-shared-application-modules.md) (mục privacy-dsr, search).
- **Phát hành:** `platform-starter-ai` (platform-java 0.14.0; follow-up: remote, Valkey, OCR sidecar, rerank, MCP, đồng bộ
  files), `@vyforge/ai` 0.1.0 (platform-web), admin MFE `/m/ai`
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

### Bổ sung đợt follow-up (29/10/2026)

13. **Chế độ `remote` cùng interface:** `vyforge.ai.mode=remote` cho app `AiGateway` và `KnowledgeBases` gọi
    `/api/v1/ai/internal/**` của server bằng token client credentials (scope `ai.internal` hoặc `services.internal`,
    tenant theo `X-Tenant-Id`, danh sách tenant theo `azp`), như sequence. App chuyển tiếp người dùng hiện tại (subject,
    role, group); server dựng lại principal đó nên quyền `ai:*`, ngân sách theo người dùng và `ai_call.sub` đúng người
    dùng. Một bộ contract test chạy cùng kịch bản trên bản nhúng và bản remote. `AiAgents`, `AiExtraction`, eval chỉ ở
    server.
14. **Bộ đếm ngân sách trên Valkey** khi app có Valkey/Redis (`budget.counter-store: valkey`, mặc định): giữ chỗ và quyết
    toán là script Lua nguyên tử trên mọi ngân sách của lời gọi; DB vẫn là bản chuẩn, nhận phần đã dùng qua ghi bù mỗi
    10 giây, khoá thiếu được nạp lại từ DB. Valkey mất: `fail-mode: closed` (mặc định) từ chối với
    `ai.storage_unavailable`, `open` giữ chỗ trên DB.
15. **OCR qua sidecar HTTP:** adapter docling-serve (`/v1/convert/file`, Markdown), PaddleOCR 3.x serving (`/ocr`),
    tesseract-server (`/tesseract`); chuỗi chọn engine đọc lớp text (docling, Tika) trước, PDF scan và ảnh sang OCR
    (`ocr-default` thử trước), engine mất thì thử engine kế. SPI `DocumentExtractor` thêm `ocr()`.
16. **Rerank qua gateway:** `AiGateway.rerank` và SPI `RerankModel` (`openai-compatible` gọi API `/rerank` chung của vLLM,
    Infinity, Jina, Cohere v2); đi qua cổng dữ liệu, ngân sách, nhật ký và fallback như embedding. KB khai báo
    `retrieval.rerank`; reranker lỗi thì giữ thứ tự RRF.
17. **MCP client tự viết (Streamable HTTP, JSON-RPC)** thay vì kéo MCP Java SDK: chỉ cần `initialize`, `tools/list`,
    `tools/call`; không theo redirect; host phải nằm trong `mcp.client.allowed-hosts`. Tool MCP mặc định `WRITE` +
    `ALWAYS` + `egress` + taint; quản trị phân loại lại (riêng tool MCP được nới) bằng `ai_tool_setting`.
18. **Đồng bộ files theo event:** KB khai báo `sync.files.bucket`; `FileAvailable` → ingest version mới,
    `FileTrashed`/`FilePurged` → xoá, chạy sau commit của files. Quyền từng file vẫn kiểm lúc truy xuất
    (`files:file:Download`), nên `FileAttached` không cần đổi `access_tags`. `platform-starter-files` là phụ thuộc
    `optional`.

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
| MCP Java SDK / `spring-ai-mcp` cho MCP client | Chỉ dùng 3 method; SDK kéo thêm phụ thuộc và kiểu riêng vào agent. Client tự viết ~200 dòng, test bằng server giả |
| Token exchange RFC 8693 ở chế độ remote ngay từ đầu | Cần IdP hỗ trợ và cấu hình theo app; bản đầu tin danh tính do service có scope nội bộ chuyển tiếp (giống mô hình sequence) |
| Chỉ Valkey cho ngân sách (bỏ DB) | Mất số khi Valkey khởi động lại; DB giữ bản chuẩn và báo cáo |

## Hệ quả

- Cần extension `vector` và `unaccent`; migration thử `CREATE EXTENSION`, không có quyền thì báo lỗi rõ.
- Cooldown, cache và giới hạn đồng thời đang trong bộ nhớ từng instance (bộ đếm ngân sách đã lên Valkey). Nhiều replica
  thì cooldown không chia sẻ.
- Valkey: số đã dùng trong DB chậm tối đa một `flush-interval`; khoá hết hạn đúng lúc chưa ghi bù thì có thể đếm thiếu
  trong khoảng đó. Ngưỡng cảnh báo tính theo số trên Valkey.
- Chế độ remote tin danh tính người dùng do app có scope `ai.internal` chuyển tiếp; giới hạn bằng
  `internal-api.clients.<azp>.tenants` và gateway chặn `/internal/**` từ ngoài.
- `JdbcClient`/`TransactionTemplate` của module gói trong bean `AiDb` để không chiếm bean của app.
- REST app chỉ bật khi có Spring Security; API quản trị chỉ bật khi có `-authz`.

## Lệch so với thiết kế

- Thêm cột/bảng: `ai_budget_counter.exhausted`, bảng `ai_budget_reservation`, `ai_document.content`,
  `ai_extraction.source_text/data_class/error_code`, `ai_run.roles/transcript`, `ai_run_step.tool_call_id`,
  `ai_embedding_space.index_ddl`; mọi bảng partition có partition `default` + job chuyển dòng.
- Mã số thuế để `KEEP`, không nâng mức dữ liệu của lời gọi.
- Cấu hình platform qua `?scope=platform` + quyền trên `vrn:ai:<loại>/platform`.

- Alert `AiAllDeploymentsCoolingDown` tính theo service (gauge cooldown không có nhãn alias); `AiProviderErrorRate` theo
  service vì lời gọi lỗi hết deployment không có model. Thêm gauge `vyforge.ai.approvals.pending.oldest`,
  `vyforge.ai.incidents.report.due` và histogram cho `vyforge.ai.ttft` để alert p95.

## Còn mở

- Cooldown và cache response trên Valkey; token exchange RFC 8693 cho chế độ remote; `AiAgents`/`AiExtraction` ở chế
  độ remote (tool chạy trong app); OCR bằng model vision.
- Sidecar OCR mới được kiểm bằng server giả theo hợp đồng HTTP; cần đo với image thật trên tài liệu tiếng Việt.
- PrometheusRule (`platform-observability`, `aiAlerts`) đã có; dashboard Grafana cho ai chưa có.

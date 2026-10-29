# AI — cổng AI dùng chung (LLM, RAG, agent, trích xuất)

## 1. Mục đích

Cho mọi app một cổng AI có kiểm soát: mọi lời gọi LLM đi qua use case đã khai báo, có quyền IAM, che dữ liệu cá nhân
(detector Việt Nam), chặn đưa dữ liệu ra nước ngoài theo Luật 91/2025, ngân sách theo tenant/người dùng, nhật ký và báo
cáo sự cố theo luật AI. Trên đó có RAG (kho tri thức pgvector), agent gọi tool dưới danh tính người dùng, và trích xuất
dữ liệu từ hoá đơn/CCCD/giấy phép kinh doanh. Spring AI 2.0.2 chỉ là adapter provider, app không thấy kiểu của Spring AI.
Thiết kế: `docs/design/ai.md`, ADR-0025.

## 2. Thành phần

| Loại | Tên | Ghi chú |
|---|---|---|
| Java starter | `com.vyforge.platform:platform-starter-ai` | platform-java 0.16.1; thêm provider Spring AI cần dùng (vd `spring-ai-openai` cho OpenAI/vLLM/LiteLLM) |
| Package web | `@vyforge/ai` 0.1.0 | client server-only + route handler SSE, `/stream`, `/react` (`useAiChat`), `/ui` (`AiChat`, `CitationList`, `AiDisclosure`…), `/admin` |
| Zone admin | `apps/ai-admin` (`/m/ai`) | usage, sổ đăng ký use case, chính sách dữ liệu, provider, ngân sách, prompt, KB, sự cố |
| Phụ thuộc bắt buộc | PostgreSQL có `vector` (pgvector ≥ 0.8) và `unaccent`, `platform-starter-scheduling` (queue `ai`) | |
| Tuỳ chọn | Valkey/Redis (ngân sách nguyên tử), OCR sidecar (docling, paddleocr, tesseract) | |

## 3. Tính năng chính

- **`AiGateway`**: `call`/stream SSE/structured output; pipeline: quyền `ai:usecase:Invoke` → prompt có version → che
  PII → guardrail → giữ chỗ ngân sách → cache → router (cổng dữ liệu, fallback, cooldown) → kiểm output JSON Schema (thử
  lại 2 lần) → khôi phục PII → nhật ký `ai_call`.
- **Cổng dữ liệu**: deployment khai `location` (`ON_PREM`, `VN`, `FOREIGN`) và `max-data-class`; dữ liệu `PERSONAL` trở
  lên không tới `FOREIGN` khi `crossBorder = DENY` (mặc định); `REDACTED_ONLY`; `ALLOW` bắt buộc số hồ sơ đánh giá tác
  động (`ctiaRef`).
- **Che PII**: email, SĐT, CCCD (mã tỉnh), CMND/MST/STK/hộ chiếu theo ngữ cảnh, thẻ (Luhn), biển số + `PiiHints` của app;
  pseudonymize `[PHONE_1]` và khôi phục cả trong stream; luật `PSEUDONYMIZE|MASK|DROP|BLOCK|BLOCK_FOREIGN|KEEP`.
- **Ngân sách** (`ai_budget`): theo tenant/người dùng/use case/agent/API key, kỳ ngày/tháng/tổng, `HARD`/`SOFT`, giữ chỗ
  trước khi gọi rồi quyết toán theo usage thật; Valkey (Lua) khi có.
- **Prompt & eval**: prompt YAML (`META-INF/vyforge/ai/prompts`) thành version bất biến, nhãn
  `production`/`latest`, render Liquid giới hạn thời gian; eval gate; evaluator `exact`, `contains`, `regex`,
  `json_schema`, `llm_judge`, `retrieval_hit_at_k`; phản hồi 👍/👎.
- **`KnowledgeBases`** (RAG): ingest nền, chunk `STRUCTURE`/`TOKEN`/`LEGAL_VN`/`PAGE`, chuẩn hoá TCVN3/VNI, truy xuất
  hybrid (vector + toàn văn `unaccent`, RRF), lọc quyền (`access_tags`, `DocumentAccessVerifier`), trích dẫn đã kiểm,
  rerank tuỳ chọn, đồng bộ bucket của files (`withFilesSync`).
- **`AiAgents`**: tool `@AiTool` chạy dưới danh tính người khởi chạy; `WRITE` cần xác nhận, `DESTRUCTIVE` luôn duyệt,
  `egress` duyệt khi đã nạp nội dung không tin cậy; giới hạn bước/chi phí/thời gian; MCP client (Streamable HTTP).
- **`AiExtraction`**: schema `vn.invoice.v1`, `vn.citizen-id.v1`, `vn.business-registration.v1`, `generic.table.v1`;
  XML hoá đơn điện tử đọc thẳng; OCR qua sidecar; validator `vn.tax-code`, `vn.citizen-id`, `vn.phone`, `money.sum`,
  `vat.rate`; độ tin cậy từng trường, `NEEDS_REVIEW` → duyệt.
- **Tuân thủ**: `AiIncidents` (hạn báo cáo 72 giờ / 05 ngày làm việc — NĐ 142/2026 Đ.19, khoá use case ngay), sổ đăng ký
  use case xuất CSV (Luật 134/2025 Đ.9), cờ `aiGenerated` trên mọi response, `AiErasure` xoá theo chủ thể.
- **REST** (`/api/v1/ai`): `POST /chat` (JSON/SSE), `/calls/{id}/feedback`, `/conversations`,
  `/kbs/{kb}/ask|retrieve|documents`, `/runs`, `/approvals`, `/extractions`; `/admin/**` (usage, calls, use-cases +
  `register.csv`, policy, providers, deployments, aliases, budgets, prompts, eval, kbs, embedding-spaces,
  extraction-schemas, agents, tools, mcp-servers, incidents, tasks, subjects, `info`); `/internal/**` cho chế độ remote.
- **SPI**: `ProviderFactory`, `AiAdvisor`, `PiiDetector`, `AiGuardrail`, `Chunker`, `DocumentExtractor`,
  `ContentResolver`, `AccessTagResolver`, `DocumentAccessVerifier`, `ExtractionValidator`, `AiEvaluator`.

## 4. Cách dùng nhanh

```xml
<dependency>
    <groupId>com.vyforge.platform</groupId>
    <artifactId>platform-starter-ai</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai</artifactId>
</dependency>
```

```yaml
vyforge.ai:
  default-policy: { cross-border: DENY, max-data-class: PERSONAL, capture: METADATA_ONLY }
  providers:
    vllm: { type: openai-compatible, base-url: http://vllm.ai.svc:8000/v1 }
  deployments:
    qwen-onprem: { provider: vllm, model: qwen3-32b, location: ON_PREM, max-data-class: SENSITIVE, price: { input: 0, output: 0 } }
  aliases:
    chat-default: { deployments: [ { ref: qwen-onprem, priority: 1 } ] }
```

```java
@Bean
AiUseCase replyDraft() {
    return AiUseCase.builder("crm.reply-draft")
            .prompt("crm.reply-draft").maxDataClass(DataClass.PERSONAL)
            .oversight(HumanOversight.HUMAN_SENDS)
            .purpose("Soạn nháp trả lời khiếu nại cho nhân viên CSKH")
            .build();
}

AiResponse draft = ai.call(AiRequest.builder("crm.reply-draft")
        .variable("ticket", ticket.body())
        .piiHints(PiiHints.of(PiiTypes.PERSON, customer.fullName()))
        .build());
```

Frontend: route handler `createAiChatRoute()` + `useAiChat({ endpoint, useCase })` + `<AiChat>`; luôn hiện
`AiDisclosure`.

## 5. Cấu hình quan trọng

| Khoá | Mặc định | Ý nghĩa |
|---|---|---|
| `vyforge.ai.mode` | `embedded` | `remote`: dùng module ở platform-services |
| `vyforge.ai.default-policy.cross-border` | `DENY` | `DENY` / `REDACTED_ONLY` / `ALLOW` (cần `ctiaRef`) |
| `vyforge.ai.default-policy.max-data-class` | `PERSONAL` | Lớp dữ liệu cao nhất mặc định cho use case |
| `vyforge.ai.providers.*`, `deployments.*`, `aliases.*` | — | Provider, model, vị trí dữ liệu, giá, nhóm fallback |
| `vyforge.ai.knowledge.default-space`, `spaces.*` | — | Embedding space cho RAG |
| `vyforge.ai.budget.fail-mode` | `closed` | Valkey mất: từ chối (`closed`) hay giữ chỗ trên DB (`open`) |
| `vyforge.ai.budget.flush-interval` / `counter-store` | `10s` / tự chọn | `jdbc` để tắt Valkey |
| `vyforge.ai.prompts.eval-gate` / `max-render-time` | — / `200ms` | Chặn lên production khi eval chưa đạt |
| `vyforge.ai.mcp.client.enabled` / `allowed-hosts` | — | MCP client |
| `vyforge.ai.extraction.engines.<docling\|paddleocr\|tesseract>.url`, `ocr-default` | — | OCR sidecar |
| `vyforge.ai.schema.migrate` | `true` | Flyway riêng `flyway_schema_history_ai` |
| `vyforge.ai.remote.url` | — | Server (thường `http://platform-services`), registration `vyforge-ai`, scope `ai.internal` |

Provider/deployment/alias/giá/chính sách cũng quản trị được trong DB qua `/api/v1/ai/admin/**` (`?scope=platform` cho
người vận hành; tenant tự mang khoá khi `tenants-may-bring-keys`). Secret mã hoá bằng `SecretCipher`.

## 6. Tích hợp với module khác

| Module | Tích hợp |
|---|---|
| files | KB đồng bộ bucket (`FileAvailable`, `FileTrashed`/`FilePurged`); quyền tài liệu `files:file:Download` |
| authz | Bắt buộc cho REST quản trị và kiểm quyền tool của agent |
| scheduling | Queue `ai` (ingest, eval, agent resume) |
| privacy-dsr | `AiErasure` xoá theo chủ thể (`ai:subject:Get\|Erase`) |
| collaboration, workflow, automation | Có điểm nối trong thiết kế (`ContentClassifier`, `ApprovalRouter`, hành động AI) nhưng **chưa có adapter** trong code |

## 7. Chế độ triển khai, IAM, sự kiện

- **Chế độ**: `embedded` và `remote` (đã cài cho `AiGateway`, `KnowledgeBases`: gọi `/api/v1/ai/internal/**`, chuyển
  tiếp subject/role/group của người dùng; server giới hạn tenant theo client
  `vyforge.ai.internal-api.clients.<azp>.tenants`). Agent, trích xuất, eval chỉ chạy trên server.
- **IAM**: `ai:usecase:Invoke|List|Update`, `ai:call:List|Get|GetContent`, `ai:conversation:Get|Delete`,
  `ai:kb:List|Get|Query|Manage`, `ai:document:Ingest|Delete`, `ai:agent:Run|Approve`, `ai:run:Get|Cancel`,
  `ai:tool:Manage`, `ai:extraction:Run|Get|Review`, `ai:prompt:List|Get|Update|Publish`, `ai:eval:Run|Manage`,
  `ai:budget:Get|Update`, `ai:policy:Get|Update`, `ai:provider:Manage`, `ai:usage:Get`, `ai:incident:Manage`,
  `ai:subject:Get|Erase`, `ai:task:Manage`.
- **Sự kiện**: `AiBudgetThresholdReached`, `AiBudgetExhausted`, `AiDocumentIngested`, `AiDocumentFailed`,
  `AiKbReindexed`, `AiPromptPublished`, `AiEvalRunCompleted`, `AiExtractionCompleted`, `AiExtractionReviewed`,
  `AiRunCompleted`, `AiRunWaitingApproval`, `AiIncidentRecorded`, `AiProviderDegraded`, `AiProviderRecovered`.
- **Giấy phép**: module `licensed` (`ai`, từ 0.14.0).

## 8. Trạng thái & còn mở

- Đã có: gateway đầy đủ pipeline, cổng dữ liệu, che PII VN, ngân sách (Valkey), prompt/eval, RAG + rerank, agent + MCP
  client, trích xuất + OCR sidecar, sự cố/sổ đăng ký, chế độ remote cho gateway/KB, zone admin.
- Chưa có: cooldown deployment và cache response trên Valkey (vẫn trong bộ nhớ từng instance); agent/trích xuất ở chế độ
  remote; token exchange RFC 8693 (đang chuyển tiếp danh tính bằng header); OCR bằng model vision; MCP server.
- Pháp lý: hạn báo cáo sự cố và sổ đăng ký dựa trên NĐ 142/2026, Luật 134/2025 — cần pháp chế của khách xác nhận khi
  triển khai.

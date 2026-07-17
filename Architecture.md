# Digital Expert Agents — Architecture (Hack CX Together 2026, đề bài SHB)

> Kiến trúc giải đề **"Digital Expert Agents — Đội ngũ Chuyên gia AI cho Vận hành Ngân hàng"**.
> Tái dùng **triết lý kiến trúc** từ [`aucobot`](../aucobot/aucobot-architecture.md) (planner-executor star topology, Agent/Bot tách biệt, Block execution có kiểm soát, plugin registry) — **không tái dùng domain/data** (Aucobot là marketing SMB, đề này là banking ops). Xem phân tích đối chiếu đầy đủ ở cuối file (§9).

### Ký hiệu trạng thái

| Ký hiệu | Ý nghĩa |
|---|---|
| **✅ Đã chốt** | Quyết định rõ, sẵn sàng implement |
| **🔜 Sprint kế tiếp** | Việc cần làm ngay, chưa code |
| **💡 Mở / sau demo** | Ý tưởng, không bắt buộc cho bản demo thi |

---

## 0. Quyết định đã chốt (đừng bàn lại giữa chừng)

| # | Quyết định | Lý do |
|---|---|---|
| 1 | **Backend: NestJS/TypeScript** (`backend/`), **Frontend: Next.js/React** (`frontend/`), **MCP server: process riêng** (thuộc ownership backend) | Giám khảo Hack CX Together còn đánh giá **năng lực triển khai/devops** (nhiều service, Docker, orchestration thật) — kiến trúc nhiều service thể hiện chiều sâu kỹ thuật hơn 1 app fullstack đơn |
| 2 | **DB: PostgreSQL tự quản** (Docker Compose local → Railway khi deploy) + **pgvector** | Kiểm soát hạ tầng hoàn toàn, kể chuyện devops rõ (migration, seed, backup) thay vì "managed black-box" |
| 3 | **Deploy:** `backend/` (+ MCP servers) → Railway/Docker, `frontend/` → Vercel | Tách deploy đúng ranh giới team; FE không cần biết DB/MCP |
| 4 | **MCP là giao thức tool THẬT** (`@modelcontextprotocol/sdk`), mỗi hệ thống vận hành = **1 MCP server độc lập process** (transport stdio hoặc SSE) | Đánh trúng chữ đề bài *"...bao gồm cả MCP khi cần thiết"*; nhiều process độc lập = thêm điểm devops |
| 5 | **Không có quyền truy cập hệ thống SHB thật** — mock toàn bộ bằng dữ liệu công khai (SBV, chính sách công khai SHB) | Không có sandbox API từ ban tổ chức |
| 6 | Giữ **star topology qua Planner**, KHÔNG mesh agent-gọi-agent trực tiếp ở tầng cao nhất — nhưng cho phép **Specialist tự "fractal" thành planner con** trong phạm vi hẹp, có giới hạn (xem §2.3) | Đúng nguyên tắc đã kiểm chứng ở Aucobot (tránh vòng lặp vô tận, dễ audit) + đáp ứng yêu cầu "agent tự tạo multi-agent" mà vẫn kiểm soát được |
| 7 | Repo riêng `auco-team/` (không sửa `aucobot/` sản phẩm marketing) | Tách rủi ro — đề thi không làm bẩn roadmap sản phẩm chính |
| 8 | **Chỉ 2 phần: `frontend/` + `backend/`** — **không** monorepo packages, **không** Turborepo/pnpm workspace phức tạp | Team frontend và backend làm **độc lập**, không đụng code nhau; nối lại qua REST + WebSocket (xem §8). Giảm setup, giảm conflict merge |
| 9 | **Copy code Aucobot vào bên trong `backend/`** (llm, prisma, mcp helper) — không giữ package workspace riêng | Vẫn tái dùng logic đã có, nhưng nằm trong 1 repo backend tự chứa đủ phụ thuộc |
| 10 | **RAG: LlamaIndex.TS** trong `backend/src/rag/`, **Vercel AI SDK** cho agent loop | Tách trách nhiệm rõ — LlamaIndex chỉ lo RAG; xem §4 |
| 11 | **Không auth / không multi-user** trong bản demo — 1 phiên cố định, mở app là dùng | Không phải yêu cầu đề bài; bỏ Passport/JWT/OTP/login UI để dồn thời gian cho planner, MCP, RAG, Dashboard |
| 12 | **Mô hình multi-agent 2 tầng (đã chốt):** (1) Planner cấp cao chia việc cho Specialist; (2) Specialist **tự tạo worker tạm** để chạy song song / đa nguồn rồi **tự tổng hợp** — xem §2.3 | Đúng yêu cầu "agent chuyên gia tự tạo multi-agent"; fractal star có giới hạn cứng → thuyết trình rõ, runtime kiểm soát được |
| 13 | **Không mesh linh hoạt** (CrewAI/AutoGen-style) — vẫn đủ đề bài; ưu tiên quan sát / đánh giá / bảo mật (§7.5) | Đề bài chấm năng lực collaboration + action, không chấm topology mesh; star + worker spawn mạnh hơn về audit cho banking |

> **Đã cân nhắc và loại bỏ:**
> - Phương án Next.js fullstack + Vercel + Supabase — loại vì giám khảo đánh giá devops.
> - Auth / multi-tenant — loại vì không thuộc deliverable đề bài.
> - Monorepo `packages/*` + Turborepo (pattern Aucobot) — loại vì tăng độ phức tạp khi đã chia team FE/BE làm song song; FE không cần import shared package nếu chỉ nói chuyện qua API.
> - Mesh agent↔agent tự do (CrewAI-style) — loại vì vòng lặp / khó audit; thay bằng fractal star có trần worker. **Vẫn đúng đề bài** (§7.5.4).
>
> Ghi lại ở đây để không đề xuất lại.

---

## 1. Đề bài — tóm tắt

**Tên:** Digital Expert Agents — Đội ngũ Chuyên gia AI cho Vận hành Ngân hàng

**Yêu cầu cốt lõi:** Hệ thống AI đa tác tử, mỗi tác tử = chuyên gia số một mảng nghiệp vụ (Tín dụng, Pháp chế & Tuân thủ, Sản phẩm, Vận hành). Agent phải: tự chủ lập kế hoạch (planning), dùng tool/function calling, RAG chuyên biệt theo agent, cộng tác chéo, **thực thi hành động thật** trên hệ thống vận hành (không chỉ trả lời text).

**Deliverables bắt buộc:**
1. Demo ≥ 2-3 chuyên gia số cộng tác giải 1 yêu cầu phức tạp
2. Cơ chế **planner agent** chia việc cho **executor agents**
3. Tool-use thật: gọi API / truy vấn dữ liệu / hành động cụ thể
4. **Dashboard** hiển thị trace agent, trạng thái nhiệm vụ, quyết định, luồng cộng tác
5. **So sánh** single-agent chatbot vs hệ thống multi-agent

---

## 2. Mô hình multi-agent đã chốt — Planner + Specialist + Worker

Kiến trúc điều phối: **planner-executor** (đúng deliverable đề bài) kết hợp **Specialist tự spawn worker** (§0 #12). Topology hình sao lồng 1 cấp (fractal star) — không mesh agent↔agent.

**Ví dụ demo chính:**

> *"Khách hàng Nguyễn Văn A muốn vay 2 tỷ mua nhà, kiểm tra đủ điều kiện tín dụng không, có vướng quy định AML/tuân thủ không, sản phẩm vay nào phù hợp nhất, và tạo hồ sơ vận hành nếu đủ điều kiện."*

```text
Planner nhận goal → sinh TaskPlan (structured output, KHÔNG tự do văn bản):

  Step 1 [Credit Agent]      — song song ──┐   (Credit tự spawn ≤3 worker bên trong)
  Step 2 [Legal/Compliance]  — song song ──┼──► Step 4 [Product Agent]
                                            │      (chờ Step 1+2 xong)
                                            ▼
                              Step 5 [Operations Agent] — tạo hồ sơ
                              (CHỈ chạy nếu Step1=đạt AND Step2=không vướng)
                                            ▼
                              Step 6 [Planner] — tổng hợp trả lời user
```

### 2.1 Data model — `TaskRun` / `TaskStep`

```prisma
model TaskRun {
  id         String   @id @default(cuid())
  // không có ownerId — demo không auth / không multi-user (§0 #11)
  goal       String   @db.Text        // câu yêu cầu gốc của user
  status     String                    // planning | running | done | failed
  planJson   Json                      // TaskPlan structured output từ Planner
  finalAnswer String? @db.Text
  createdAt  DateTime @default(now())
  steps      TaskStep[]
}

model TaskStep {
  id               String   @id @default(cuid())
  taskRunId        String
  agentRole        String                    // credit | legal | product | ops
  mode             String   @default("direct") // direct | spawn_workers — Specialist tự chọn (§2.3)
  input            Json
  output           Json?
  status           String                    // pending | running | waiting_approval | done | failed
  dependsOn        String[]                  // id các TaskStep khác phải xong trước
  toolCalls        Json[]  @default([])      // trace: { tool | worker, mcpServer?, input, output, ragCitations? }
  startedAt        DateTime?
  finishedAt       DateTime?
}
```

> Bảng này **vừa là engine chạy plan, vừa là nguồn dữ liệu duy nhất nuôi Dashboard** (§6) — thiết kế 1 lần, dùng 2 mục đích, không trùng lặp state.

### 2.2 Luồng chạy

```text
1. User gửi goal → Planner Agent (LLM) gọi generateObject (Zod schema TaskPlan)
   → sinh danh sách TaskStep + dependsOn (KHÔNG tự do — ép schema)
   → với case demo chính: few-shot ghim TaskPlan đã validate (§2.4)
2. Orchestrator đọc TaskPlan → topological sort:
   - Step không có dependsOn / dependsOn đã done → dispatch song song (concurrency cap)
   - Step có dependsOn chưa done → giữ pending
3. Mỗi TaskStep dispatch tới đúng Specialist Agent (theo agentRole):
   - Specialist TỰ chọn DIRECT hoặc SPAWN WORKERS (§2.3) — demo: Credit luôn SPAWN
   - Mode SPAWN: ≤3 worker song song → Specialist aggregate → output
   - trả output theo outputSchema cố định
4. Step có side-effect thật (submit hồ sơ, flag AML) → status = waiting_approval,
   chờ người duyệt (Approval queue, §5) mới chuyển done
5. Khi mọi step done → Planner step cuối "synthesize" → tổng hợp câu trả lời cuối + trích dẫn
```

**Guardrail:** giới hạn số step tối đa / `TaskRun`, timeout mỗi step, turn budget — tránh token nổ và plan vô hạn.

### 2.3 Specialist tự tạo worker rồi tổng hợp

Agent chuyên gia **không chỉ nhận việc rồi gọi 1 tool** — khi cần đa nguồn / song song, nó **tự tạo worker tạm**, chạy song song, **tự gom dữ liệu**, rồi trả kết quả đã tổng hợp lên Planner.

#### Hai tầng (không lẫn)

```text
Tầng 1 — PLANNER (điều phối liên phòng ban)
  User goal → chia TaskStep cho Credit / Legal / Product / Ops
  → chờ đủ phụ thuộc → synthesize câu trả lời cuối

Tầng 2 — SPECIALIST (chuyên gia số trong 1 nghiệp vụ)
  Nhận TaskStep của mình → TỰ quyết định:
    A) DIRECT: 1-2 tool/RAG là đủ → làm luôn, trả output
    B) SPAWN WORKERS: cần nhiều nguồn song song → tự tạo ≤3 worker tạm
         → chạy song song → LLM aggregate → trả output đã gom
```

#### Ví dụ Mode SPAWN — Credit Agent

```text
Credit Agent nhận: "Đánh giá khả năng vay 2 tỷ của KH Nguyễn Văn A"
  → Credit Agent TỰ quyết định cần đa nguồn → spawn:

      Worker W1 ──► get_credit_score (MCP core-banking)     ─┐
      Worker W2 ──► get_transaction_history (MCP)            ─┼─► Credit Agent aggregate
      Worker W3 ──► check_loan_eligibility (MCP los) + RAG   ─┘     → 1 báo cáo tín dụng

  → Trả Planner: { eligible, score, risks, citations, recommendation }
  → Planner chỉ thấy output đã gom; Dashboard vẫn hiện W1/W2/W3 lồng trong node Credit
```

**Thuật ngữ thuyết trình:**

| Tên | Là gì | Sống bao lâu |
|---|---|---|
| **Planner** | Agent hệ thống điều phối liên domain | Cả `TaskRun` |
| **Specialist** | Agent chuyên gia cố định (Credit/Legal/Product/Ops) | Seed sẵn / cả `TaskRun` |
| **Worker** | "Agent tạm" do Specialist tạo cho 1 tiểu-nhiệm | **Chỉ trong 1 TaskStep** — xong thì huỷ, không vào danh bạ |

→ Giám khảo hỏi *"có tự tạo agent không?"*: **Có — Specialist tự spawn worker (ephemeral agents) cho tác vụ song song, rồi tự tổng hợp.** Không tạo vô hạn agent vĩnh viễn.

#### Giới hạn cứng

| Giới hạn | Giá trị | Lý do |
|---|---|---|
| Ai được spawn | **Chỉ Specialist** — worker **không** spawn tiếp | Độ sâu tối đa = 1 (không đệ quy) |
| Số worker / step | **≤ 3** | Đủ minh họa song song, token/latency kiểm soát được |
| Ai quyết định Mode SPAWN | **Specialist tự quyết** | Đúng tinh thần "tự tạo"; không bắt Planner micromanage |
| Side-effect | Worker **chỉ đọc / phân tích**; `mutates: true` → Specialist đề xuất → Approval | Không để worker tạm tự ghi vào hệ thống ngân hàng |
| Demo chính | **Credit Agent luôn SPAWN** (ghim few-shot/prompt) | Thuyết trình trôi chảy, DAG nhìn thấy spawn song song |

Trace: mỗi worker = 1 entry trong `TaskStep.toolCalls` `{ type: "worker", workerId, goal, tool, input, output }` — không thêm bảng Prisma.

**Không mesh Specialist↔Specialist:** Credit không gọi thẳng Legal. Planner giữ vai trò lập kế hoạch liên phòng ban; Specialist chỉ spawn **trong domain của mình** (đúng nghiệp vụ ngân hàng).

### 2.4 Kịch bản demo dựng sẵn

Nguy cơ khi Planner tự do sinh TaskPlan lúc live: plan lệch mỗi lần chạy. Giải pháp — **ghim kịch bản bằng few-shot**, không hard-code giả:

```text
1. Chọn CỐ ĐỊNH 1-2 "goal chính" cho demo (ví dụ ở đầu §2)
2. Chạy thử Planner nhiều lần → validate TaskPlan ổn định
   (đúng số step, agentRole, dependsOn)
3. Đưa TaskPlan đã validate vào few-shot system prompt của Planner
   → Planner vẫn generateObject thật, nhưng case demo lệch xác suất thấp
4. Case ngoài kịch bản vẫn chạy engine chung — không giả, chỉ kém ổn định hơn case đã luyện
```

Mục tiêu: **thuyết trình trôi chảy cho case chính**, hệ thống vẫn tổng quát khi bị hỏi xoáy.

---

## 3. MCP servers — mock hệ thống vận hành SHB

Mỗi hệ thống vận hành = **1 MCP server độc lập process**, dùng `@modelcontextprotocol/sdk` (TypeScript), transport **stdio** (local/dev) hoặc **SSE/Streamable HTTP** (khi deploy). Agent NestJS (`backend/`) đóng vai **MCP client**. Tách process đúng nguyên tắc "core không import feature" đã có trong Aucobot — và là điểm devops giám khảo muốn thấy: nhiều service chạy song song, orchestrate bằng Docker Compose / Railway.

| MCP server | Domain agent dùng | Tool mock bên trong | Dữ liệu backing |
|---|---|---|---|
| `mcp-core-banking` | Credit, Operations | `get_account_balance`, `get_transaction_history`, `get_credit_score` | Postgres seed — dữ liệu khách hàng giả lập |
| `mcp-los` (Loan Origination) | Credit, Operations | `check_loan_eligibility`, `submit_loan_application` (⚠️ side-effect → qua Approval) | Postgres seed |
| `mcp-compliance` | Legal/Compliance | `run_aml_check`, `search_regulation` (RAG-backed), `flag_transaction` (⚠️ side-effect) | Văn bản SBV công khai + regulation embeddings |
| `mcp-product` | Product | `list_products`, `check_product_eligibility`, `compare_products` | Bảng sản phẩm công khai SHB (biểu phí, lãi suất) |
| `mcp-ops` | Operations | `create_service_ticket`, `get_ticket_status`, `assign_department` | Postgres seed |

**Mỗi tool khai báo rõ side-effect hay không** (field `mutates: boolean` trong tool definition) — Orchestrator dùng field này để tự động ép `TaskStep` vào `waiting_approval` khi cần, không phải agent tự quyết (đúng nguyên tắc *"Agent không có quyền quyết định"* đã chốt sẵn trong Aucobot).

> **Ưu tiên dựng cho demo:** 2 MCP server (`mcp-los` + `mcp-compliance`) đủ minh họa side-effect + RAG. 3 server còn lại có thể mock nông hơn hoặc stub cứng nếu thời gian thiếu — nhưng vẫn giữ folder riêng để kể chuyện "5 hệ thống vận hành độc lập".

---

## 4. RAG chuyên biệt theo agent — LlamaIndex.TS

Aucobot hiện tại **chủ đích defer** RAG (doc ngắn → nhồi context window, không vector DB) — hợp lý cho marketing, **không đủ cho banking**. Phải nâng cấp thật cho bài thi này.

### 4.1 Vì sao chọn LlamaIndex (và phạm vi dùng)

| Quyết định | Lý do |
|---|---|
| **Dùng LlamaIndex.TS** | Mạnh nhất ở tầng **ingest** (đọc tài liệu → chunk theo nhiều chiến lược → embed → lưu vector store) — tự viết tốn thời gian, LlamaIndex làm sẵn tốt, đủ mature cho TypeScript |
| **Chỉ dùng cho ingest + retrieval** | Không dùng LlamaIndex làm agent framework (AgentWorkflow / ReActAgent...). Agent loop / tool-calling vẫn giữ **Vercel AI SDK** — tránh 2 framework AI chồng lấn trách nhiệm; copy pattern `streamAgentWithTogether` từ Aucobot vào `backend/src/llm/` |
| **Cách ly rủi ro** | Nếu LlamaIndex có vấn đề (bản TS kém trưởng thành hơn Python), chỉ ảnh hưởng module `backend/src/rag/` — không lan ra orchestrator / MCP / Dashboard |

### 4.2 Cấu hình kỹ thuật

| Thành phần | Quyết định |
|---|---|
| Framework RAG | **LlamaIndex.TS** (`llamaindex`) — `SimpleDirectoryReader` / `Document` → `SentenceSplitter` → `VectorStoreIndex` |
| Vector store | **pgvector** qua adapter LlamaIndex (`PGVectorStore`) — Postgres đã có trong stack Docker Compose, không thêm hạ tầng mới |
| Namespace | **Theo domain agent** — Credit / Legal / Product / Ops mỗi domain 1 index/collection riêng, **không lẫn tri thức** (Legal không thấy chunk của Product) |
| Embedding | Together AI embedding API (cùng provider `llm-services` đang dùng) — LlamaIndex hỗ trợ custom `Embedding` class, wrap Together |
| Nguồn dữ liệu (mock, công khai) | Văn bản SBV (thông tư tín dụng, quy định AML/KYC công khai), trang chính sách/sản phẩm công khai của SHB, quy trình vận hành mẫu tự soạn |
| Bắt buộc trả **citation** | Mọi kết quả RAG trả kèm `{ sourceDoc, section, score }` từ LlamaIndex `NodeWithScore` metadata — hiển thị trên Dashboard, mô phỏng yêu cầu compliance-grade của ngân hàng |

```text
backend/src/rag/                       # nằm trong backend, không phải package workspace riêng
  ingest.ts                            # đọc data/ → LlamaIndex Document → chunk → embed → PGVectorStore
  retrieve.ts                          # retrieve(domain, query, topK) → NodeWithScore[] + citation
  embeddings/together.ts               # custom Embedding class wrap Together AI
  indexes.ts                           # factory: getIndex(domain) → VectorStoreIndex riêng theo domain
```

RAG search expose thành **MCP tool riêng cho mỗi domain** (`credit_kb_search`, `legal_kb_search`...) — không phải hàm nội bộ — vì đề bài liệt tách biệt "RAG" và "tool use" nhưng về mặt kỹ thuật RAG search chính là 1 tool. Bên trong tool gọi `backend/src/rag/retrieve(domain, query)`.

```prisma
// Prisma — bảng metadata quan hệ (để Dashboard liệt kê nguồn, không phải chính vector store)
model KnowledgeDocument {
  id        String   @id @default(cuid())
  domain    String                    // credit | legal | product | ops
  title     String
  sourceUrl String?
  content   String   @db.Text
  // vector thực tế do LlamaIndex PGVectorStore quản lý (bảng riêng của LlamaIndex,
  // KHÔNG khai báo qua Prisma — tránh xung đột schema)
}
```

---

## 4.5 Agent framework & memory strategy

### 5.1 Có cần CrewAI / AutoGen không?

**Không dùng CrewAI / AutoGen cho bản demo này.** Lý do không phải vì các framework đó kém, mà vì chúng không khớp với mục tiêu hiện tại:

| Framework | Đánh giá | Quyết định |
|---|---|---|
| **CrewAI** | Mạnh ở mô hình role-based team, nhanh để demo bằng Python. Nhưng stack chính của repo là TypeScript/NestJS, và CrewAI thiên về agent collaboration hơn là audit/state/approval đặc thù ngân hàng | Không dùng |
| **AutoGen** | Mạnh cho multi-agent conversation / research loop, nhưng mesh/handoff linh hoạt dễ khó kiểm soát khi cần trace, approval và side-effect rõ ràng | Không dùng |
| **LangGraph.js** | Có TS/JS, mạnh về stateful graph. Cân nhắc nếu workflow phức tạp hơn sau demo | Defer |
| **Mastra / framework TS agent khác** | Hữu ích nếu muốn memory/workflow có sẵn. Nhưng thêm framework mới lúc này làm tăng integration risk | Defer |
| **Vercel AI SDK + orchestration tự kiểm soát** | Đã có code Aucobot, native TypeScript, tool-calling/structured output/streaming đủ dùng. State nằm trong `TaskRun`/`TaskStep`, không bị ẩn trong framework | **Chốt dùng** |

**Kết luận:** tự điều phối bằng domain state machine (`TaskRun`/`TaskStep`) + Vercel AI SDK là lựa chọn đúng cho cuộc thi. Nó đủ để chứng minh multi-agent, nhưng vẫn kiểm soát được audit, approval và trace.

### 5.2 Thư viện TS nên dùng

| Nhu cầu | Thư viện / module |
|---|---|
| Agent loop, structured output, tool calling, streaming | **Vercel AI SDK** (`ai`) — copy/adapt từ Aucobot `llm-services` vào `backend/src/llm/` |
| RAG ingest + retrieval | **LlamaIndex.TS** trong `backend/src/rag/` |
| Vector store | PostgreSQL + pgvector |
| Backend API + WS | NestJS |
| Multi-agent orchestration | Code domain riêng: `backend/src/planning/`, không dùng framework agent mesh |
| MCP tool adapter | `@modelcontextprotocol/sdk` trong `backend/mcp-servers/*` |

Nếu sau demo cần nâng cấp, ứng viên hợp lý nhất là **LangGraph.js** cho durable graph phức tạp hơn. Nhưng không nên đưa vào MVP thi vì `TaskRun`/`TaskStep` đã đủ và dễ demo hơn.

### 5.3 Chiến lược memory — short, working, long

Banking agent không nên “nhớ tự do” như chatbot cá nhân. Memory phải phân tầng, có nguồn, có TTL hoặc có audit.

| Loại memory | Lưu ở đâu | Dùng để làm gì | TTL / kiểm soát |
|---|---|---|---|
| **Short-term / conversation memory** | `TaskRun.goal`, `TaskStep.input/output`, `toolCalls` | Context trong 1 yêu cầu đang chạy, Dashboard trace | Theo `TaskRun`, giữ để audit demo |
| **Working memory** | Biến runtime trong từng Specialist / worker, sau đó ghi vào `TaskStep.output` | Tổng hợp kết quả worker song song trước khi trả Planner | Không lưu riêng; muốn giữ thì phải nằm trong output |
| **Long-term domain memory** | RAG index qua LlamaIndex + pgvector (`KnowledgeDocument`, vector store) | Quy định tín dụng, AML, sản phẩm, quy trình vận hành | Chỉ cập nhật qua seed/ingest, có source/citation |
| **Agent profile memory** | Config seed trong code/DB: role, instructions, tool allowlist | Giữ persona và phạm vi của Credit/Legal/Product/Ops | Cố định trong demo |
| **Learned memory** | **Không làm trong demo** | Agent tự học sau mỗi phiên | Defer — banking cần duyệt trước khi ghi nhớ bền |

### 5.4 Quy tắc memory cho demo

1. **Không để agent tự ghi long-term memory** trong demo. Long-term chỉ đến từ tài liệu đã ingest.
2. **Mọi kết luận nghiệp vụ phải có citation** từ RAG hoặc tool result.
3. **Worker memory không sống lâu:** worker chạy xong, kết quả được Specialist gom vào `TaskStep.output`, worker bị huỷ.
4. **Task trace là nguồn sự thật:** Dashboard đọc `TaskRun`/`TaskStep`, không đọc log rời rạc.
5. **Nếu cần “nhớ” khách hàng trong demo:** seed sẵn trong mock DB (`Customer`, `Account`, `LoanApplication`) thay vì để LLM tự nhớ.

### 5.5 Có nên làm learned memory sau demo?

Có, nhưng phải làm kiểu enterprise:

```text
Agent đề xuất fact mới
  → ghi vào MemorySuggestion(status=pending)
  → human duyệt
  → mới promote thành AgentMemory / KnowledgeDocument
```

Không tự động ghi thẳng vì banking rất nhạy với sai lệch dữ liệu và compliance.

---

## 5. Thực thi hành động có kiểm soát (tái dùng nguyên khuôn Block/Bot Aucobot)

Đây là tài sản **mạnh nhất và tái dùng được nhiều nhất** từ Aucobot — nguyên tắc đã chốt sẵn:

> *"AI Agent... không tự quyết định thay user về việc có tác động thật... Mọi quyết định cuối cùng thuộc về con người."*

Áp dụng thẳng cho banking — bất kỳ `TaskStep` nào gọi tool có `mutates: true` (submit hồ sơ vay, flag AML, tạo ticket vận hành):

```text
Agent chuẩn bị action → dry-run (không side-effect thật) → hiện preview cho "người duyệt"
   → user (đóng vai chuyên viên/quản lý) bấm Duyệt / Từ chối trên Dashboard
   → Duyệt → step chuyển done, tool thực thi thật (trên mock DB)
   → Từ chối → step failed, Planner tự điều chỉnh câu trả lời cuối
```

Tái dùng 3 tầng lỗi đã thiết kế (`BlockUserError` / `BlockOperationalError` / `BlockUnexpectedError`) để dịch lỗi sang tiếng người trên Dashboard — điểm cộng lớn khi demo trước giám khảo không rành kỹ thuật.

**Approval UI cho demo:** dùng người duyệt thật (bấm nút) cho case chính (thuyết phục hơn), có **nút "auto-approve demo mode"** dự phòng nếu cần chạy nhanh không đứng chờ — an toàn cho lúc live demo.

---

## 6. Dashboard — trace, quyết định, luồng cộng tác

Chưa tồn tại trong Aucobot (chỉ có WS events rời rạc) — **xây mới**, nhưng dữ liệu lấy thẳng từ `TaskRun`/`TaskStep` (§2.1), không cần tầng lưu trữ riêng. Cập nhật realtime qua **WebSocket NestJS** (`@nestjs/websockets` + `ws`, pattern đã chốt trong Aucobot) — push event `task.step.updated` / `approval.updated` khi step đổi trạng thái. Client `frontend/` subscribe qua `lib/stream/`.

| Khu vực UI | Nội dung |
|---|---|
| **DAG view** | Vẽ TaskStep dạng graph (node = agent + trạng thái màu: pending/running/waiting_approval/done/failed), cạnh = `dependsOn`. Step Chế độ B (§2.3) có thể expand hiện worker call lồng bên trong |
| **Timeline / trace log** | Từng step: tool nào được gọi, input/output, RAG citation (từ LlamaIndex `NodeWithScore`), thời gian chạy |
| **Approval panel** | Danh sách step đang `waiting_approval`, nút Duyệt/Từ chối, preview trước khi duyệt |
| **Final answer** | Câu trả lời tổng hợp cuối + nguồn trích dẫn gộp từ mọi agent |
| **So sánh single vs multi-agent** | Bảng cạnh nhau: cùng 1 câu hỏi → chatbot đơn (1 agent, full tool, không planner) vs hệ thống này — thời gian, số tool gọi đúng, có trích dẫn hay không, có hành động thật hay chỉ trả lời text |

**Component:** tự viết nhẹ (React + SVG/Canvas đơn giản cho DAG), theo convention **"Components tự viết + Tailwind CSS"** — không cần kéo-thả nặng như React Flow vì Dashboard chỉ **hiển thị để xem**, không cần chỉnh sửa plan bằng tay trong bản demo.

> **Lệch với Aucobot:** Aucobot chốt CSS Modules + cấm Tailwind. Repo `auco-team/` (đề thi) **chủ đích dùng Tailwind** để dựng Dashboard nhanh hơn trong thời gian thi ngắn — chấp nhận lệch convention sản phẩm chính vì 2 repo độc lập.

---

## 7. So sánh single-agent vs multi-agent (deliverable #5)

Eval harness đơn giản, rẻ vì tái dùng `TaskRun`:

```text
Baseline "single-agent":  1 agent, system prompt gộp cả 4 domain, có tất cả tool/MCP
                           nhưng KHÔNG planner, KHÔNG routing theo domain
Hệ thống multi-agent:      Planner + 4 Specialist (kiến trúc §2)

Chạy cùng bộ N câu hỏi cross-functional (ví dụ ở §2) qua cả 2 luồng → so sánh:
  - Tỷ lệ gọi đúng tool / đúng domain
  - Có trích dẫn nguồn hay bịa (hallucination check thủ công trên N mẫu)
  - Có thực sự "hành động" (side-effect có audit) hay chỉ trả lời text
  - Thời gian phản hồi (multi-agent chậm hơn do nhiều lượt LLM — nói rõ trade-off, đừng giấu)
```

Kết quả hiển thị ngay trên Dashboard (§6) dạng bảng — vừa là deliverable #5, vừa là nội dung thuyết trình mạnh ("đây là lý do ngân hàng cần multi-agent, không chỉ chatbot").

---

## 7.5 Giám sát, Đánh giá & Bảo mật (Production-Ready)

Đề bài yêu cầu agent **thực thi hành động** trên hệ thống vận hành — với ngân hàng, phần này không thể chỉ là “AI trả lời hay”. Bản demo phải **minh họa được** 3 trụ: quan sát (observe), đánh giá (evaluate), bảo vệ (secure). Production đầy đủ làm sau; demo chứng minh đúng seam.

### 7.5.1 Giám sát (Observability) — đã có sẵn trong kiến trúc

| Tầng | Cơ chế | Hiện trên Dashboard |
|---|---|---|
| **Trace orchestration** | Mỗi `TaskRun` / `TaskStep` / `toolCalls` / worker spawn ghi DB | DAG view + timeline |
| **Realtime** | WebSocket `task.step.updated`, `approval.updated`, `task.done` | UI cập nhật live khi agent chạy |
| **Citation** | RAG trả `{ sourceDoc, section, score }` | Timeline + final answer |
| **Audit side-effect** | Mọi tool `mutates: true` bắt buộc qua Approval trước khi ghi mock DB | Approval panel + trạng thái `waiting_approval` |
| **Structured log (BE)** | Pino/Nest logger: `taskRunId`, `stepId`, `agentRole`, `tool`, latency, error class | Devops / postmortem |

**Nguyên tắc:** Dashboard không “vẽ ảo” — chỉ đọc state đã persist. Nếu step không có trong DB thì không hiện. Đây chính là điểm mạnh hơn mesh CrewAI/AutoGen (nơi hội thoại agent dễ mất trace).

### 7.5.2 Đánh giá (Evaluation) — deliverable #5 + guardrail chất lượng

| Loại eval | Cách làm trong demo | Metric |
|---|---|---|
| **A/B architecture** | Cùng goal: single-agent vs multi-agent (§7) | Thời gian, số tool đúng domain, có citation?, có action thật? |
| **Plan quality** | Few-shot ghim + validate schema Zod `TaskPlan` | Plan parse được, đủ Specialist, không vượt max steps |
| **RAG faithfulness** | Mỗi câu trả lời Legal/Credit phải kèm citation; mẫu N case kiểm thủ công | % câu có nguồn / không bịa |
| **Action safety** | Không có side-effect nào bypass Approval | 100% mutate tools đi qua queue |
| **Latency / cost** | Log token + wall-clock từng step | Biết rõ multi-agent chậm hơn nhưng đúng hơn — nói thật với giám khảo |

**Không làm trong demo (defer production):** LLM-as-judge tự động, regression suite hàng trăm case, online eval A/B traffic thật.

### 7.5.3 Bảo mật (Security) — demo tối thiểu + lộ trình production

| Hạng mục | Demo (làm ngay) | Production SHB (nói trong thuyết trình) |
|---|---|---|
| **AuthN/AuthZ** | Bỏ (đã chốt §0 #11) | OIDC/SSO ngân hàng, RBAC theo phòng ban, AgentGrant |
| **Human-in-the-loop** | Approval trước mọi side-effect | Maker-checker, dual control với giao dịch lớn |
| **Tool allowlist** | Mỗi Specialist chỉ thấy MCP tool đúng domain | Policy engine + least privilege per agent |
| **Secrets** | `.env` local / Railway secrets | Vault, không lộ token trong prompt/log |
| **Prompt injection / data exfil** | System prompt khóa domain; worker không được mutate | Input sanitization, output filter, DLP trên tool response |
| **Network boundary** | MCP server tách process, Docker network nội bộ | MCP/API chỉ trong VPC ngân hàng, mTLS |
| **Audit** | `TaskStep` + Approval history | Immutable audit log, retention theo quy định NHNN |
| **Data** | Mock / văn bản công khai | Phân loại dữ liệu, không đưa PII thật vào LLM public cloud nếu policy cấm |

**Câu trả lời khi giám khảo hỏi “production-ready chưa?”:**

> Bản demo chứng minh đúng các **seam kiểm soát** mà ngân hàng cần: trace end-to-end, approval trước hành động, tool allowlist, citation, và ranh giới MCP. Auth/RBAC/Vault là lớp tích hợp hạ tầng SHB — nhóm cố ý không làm trong hackathon để dồn effort vào multi-agent + action execution, nhưng kiến trúc đã chừa chỗ (không cần đập đi xây lại).

### 7.5.4 Không dùng mesh linh hoạt — có đúng đề bài không? ✅ Có

Đề bài yêu cầu năng lực, **không** bắt buộc topology mesh:

| Yêu cầu đề bài | Fractal star (kiến trúc đã chốt) đáp ứng thế nào |
|---|---|
| Multi-agent, mỗi agent = chuyên gia domain | Planner + Credit/Legal/Product/Ops |
| Tự chủ lập kế hoạch | Planner `generateObject` → `TaskPlan` DAG |
| Tool use + RAG | MCP tools + LlamaIndex per domain |
| Cộng tác chéo | Planner điều phối liên phòng ban; Specialist spawn worker song song trong domain rồi tổng hợp |
| Thực thi hành động trên hệ thống vận hành | MCP mutate + Approval |
| Planner-executor | Đúng mô hình đề xuất trong “Suggested Technologies” |
| MCP khi cần | MCP server process thật |
| Memory & state | `TaskRun`/`TaskStep` + RAG long-term |
| Dashboard traces | §6 + §7.5.1 |

**“Cộng tác” ≠ “mọi agent gọi mọi agent”.** Cộng tác đúng nghĩa ngân hàng là: nhiều chuyên gia cùng giải một yêu cầu liên chức năng, có người điều phối, có bàn giao kết quả, có kiểm soát. Mesh tự do (CrewAI/AutoGen-style) dễ tạo vòng lặp và khó audit — **yếu hơn** với tiêu chí production-ready của ngân hàng, dù trông “linh hoạt” hơn trên slide.

**Câu trả lời ngắn khi bị hỏi xoáy:**

> Nhóm cố ý không dùng mesh linh hoạt vì đề bài vừa yêu cầu cộng tác vừa yêu cầu thực thi hành động trong môi trường ngân hàng. Topology hình sao + Specialist tự spawn worker vẫn là multi-agent collaboration có lập kế hoạch, song song và tổng hợp — đồng thời quan sát được, đánh giá được và khóa được side-effect. Đây là lựa chọn **đúng đề bài và hướng production**, không phải cắt giảm.

---

## 8. Cấu trúc thư mục — chỉ `frontend/` + `backend/` (không packages)

### 8.1 Nguyên tắc làm việc song song (FE team ↔ BE)

| Ai | Sở hữu | Được phép | Không được |
|---|---|---|---|
| **Team frontend** | `frontend/` | UI, mock API client, Tailwind, Dashboard | Import code NestJS / Prisma / LLM / MCP |
| **Backend (bạn)** | `backend/` (+ `mcp-servers/` nếu tách folder) | NestJS, Prisma, Planner, agents, RAG, MCP | Sửa component React trừ khi sửa contract API |
| **Cả hai** | `API.md` (contract) | Đọc & đề xuất thay đổi contract trước khi merge | Nối thẳng DB từ frontend |

```text
Giai đoạn 1 (song song, chưa nối):
  FE  → dựng UI + Dashboard với mock JSON (fixtures) khớp API.md
  BE  → dựng NestJS + Planner + MCP + RAG, test bằng Postman / curl / Swagger

Giai đoạn 2 (nối API):
  FE  đổi NEXT_PUBLIC_API_URL → backend thật
  Bỏ mock, giữ nguyên shape response đã chốt trong API.md
```

**Contract duy nhất giữa 2 team:** REST + WebSocket — ghi trong `API.md` ở root (OpenAPI optional). Không dùng shared TypeScript package để “ép” type giữa FE/BE trong thời gian thi — tránh workspace phức tạp; nếu cần type an toàn, FE copy Zod schema nhẹ vào `frontend/lib/schemas/` từ `API.md`.

### 8.2 Cây thư mục

```
auco-team/
├── Architecture.md
├── API.md                              # contract REST + WS — FE/BE cùng giữ đúng
├── frontend/                           # Next.js + Tailwind — TEAM FRONTEND sở hữu
│   ├── package.json                    # độc lập, npm/pnpm riêng trong folder này
│   ├── src/
│   │   ├── app/
│   │   │   ├── dashboard/              # DAG, approval, so sánh
│   │   │   └── page.tsx                # nhập goal / chat demo
│   │   ├── components/
│   │   │   ├── dag-view/
│   │   │   ├── approval-panel/
│   │   │   └── compare-table/
│   │   └── lib/
│   │       ├── api/                    # REST client → NEXT_PUBLIC_API_URL
│   │       ├── stream/                 # WS client
│   │       ├── mock/                   # fixtures khi chưa có BE (Giai đoạn 1)
│   │       └── schemas/                # Zod copy từ API.md (optional)
│   └── .env.example                    # NEXT_PUBLIC_API_URL=
│
├── backend/                            # NestJS — BACKEND sở hữu (tự chứa đủ, không packages/)
│   ├── package.json                    # độc lập
│   ├── prisma/
│   │   └── schema.prisma               # TaskRun, TaskStep, KnowledgeDocument, …
│   ├── src/
│   │   ├── main.ts
│   │   ├── llm/                        # copy từ aucobot packages/llm-services
│   │   ├── rag/                        # LlamaIndex.TS ingest + retrieve (§4)
│   │   ├── mcp-client/                 # gọi các MCP server
│   │   ├── planning/                   # TaskRun/TaskStep + Planner
│   │   ├── agents/                     # Credit / Legal / Product / Ops
│   │   ├── approvals/
│   │   └── realtime/                   # WS gateway
│   ├── data/                           # SBV/SHB seed cho RAG
│   ├── scripts/
│   │   └── seed-knowledge.ts
│   ├── docker-compose.yml              # postgres (+ redis nếu cần) + mcp + api
│   ├── mcp-servers/                    # process riêng, vẫn thuộc BE — ưu tiên los + compliance
│   │   ├── mcp-los/
│   │   ├── mcp-compliance/
│   │   ├── mcp-core-banking/           # stub nếu thiếu thời gian
│   │   ├── mcp-product/
│   │   └── mcp-ops/
│   └── .env.example
```
> **Không có** `packages/`, **không có** Turborepo, **không có** pnpm workspace root. Mỗi bên `cd frontend` / `cd backend` rồi `pnpm install` / `npm install` riêng. Copy từ Aucobot: chỉ **nội dung file** llm/prisma helper vào `backend/src/`, không kéo cả monorepo scaffold.

### 8.3 Endpoint tối thiểu trong `API.md` (để FE mock trước)

| Method | Path | Mục đích |
|---|---|---|
| POST | `/api/task-runs` | Tạo TaskRun từ `goal` → trả `{ id, status, plan }` |
| GET | `/api/task-runs/:id` | Snapshot TaskRun + steps (Dashboard poll / first load) |
| POST | `/api/task-runs/:id/approve/:stepId` | Duyệt / từ chối step `waiting_approval` |
| POST | `/api/compare` | Chạy cùng goal trên single-agent vs multi-agent (deliverable #5) |
| WS | `/api/ws/task-runs/:id` | Push `task.step.updated`, `approval.updated`, `task.done` |

Payload chi tiết (Zod / JSON Schema) viết trong `API.md` — chốt trước khi FE dựng mock sâu.

## 9. Đối chiếu với Aucobot — cái gì tái dùng, cái gì xây mới

| Thành phần | Nguồn Aucobot | Mức tái dùng |
|---|---|---|
| Agent identity/instructions/skillGroups | `core/agents`, `agent-plan.md` §A.1 | Đổi preset data (Credit/Legal/Product/Ops), giữ nguyên schema |
| Planner-executor | Ý tưởng star topology, `aucobot-architecture.md` §Luồng tương tác trong Room | Giữ nguyên tắc star, **xây mới** logic DAG planning (§2) + **2 chế độ Specialist** (direct / sub-planner fractal, §2.3) |
| Tool use / MCP | `core/plugins` registry pattern | Đổi từ tool-wrap nội bộ → **MCP server process thật** (§3) |
| RAG | `AgentDocument` (text ngắn, không vector) | Nâng cấp lên **LlamaIndex.TS** + pgvector + namespace theo domain + citation bắt buộc (§4) |
| Thực thi có kiểm soát | Triết lý Block/Bot: schema ép giữa bước, dry-run, Approval, 3 tầng lỗi | **Tái dùng gần như nguyên khuôn**, đổi Block social → Block banking |
| State/trace | Ý tưởng `WorkflowRun/WorkflowStep` | Đổi tên `TaskRun/TaskStep`, dùng chung cho engine + Dashboard |
| Dashboard | WS events rời rạc (`message.chunk`, `approval.updated`) | **Xây mới** hoàn toàn UI trace |
| So sánh single vs multi-agent | Không có | **Xây mới**, nhưng rẻ vì tái dùng `TaskRun` |

---

## 10. Roadmap theo sprint (thời gian thi ngắn)

```text
Sprint 0 — Tách team + contract
  Tạo folder frontend/ + backend/ (mỗi bên package.json riêng, KHÔNG workspace)
  Viết API.md (REST + WS tối thiểu §8.3) — FE mock theo đây
  BE: NestJS skeleton + Prisma + docker-compose Postgres
  FE: Next.js + Tailwind skeleton + mock fixtures

Sprint 1 — BE: khung dữ liệu + Planner · FE: Dashboard shell (mock)
  BE: TaskRun/TaskStep + Planner generateObject + 1 Specialist Credit mock tool
  FE: DAG view + timeline UI đọc mock JSON khớp API.md
  Ghim kịch bản demo (§2.4) trên BE

Sprint 2 — BE: MCP + LlamaIndex RAG · FE: nối WS khi BE sẵn sàng
  BE: mcp-los + mcp-compliance; backend/src/rag (LlamaIndex); seed knowledge
  FE: thay mock bằng NEXT_PUBLIC_API_URL khi endpoint Sprint 1 ổn định

Sprint 3 — BE: DAG đầy đủ + Chế độ B + Approval · FE: Approval panel
  Dispatch song song + dependsOn + waiting_approval
  WS push task.step.updated

Sprint 4 — So sánh single vs multi-agent + polish Dashboard
  BE: POST /api/compare
  FE: bảng so sánh + expand worker Chế độ B

Sprint 5 — Devops + demo script
  Docker Compose full stack (backend + mcp + postgres) 1 lệnh
  FE deploy Vercel trỏ API production URL
  Script demo 1-2 case chính chạy mượt
```

---

## 11. Chiến lược startup — không gói gọn 1 bài thi

```text
Tầng 1 — Core ngang (đã có bóng dáng trong aucobot, generalize thêm)
  Agent (persona + tool allowlist) · Planner (DAG orchestration — mới ở đề này)
  Block/Bot (action có kiểm soát, approval, audit) · MCP registry
  → định vị lại: "Agent-Ops OS cho doanh nghiệp cần kiểm soát khi AI hành động"
    (không chỉ marketing — banking là vertical thứ 2, chứng minh tính tổng quát của core)

Tầng 2 — Vertical pack (cắm vào core)
  Pack Marketing (aucobot hiện có): Facebook/TikTok Block
  Pack Banking (bài thi này): Credit/Legal/Product/Ops Agent + MCP banking + RAG banking
  Pack tương lai: Insurance-Ops, Legal-Ops, Healthcare-Ops...

Tài sản bán độc lập — không phụ thuộc thắng/thua cuộc thi:
  "Banking Ops MCP Toolkit" — bộ connector chuẩn MCP (core-banking/LOS/compliance)
  dùng được với BẤT KỲ framework agent nào (LangGraph, CrewAI, Copilot Studio, Claude...)
  → hiện chưa có ai làm sẵn tại Việt Nam, giá trị độc lập với kết quả hackathon
```

---

## 12. Câu hỏi còn mở (cần quyết trước khi code sâu)

| # | Câu hỏi | Trạng thái |
|---|---|---|
| 1 | Mức độ DAG cần demo: cố định 4-5 step hay agent tự sinh số step động? | ✅ Đã chốt hướng — Planner vẫn `generateObject` thật, nhưng **ghim kịch bản** bằng few-shot TaskPlan đã validate (§2.4); ngoài case demo vẫn chạy tổng quát |
| 2 | Approval trong demo: người thật bấm duyệt hay auto-approve? | 💡 mở — gợi ý: người thật cho case chính, có nút demo-mode dự phòng |
| 3 | Có cần multi-user/auth trong bản demo? | ✅ **Đã chốt: bỏ auth** — 1 phiên cố định, không login (§0 #11) |
| 4 | Số lượng MCP server thực sự cần dựng cho demo: cả 5 hay 2-3 đủ? | 💡 mở — gợi ý: ưu tiên `mcp-los` + `mcp-compliance` (đủ minh hoạ side-effect + RAG), `mcp-product`/`mcp-ops` có thể mock nông hơn |
| 5 | Nguồn văn bản SBV/SHB công khai cụ thể nào dùng cho RAG? | 🔜 cần tổng hợp danh sách link cụ thể trước Sprint 2 |
| 6 | Specialist nào trong kịch bản demo chính dùng Mode B (spawn workers)? | ✅ **Đã chốt: Credit Agent** — luôn spawn ≤3 worker (score + lịch sử GD + eligibility/RAG) rồi tự tổng hợp (§0 #12, §2.3) |
| 7 | LlamaIndex.TS version / PGVectorStore adapter ổn định chưa? | 🔜 cần spike ngắn Sprint 0 — nếu adapter kém, fallback tự viết retrieval mỏng trên pgvector, giữ LlamaIndex chỉ cho chunking |

---

## 13. Q&A kiến trúc — khi giám khảo hỏi về lựa chọn công nghệ

### 13.1 Câu trả lời mở đầu

> Các công nghệ trong đề là **gợi ý**, còn tiêu chí cốt lõi là năng lực hệ thống: lập kế hoạch, phối hợp đa tác tử, RAG, tool use, MCP, thực thi hành động, human-in-the-loop và quan sát được toàn bộ trace. Nhóm không thay đổi yêu cầu; nhóm chọn công nghệ tương đương phù hợp hơn với năng lực TypeScript và tài sản đã có để dành thời gian giải quyết nghiệp vụ ngân hàng. Mỗi lựa chọn đều được cô lập sau contract chuẩn như MCP và Zod, nên không khóa hệ thống vào một framework.

Không trả lời theo hướng *“công nghệ của nhóm tốt hơn công nghệ đề xuất”*. Cần nhấn mạnh ba tiêu chí:

1. **Tốc độ giao hàng:** tái sử dụng code đã kiểm chứng thay vì học lại framework trong thời gian thi.
2. **Khả năng kiểm soát:** banking cần plan có schema, audit, approval và trạng thái bền vững hơn agent chạy tự do.
3. **Khả năng thay thế:** MCP, REST/WebSocket và Zod contract giữ lõi không phụ thuộc framework.

### 13.2 “Tại sao không dùng FastAPI mà dùng NestJS?”

> FastAPI rất phù hợp với hệ sinh thái AI Python. Tuy nhiên đội ngũ mạnh TypeScript và đã có sẵn các module NestJS cho REST, WebSocket, Prisma, logging và Docker. Dùng NestJS giúp chúng tôi giữ một ngôn ngữ xuyên suốt frontend, backend, MCP server và shared schema, giảm lỗi contract và thời gian tích hợp. Phần AI không bị hạn chế vì model provider, LlamaIndex và MCP đều có SDK TypeScript. Nếu SHB yêu cầu Python, các MCP server và API contract hiện tại cho phép thay service orchestration mà không đổi frontend hay hệ thống nghiệp vụ.

**Minh chứng nên chỉ trên demo:** `API.md` contract rõ giữa FE/BE, WebSocket trace realtime và MCP server độc lập.

### 13.3 “Tại sao không dùng LangGraph?”

> LangGraph mạnh cho workflow agent có state và checkpoint. Chúng tôi không phủ nhận giá trị đó. Với phạm vi demo, chúng tôi cần một DAG nhỏ nhưng phải lưu được từng bước vào PostgreSQL, hiển thị trực tiếp trên Dashboard và dừng chính xác tại bước phê duyệt. Vì vậy nhóm triển khai state machine miền nghiệp vụ bằng `TaskRun`/`TaskStep` với Zod schema. Cách này làm audit trail và approval trở thành dữ liệu hạng nhất, không phải trạng thái ẩn trong framework. Nếu workflow phát triển phức tạp hơn, LangGraph hoặc một durable workflow engine có thể thay executor phía sau mà không đổi `TaskPlan`, MCP tools hay UI.

**Không nên nói:** “LangGraph nặng” hoặc “tự viết tốt hơn”.  
**Nên nói:** phạm vi DAG được giới hạn cứng, không xây một framework tổng quát mới.

### 13.4 “Tại sao không dùng CrewAI hoặc AutoGen? Không mesh thì có đúng đề bài không?”

> CrewAI và AutoGen phù hợp với mô hình nhiều agent hội thoại và handoff linh hoạt. Trong ngân hàng, mesh agent-to-agent tự do làm tăng nguy cơ vòng lặp, chi phí token và khó truy nguyên trách nhiệm. Chúng tôi chọn topology hình sao: Planner cấp cao giao việc; Specialist chỉ được tạo tối đa ba worker trong một tầng con. Mô hình này vẫn chứng minh collaboration, nhưng mọi nhiệm vụ, tool call và quyết định đều có chủ thể và trace rõ ràng.
>
> Đề bài yêu cầu **năng lực** (planning, tool use, RAG, cộng tác, thực thi hành động, MCP, dashboard) — không bắt buộc mesh. Fractal star đáp ứng đủ các năng lực đó và còn khớp hơn với Giám sát / Đánh giá / Bảo mật (§7.5): cộng tác quan sát được và side-effect khóa được. “Cộng tác” ở đây là nhiều chuyên gia cùng giải yêu cầu liên chức năng dưới planner, không phải mọi agent gọi lẫn nhau.

Điểm nhấn trên Dashboard: giám khảo nhìn thấy Planner → Specialist → worker, không có trao đổi ẩn giữa các agent.

### 13.5 “Tại sao dùng Vercel AI SDK cho agent loop?”

> Vercel AI SDK cung cấp streaming, structured output, tool calling và hỗ trợ nhiều model provider trong TypeScript. Nhóm dùng nó như lớp gọi model mỏng, không dùng nó làm workflow engine. Planning, policy, approval và persistence vẫn do domain layer kiểm soát. Vì vậy đổi Together AI sang OpenAI, Claude hoặc model nội bộ của SHB không làm thay đổi kiến trúc điều phối.

### 13.6 “Tại sao dùng LlamaIndex.TS thay vì tự viết RAG hoặc dùng LangChain?”

> Giá trị cần chứng minh không phải là tự viết thuật toán chunking, mà là RAG chuyên biệt theo từng nghiệp vụ, có citation và kiểm soát nguồn. LlamaIndex chuyên về ingest, indexing và retrieval nên giúp nhóm hoàn thành phần này nhanh và có cấu trúc. Chúng tôi chỉ dùng LlamaIndex trong `backend/src/rag`; agent loop vẫn dùng Vercel AI SDK. Việc tách phạm vi tránh hai framework cùng điều phối agent và cho phép thay retrieval implementation nếu adapter TypeScript không đáp ứng.

**Kế hoạch dự phòng:** giữ LlamaIndex cho document loading/chunking, query pgvector bằng repository mỏng nếu `PGVectorStore` không ổn định.

### 13.7 “Tại sao PostgreSQL + pgvector, không dùng vector database chuyên dụng?”

> Dữ liệu demo có quy mô nhỏ; yêu cầu quan trọng hơn là transaction, audit và liên kết citation với task. PostgreSQL vừa lưu trạng thái orchestration vừa hỗ trợ vector search qua pgvector, giúp giảm một hệ thống phải vận hành và sao lưu. Ranh giới `backend/src/rag` cho phép chuyển sang Qdrant, Pinecone hoặc vector store nội bộ của SHB khi quy mô hay chính sách dữ liệu yêu cầu.

### 13.8 “Tại sao MCP server phải tách process? Gọi REST trực tiếp có đơn giản hơn không?”

> REST đơn giản hơn cho demo, nhưng MCP tạo một contract chuẩn để agent khám phá và gọi công cụ độc lập với framework/model. Tách từng hệ thống — LOS, Compliance, Core Banking — thành MCP server riêng mô phỏng đúng ranh giới hệ thống hiện hữu của ngân hàng, cô lập credential và policy, đồng thời cho phép triển khai on-prem từng connector. NestJS orchestration chỉ là MCP client, không biết chi tiết implementation của hệ thống đích.

Lưu ý: không tuyên bố MCP thay thế toàn bộ API ngân hàng. MCP là **adapter an toàn cho agent** nằm trước REST/gRPC/API hiện hữu.

### 13.9 “Tại sao Next.js thay vì React thuần?”

> Next.js vẫn là React, nhưng cung cấp routing, bundling và deployment convention sẵn có. Dashboard là frontend mỏng, chỉ gọi REST và nhận WebSocket từ NestJS; business logic agent không nằm trong Next.js. Chọn Next.js giảm cấu hình frontend mà vẫn đáp ứng đúng yêu cầu giao diện React.

### 13.10 “Tại sao dùng Tailwind trong khi Aucobot dùng CSS Modules?”

> `auco-team` là repo thi độc lập. Tailwind giúp dựng nhanh nhiều trạng thái trực quan của Dashboard như node status, timeline và approval panel. Đây là lựa chọn tối ưu tốc độ UI cho cuộc thi, không phải thay đổi tiêu chuẩn của sản phẩm Aucobot.

### 13.11 “Tại sao tách nhiều service, có đang over-engineer cho demo không?”

> Chúng tôi chỉ tách ở ranh giới có ý nghĩa triển khai: frontend, orchestration API và connector MCP của hệ thống ngân hàng. Repo không dùng monorepo packages/Turborepo — chỉ `frontend/` + `backend/` để team làm song song không đụng code. Bên trong API vẫn là modular monolith, chưa tách từng agent thành microservice. Cách này vừa đủ để chứng minh Docker, health check, contract và khả năng scale độc lập, nhưng không tạo distributed system không cần thiết.

Nếu thiếu thời gian, chỉ `mcp-los` và `mcp-compliance` được triển khai đầy đủ; các connector khác dùng cùng contract nhưng mock nông.

### 13.11b “Tại sao không dùng monorepo packages như Aucobot?”

> Aucobot cần shared packages vì sản phẩm dài hạn và nhiều app cùng schema. Với cuộc thi, mục tiêu là FE và BE làm song song rồi nối API. Shared TypeScript package buộc cả hai team cùng toolchain/workspace, tăng conflict và thời gian setup. Contract HTTP/WS trong `API.md` đủ để FE mock và BE implement độc lập; khi nối chỉ đổi base URL.

### 13.12 “Tại sao không làm authentication?”

> Authentication không nằm trong deliverable và không chứng minh năng lực multi-agent. Bản demo chạy trong môi trường cô lập với dữ liệu giả lập, nên nhóm chủ động bỏ auth để đầu tư vào planner, RAG, MCP, approval và trace. Đây không phải thiết kế production: khi tích hợp thật với SHB, authN/authZ, RBAC, service identity, secrets vault và audit theo người dùng là điều kiện bắt buộc trước khi cho phép side-effect.

### 13.13 “Kiến trúc này có khóa vào Together AI hay cloud công cộng không?”

> Không. Model provider nằm sau `backend/src/llm`; tool nằm sau MCP; retrieval nằm sau `backend/src/rag`. Có thể thay Together AI bằng model gateway hoặc model triển khai nội bộ SHB. Dữ liệu demo dùng cloud, nhưng production có thể đặt API, PostgreSQL, MCP connector và model endpoint trong hạ tầng ngân hàng mà không đổi contract cấp ứng dụng.

### 13.14 Câu kết khi bị hỏi dồn về stack

> Chúng tôi tối ưu cho **khả năng kiểm soát và khả năng thay thế**, không tối ưu để trình diễn số lượng framework. Phần có giá trị của giải pháp là plan có cấu trúc, collaboration có giới hạn, RAG có citation, action có approval và trace end-to-end. Stack hiện tại giúp nhóm chứng minh các năng lực đó ổn định trong thời gian thi, đồng thời giữ đường thay thế sang Python hoặc hạ tầng nội bộ SHB qua các contract chuẩn.

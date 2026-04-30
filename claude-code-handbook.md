# Claude Code Handbook - Tối ưu prompt, context và chi phí

> Cập nhật: 2026-04-30. Tài liệu này được viết lại từ bản gốc trong `archive/original-2026-04-30/`, có đối chiếu với tài liệu chính thức của Anthropic/Claude Code.

## Cách đọc nhanh

Nếu bạn chỉ có 20 phút:

1. Đọc [README.md](README.md) để nắm bản đồ tài liệu.
2. Đọc phần [2. Vòng lặp làm việc tối ưu](#2-vòng-lặp-làm-việc-tối-ưu).
3. Copy template trong [templates/claude-md-lean-template.md](templates/claude-md-lean-template.md) vào project thật.
4. Dùng checklist [checklists/daily-claude-code.md](checklists/daily-claude-code.md) trong 1 tuần.

Nếu bạn muốn giảm tiền/token:

1. Đọc [4. Context và chi phí](#4-context-và-chi-phí).
2. Thiết lập CLAUDE.md gọn.
3. Dùng `/clear` giữa các task khác nhau, `/compact` khi task dài nhưng vẫn cần tiếp tục.
4. Giới hạn MCP, hooks, file reads và output dài.
5. Nếu tài liệu lớn hoặc thay đổi thường xuyên, đọc [RAG và tối ưu context](#6-rag-và-tối-ưu-context).

---

## 1. Mental model đúng về Claude Code

Claude Code không phải chatbot hỏi-đáp. Nó là coding agent trong terminal, có thể đọc file, sửa file, chạy lệnh, dùng git, gọi MCP tools, dùng subagents và hooks.

### 1.1 Ba lớp cần tối ưu

| Lớp | Mục tiêu | Cách tối ưu |
|---|---|---|
| Prompt | Giao việc rõ, đúng phạm vi | Đưa goal, context, constraints, output format, verification |
| Context | Giữ đúng thông tin cần thiết trong cửa sổ làm việc | CLAUDE.md ngắn, search trước khi đọc file, clear/compact đúng lúc, dùng subagents cho research |
| Execution | Đảm bảo code đúng và ít vòng lặp | Plan nhỏ, edit theo scope, test liên quan, review diff, dùng checkpoint/git |

### 1.2 Điều nên nhớ

- Context lớn không đồng nghĩa với hiệu quả cao. Context quá lớn làm Claude dễ loãng trọng tâm.
- Prompt tốt không phải prompt dài. Prompt tốt là prompt làm rõ "đúng là gì" và "không được làm gì".
- CLAUDE.md là memory cố định, không phải nơi nhét toàn bộ tài liệu dự án.
- Dùng subagents cho việc đọc/radar codebase vì subagent có context riêng, chỉ trả kết quả tóm tắt về main session.
- Dùng automation cho việc lặp lại, nhưng hooks quá nhiều sẽ tốn token và làm workflow khó debug hơn.

---

## 2. Vòng lặp làm việc tối ưu

Đây là workflow mặc định cho hầu hết coding task:

```text
Explore -> Plan -> Implement -> Verify -> Review
```

### 2.1 Explore

Mục tiêu: biết đúng file, đúng contract, đúng risk trước khi edit.

Prompt mẫu:

```text
Hãy tìm hiểu codebase trước khi sửa.

Mục tiêu: [mô tả task]
Phạm vi: chỉ liên quan đến [module/file/route]

Hãy làm:
1. Tìm các file/call site liên quan bằng search trước.
2. Đọc chỉ những đoạn cần thiết.
3. Tóm tắt current behavior, proposed change, risk và test cần chạy.
4. Chưa sửa file ở bước này.
```

Khi dùng:

- Codebase mới.
- Chưa biết module nào đang sở hữu logic.
- Task có nguy cơ ảnh hưởng nhiều route/package.

### 2.2 Plan

Dùng plan mode khi thay đổi có nhiều bước hoặc có rủi ro. Plan tốt phải ngắn và có tiêu chí done.

Prompt mẫu:

```text
Lập plan ngắn trước khi implement.

Done criteria:
- [hành vi mong muốn]
- [test/build phải pass]
- [không thay đổi gì ngoài scope]

Hãy đưa plan 3-6 bước, nếu có assumption thì nói rõ.
Sau khi tôi approve thì mới edit.
```

Với task nhỏ, có thể bỏ plan dài và yêu cầu Claude làm thẳng:

```text
Sửa lỗi này trong scope hẹp nhất có thể. Nếu phải chạm file ngoài [X], dừng lại và nói lý do.
```

### 2.3 Implement

Prompt nên gắn với kết quả kiểm chứng được:

```text
Implement theo plan.

Ràng buộc:
- Giữ API/public behavior hiện tại trừ khi cần đổi.
- Không refactor unrelated code.
- Không thêm dependency mới nếu không thật sự cần.
- Sau khi sửa, chạy test liên quan và tóm tắt diff.
```

### 2.4 Verify

Claude Code mạnh nhất khi có feedback vòng lặp từ tool thực tế.

Lệnh nên yêu cầu tùy project:

- `npm test`, `pnpm test`, `pytest`, `go test ./...`
- `npm run typecheck`, `pnpm build`, `ruff`, `mypy`, `cargo test`
- `git diff --check`

Prompt mẫu:

```text
Hãy verify thay đổi:
1. Chạy test/build liên quan.
2. Nếu fail, phân tích lỗi và sửa trong scope task.
3. Nếu không chạy được, nói rõ lý do và còn rủi ro gì.
4. Cuối cùng tóm tắt file đã sửa và behavior thay đổi.
```

### 2.5 Review

Cuối task nên bắt Claude review lại chính diff của nó:

```text
Review lại diff như code reviewer khó tính.
Tập trung vào bug, regression, missing tests, edge cases.
Nếu không thấy issue, nói rõ test gap còn lại.
```

---

## 3. Prompt engineering cho Claude Code

### 3.1 Công thức prompt thực dụng

Dùng cấu trúc này cho 80% task:

```text
<goal>
[Kết quả cần đạt]
</goal>

<context>
[Thông tin domain, file, user flow, bug report, log, constraint kinh doanh]
</context>

<scope>
- Chỉ sửa: [đường dẫn/module]
- Không sửa: [những gì cấm đụng]
</scope>

<acceptance_criteria>
- [Điều kiện 1]
- [Điều kiện 2]
- [Test/build/check]
</acceptance_criteria>

<working_rules>
- Search/doc code trước khi edit.
- Giữ diff nhỏ.
- Nếu assumption không chắc, nêu rõ trước khi tiếp tục.
</working_rules>
```

XML tags giúp tách rõ các khối thông tin, nhất là khi prompt dài. Không cần làm màu mè, chỉ cần nhất quán.

### 3.2 Nên đưa "why", không chỉ "what"

Prompt yếu:

```text
Tối ưu trang checkout.
```

Prompt tốt hơn:

```text
Tối ưu trang checkout để giảm tỷ lệ user bỏ giỏ hàng trên mobile.
Mục tiêu là làm flow rõ hơn, không thay đổi payment API.
Tập trung vào validation message, loading state và lỗi network.
```

Lý do giúp Claude chọn tradeoff đúng hơn.

### 3.3 Prompt theo loại việc

| Việc | Prompt nên dùng |
|---|---|
| Tìm hiểu codebase | "Map current behavior, entrypoints, data flow. Chưa edit." |
| Sửa bug | "Reproduce/trace root cause trước, sửa scope hẹp, thêm regression test nếu hợp lý." |
| Thêm feature | "Plan API/UI/data changes, implement incrementally, verify acceptance criteria." |
| Refactor | "Behavior must stay identical. Nếu test không cover, đề xuất safety check trước." |
| Review PR | "Find bugs first. Severity, file/line, reproduction, suggested fix." |
| Viết docs | "Preserve source truth, restructure for navigation, add examples/checklists." |

### 3.4 Negative constraints phải cụ thể

Không nên:

```text
Đừng làm lung tung.
```

Nên:

```text
Không đổi database schema.
Không thêm dependency mới.
Không sửa generated files.
Không chạy migration lên production.
Không rewrite component nếu chỉ cần sửa validation.
```

### 3.5 Khi nào cần yêu cầu Claude hỏi lại?

Chỉ bắt Claude hỏi lại khi assumption sai có thể gây hại:

- Xóa data, migration, payment, auth, security.
- Thay đổi public API.
- Chọn direction sản phẩm/design chưa rõ.
- Cần credential, secret hoặc permission ngoài local.

Với task bình thường, nên để Claude tự inspect và quyết định trong scope.

---

## 4. Context và chi phí

### 4.1 Context gồm những gì?

Một session gồm:

- System instructions và tool definitions.
- CLAUDE.md, `.claude/rules/`, auto memory nếu có.
- Conversation history.
- Kết quả tool calls: file reads, grep output, test logs, diff.
- MCP tool definitions nếu bật MCP servers.

Phần phình nhanh nhất thường là conversation history và tool outputs.

### 4.2 Quy tắc đọc file tiết kiệm

Nên:

```text
Search trước bằng rg/grep, đọc chỉ đoạn liên quan, tóm tắt lại.
```

Không nên:

```text
Đọc toàn bộ src/ hoặc paste log 5,000 dòng vào prompt.
```

Prompt mẫu:

```text
Hãy tìm file liên quan bằng search trước.
Chỉ đọc những function/component cần sửa.
Nếu cần đọc file lớn, hãy đọc theo section và nói lý do.
```

### 4.3 `/clear`, `/compact`, `--continue`, `--resume`

| Lệnh | Dùng khi | Lợi ích |
|---|---|---|
| `/clear` | Chuyển sang task khác | Giảm context rot, tiết kiệm token |
| `/compact [focus]` | Task dài vẫn cần tiếp tục | Giữ tóm tắt quan trọng, bỏ bớt lịch sử thừa |
| `claude --continue` | Quay lại session gần nhất | Không cần giải thích lại |
| `claude --resume` | Chọn session cũ | Tách workstream theo task |

Compact prompt nên có focus:

```text
/compact Focus on current bug root cause, files edited, failing tests, and next verification steps.
```

### 4.4 Khi nào context đang kém chất lượng

Dấu hiệu:

- Claude quên yêu cầu đã nói.
- Đọc nhầm file hoặc lặp lại lỗi cũ.
- Sửa quá rộng so với scope.
- Output dài nhưng ít tiến triển.

Cách xử lý:

1. Yêu cầu Claude tóm tắt state hiện tại: files, decisions, next step.
2. Chạy `/compact` với focus rõ.
3. Nếu chuyển task, dùng `/clear`.
4. Nếu đã rối, bắt đầu session mới với prompt tổng hợp ngắn.

### 4.5 Cost tactics thực tế

- Dùng Sonnet/default cho coding hằng ngày; chỉ dùng model cao hơn cho architecture/risk lớn nếu subscription/API của bạn hỗ trợ.
- Dùng subagent cho research đọc nhiều file, main session chỉ nhận summary.
- Dùng `/clear` giữa các task độc lập.
- Viết CLAUDE.md dưới 80-120 dòng, link/import tài liệu dài thay vì paste hết.
- Giới hạn MCP servers đang bật. Mỗi server thêm tools vào context.
- Yêu cầu output ngắn: "chỉ tóm tắt diff và test result".
- Dùng non-interactive `claude -p` cho task lặp lại và output có format.
- Theo dõi bằng `/cost` nếu dùng API/Console billing; với Pro/Max, `/cost` không phải thước đo billing chính xác.

---

## 5. CLAUDE.md, rules và memory

### 5.1 CLAUDE.md nên chứa gì?

Nên chứa:

- Project purpose.
- Architecture decisions không tự suy ra được từ code.
- Commands dùng để dev/test/build.
- Coding conventions có tính bắt buộc.
- Security/safety rules.
- Output/review preference.

Không nên chứa:

- Toàn bộ README.
- API docs dài.
- Danh sách file quá chi tiết.
- Nội dung hay thay đổi mỗi ngày.
- Secret, token, credential.

### 5.2 Cấu trúc gọn

```markdown
# Project
[1-3 dòng nói project là gì, user là ai]

## Commands
- Install:
- Dev:
- Test:
- Build:

## Architecture
- [decision quan trọng]
- [boundary quan trọng]

## Rules
- YOU MUST [bắt buộc 1]
- NEVER [cấm 1]

## Workflow
- Search before editing.
- Keep diffs scoped.
- Run relevant tests before final.
```

Xem template đầy đủ: [templates/claude-md-lean-template.md](templates/claude-md-lean-template.md).

### 5.3 `.claude/rules/`

Nếu project lớn, tách rule theo scope thay vì nhồi CLAUDE.md:

```text
.claude/rules/
  frontend.md
  api.md
  database.md
  security.md
```

Dùng cho:

- Rule theo folder/module.
- Rule có thể lazy-load khi Claude đọc file liên quan.
- Convention dài hơn CLAUDE.md chính.

### 5.4 Auto memory

Auto memory giúp Claude ghi lại learning qua session nếu version hỗ trợ. Vẫn nên xem nó là "ghi chú phụ", không thay thế CLAUDE.md:

- CLAUDE.md: rule và context bạn chủ động kiểm soát.
- Auto memory: note tương lai có ích, có thể cần review/prune.

### 5.5 Import syntax

CLAUDE.md có thể import file khác:

```markdown
@README.md
@docs/architecture.md
@~/.claude/my-personal-preferences.md
```

Chỉ import tài liệu thật sự cần mỗi session. Tài liệu dài nên để Claude đọc khi cần.

---

## 6. RAG và tối ưu context

### 6.1 Khi nào cần RAG?

RAG, viết tắt của Retrieval-Augmented Generation, là cách để Claude truy xuất đúng phần kiến thức cần thiết từ một kho tài liệu rồi chỉ đưa phần đó vào prompt. Mục tiêu không phải là "cho Claude biết tất cả", mà là "cho Claude đúng bằng chứng cần để trả lời hoặc hành động".

Quyết định nhanh:

| Tình huống | Cách làm nên ưu tiên |
|---|---|
| Tài liệu nhỏ, ổn định, dưới context window | Đưa trực tiếp vào prompt hoặc Project knowledge, tận dụng prompt caching |
| Tài liệu vừa, cần hỏi nhiều lần | Chia file rõ, đặt tên tốt, dùng Claude Projects/RAG tự động nếu dùng Claude.ai |
| Tài liệu lớn hoặc thay đổi thường xuyên | Build RAG riêng hoặc MCP retrieval tool |
| Query cần mã lỗi, tên API, ID chính xác | Kết hợp BM25/keyword search với embeddings |
| Cần trích dẫn nguồn | Trả kết quả retrieval kèm source/title/citation |
| Cần dùng trong Claude Code | Tạo MCP/search command trả top chunks thay vì paste cả docs |

RAG không thay thế context management. RAG tốt là một lớp lọc trước context.

### 6.2 Kiến trúc RAG tối thiểu

```text
Documents
  -> clean/normalize
  -> chunk
  -> contextualize chunk
  -> embed + BM25 index
  -> vector DB/search index
  -> retrieve top-N
  -> rerank
  -> pass top-K chunks with source to Claude
  -> answer/action with citations
```

Các thành phần:

- Corpus: docs, README, ADR, tickets, API specs, logs, runbooks.
- Chunker: chia tài liệu theo heading/section/code block, không cắt giữa ý.
- Contextualizer: thêm 50-100 tokens mô tả chunk thuộc tài liệu nào, phần nào, version nào.
- Retriever: semantic embeddings + BM25 để vừa bắt nghĩa, vừa bắt exact match.
- Reranker: lọc top-N xuống top-K để giảm noise.
- Prompt assembler: đưa chunk vào prompt có source rõ ràng.
- Eval set: bộ câu hỏi thật để đo retrieval có đúng không.

### 6.3 Setup RAG cho Claude Code

Có 3 mức setup:

**Mức 1: Không cần build RAG riêng**

- Dùng `rg`, `find`, README, docs và prompt search-first trong Claude Code.
- Phù hợp khi tài liệu nằm trong repo, quy mô nhỏ-vừa.
- Tối ưu bằng CLAUDE.md gọn và yêu cầu Claude chỉ đọc section liên quan.

**Mức 2: Local retrieval command**

Tạo script nội bộ như:

```bash
./scripts/search-knowledge "payment retry policy"
```

Script trả về:

```text
Source: docs/payments/retry-policy.md#timeouts
Title: Payment retry policy
Score: 0.87
Content:
[chunk liên quan]
```

Sau đó prompt Claude:

```text
Trước khi trả lời, hãy dùng ./scripts/search-knowledge để lấy context liên quan.
Chỉ dùng top 5 chunks. Nếu nguồn không đủ, nói rõ thiếu gì.
```

**Mức 3: MCP retrieval server**

Dùng khi muốn Claude gọi retrieval như tool:

```text
Claude Code
  -> MCP tool: search_knowledge(query, filters)
  -> Retriever service
  -> Vector DB + BM25
  -> top chunks with source/title/content
```

MCP phù hợp khi knowledge nằm ngoài repo: Google Drive, Notion, Confluence, database, ticket system, internal docs.

### 6.4 Chunking tốt

Rule of thumb:

- Chunk theo heading/section trước, theo số token sau.
- Chunk vừa đủ để tự hiểu: thường vài trăm đến khoảng 800 tokens tùy domain.
- Có overlap nhỏ nếu ý kéo dài qua section.
- Giữ metadata: `source`, `title`, `heading_path`, `updated_at`, `version`, `owner`, `tags`.
- Không embed boilerplate lặp lại như nav/footer/license dài.
- Với code docs, giữ code block liền với giải thích gần nhất.

Chunk tệ:

```text
"The default timeout is 30 seconds."
```

Chunk tốt hơn:

```text
Source: docs/payments/retry-policy.md
Heading: Payment gateway -> Timeout and retry
Context: Áp dụng cho checkout service v2 từ 2026-01.
Content: The default timeout is 30 seconds...
```

### 6.5 Contextual Retrieval

Vấn đề lớn của RAG truyền thống là chunk bị mất ngữ cảnh sau khi bị cắt khỏi tài liệu gốc. Contextual Retrieval giải quyết bằng cách prepend một đoạn context ngắn vào mỗi chunk trước khi embed và index BM25.

Prompt contextualize mẫu:

```text
<document>
{{WHOLE_DOCUMENT}}
</document>

<chunk>
{{CHUNK_CONTENT}}
</chunk>

Hãy viết 1-3 câu ngắn để đặt chunk này vào đúng ngữ cảnh của tài liệu:
- chunk thuộc tài liệu/phần nào
- nói về entity/version/feature nào
- vì sao chunk này quan trọng cho retrieval

Chỉ trả về context ngắn, không giải thích thêm.
```

Sau đó lưu:

```text
contextualized_chunk = contextual_note + "\n\n" + original_chunk
```

Contextual note giúp query như "timeout checkout v2" tìm được chunk thay vì chỉ thấy câu rời rạc "default timeout is 30 seconds".

### 6.6 Retrieval strategy

Không nên chỉ dùng embeddings. Với tài liệu kỹ thuật, exact match rất quan trọng.

Pipeline khuyến nghị:

1. Rewrite query nếu cần: thêm synonyms, product/module name.
2. Retrieve top 50-150 bằng embeddings.
3. Retrieve top 50-150 bằng BM25/keyword.
4. Merge + deduplicate bằng source/chunk id.
5. Rerank theo query.
6. Chỉ đưa top 5-20 chunks vào Claude tùy độ phức tạp.

Top-K càng lớn không phải lúc nào càng tốt. Nhiều chunk hơn tăng cơ hội có bằng chứng đúng, nhưng cũng tăng noise và chi phí.

### 6.7 Prompt dùng RAG

Prompt tốt phải bắt Claude bám nguồn:

```text
Bạn sẽ nhận một số search results từ knowledge base.

Rules:
- Chỉ dùng thông tin trong search results và codebase hiện tại.
- Nếu search results thiếu bằng chứng, nói "chưa đủ nguồn" và nêu cần tìm gì.
- Khi trả lời quyết định kỹ thuật, trích source path/title.
- Không suy đoán policy/version nếu không có nguồn.

Question:
[câu hỏi/task]

Search results:
[top-K chunks with source/title/content]
```

Với Claude API, nên dùng search result content blocks khi cần citation tự nhiên. Với Claude Code, mô phỏng bằng format source/title/content nhất quán cũng đã giúp giảm hallucination.

### 6.8 Tối ưu context khi dùng RAG

- Chỉ inject chunks cho câu hỏi hiện tại, không giữ toàn bộ docs trong session.
- Dùng top-K nhỏ trước, tăng dần nếu thiếu bằng chứng.
- Tóm tắt retrieval result thành "facts with sources" trước khi implement.
- Sau khi Claude dùng xong, `/compact` nếu session dài để giữ facts/decisions, không giữ raw chunks.
- Không đưa cùng lúc chunks trùng ý từ nhiều nguồn nếu không cần so sánh.
- Ưu tiên source mới hơn hoặc authoritative hơn.
- Với docs ổn định, cache/index trước; với docs thay đổi, lưu `updated_at` và re-index incremental.

### 6.9 RAG evals

Không có eval thì RAG chỉ là cảm giác.

Tạo file nhỏ:

```text
evals/rag-questions.yml
- question: "Timeout mặc định của checkout v2 là bao nhiêu?"
  expected_sources:
    - docs/payments/retry-policy.md#timeouts
  expected_facts:
    - "default timeout is 30 seconds"
```

Đo:

- Retrieval recall: source đúng có nằm trong top-K không?
- Answer faithfulness: câu trả lời có bám source không?
- Citation quality: citation có đúng đoạn không?
- Latency/cost: query mất bao lâu, đưa bao nhiêu tokens vào prompt?

### 6.10 RAG anti-patterns

| Anti-pattern | Tác hại | Cách sửa |
|---|---|---|
| Chunk quá nhỏ | Mất ngữ cảnh | Contextual note + chunk theo heading |
| Chỉ dùng embeddings | Miss mã lỗi, tên API, ID | Thêm BM25/keyword |
| Đưa quá nhiều chunks | Noise, tốn token | Rerank, top-K nhỏ |
| Không lưu source | Không kiểm chứng được | Metadata bắt buộc |
| Không có eval | Không biết retrieval đúng hay sai | Tạo 20-50 câu hỏi thật |
| Index docs lỗi thời | Trả lời sai version | `updated_at`, re-index, owner |
| Nhồi RAG result vào CLAUDE.md | Tốn context mọi session | Retrieve theo query |

Xem template setup: [templates/rag-setup-template.md](templates/rag-setup-template.md). Xem deep dive: [deep-dives/rag-context-optimization.md](deep-dives/rag-context-optimization.md).

---

## 7. Subagents, MCP, hooks và skills

### 7.1 Subagents

Dùng subagents khi:

- Cần research nhiều file mà không muốn làm phình main context.
- Cần role riêng: reviewer, security auditor, migration planner.
- Cần parallel exploration trên nhiều module.

Agent description phải rõ để Claude biết khi nào delegate:

```markdown
---
name: api-reviewer
description: Use when reviewing backend API changes for breaking changes, auth bugs, validation gaps, and missing tests.
tools: Read, Grep, Glob, Bash
---

You are a strict API reviewer. Find bugs first. Do not edit files.
Return findings with severity, file path, and suggested fix.
```

### 7.2 MCP

MCP dùng để kết nối service ngoài: GitHub, database, Figma, Slack, Sentry, browser, internal tools.

Quy tắc:

- Bật ít server nhất có thể.
- Không dùng MCP cho việc Claude Code đã làm được trong local repo.
- Scope credential tối thiểu.
- Prefer read-only token cho research/review.
- Tắt server không dùng trong session để tiết kiệm context.

### 7.3 Hooks

Hooks tốt cho việc deterministic:

- Block lệnh nguy hiểm.
- Chạy formatter/test liên quan sau edit.
- Cảnh báo secret.
- Gửi notification.
- Inject context ngắn khi session start.

Không nên dùng hook để thay thế reasoning. Hook quá phức tạp khó debug và có thể làm Claude nhận thêm nhiều context thừa.

### 7.4 Skills

Skills phù hợp cho workflow lặp lại có tri thức riêng:

- Deploy.
- Incident response.
- Design review.
- Data migration.
- Release notes.

Skill nên có:

- Frontmatter mô tả dùng khi nào trigger.
- Hướng dẫn ngắn.
- Scripts/references riêng, chỉ load khi cần.

---

## 8. Setup project chuẩn

### 8.1 Khởi tạo

```bash
cd your-project
git status
claude
/init
```

Sau `/init`, dùng template gọn để prune CLAUDE.md. Dùng git làm safety net.

### 8.2 Settings nên có

```json
{
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Read(./config/credentials.json)",
      "Bash(rm -rf /)",
      "Bash(sudo:*)"
    ]
  }
}
```

Ghi chú:

- `.claude/settings.json`: commit cho team.
- `.claude/settings.local.json`: cá nhân, không commit.
- Enterprise có thể có managed policy riêng.

### 8.3 Thư mục đề xuất

```text
project/
  CLAUDE.md
  .claude/
    settings.json
    settings.local.json
    agents/
      api-reviewer.md
      frontend-reviewer.md
    rules/
      frontend.md
      api.md
    commands/
      review.md
      release-notes.md
  docs/
```

---

## 9. Workflows mẫu

### 9.1 Onboard codebase mới

```text
Hãy onboard codebase này.
Mục tiêu: tôi cần hiểu architecture và workflow dev.

Hãy:
1. Tìm package/build files.
2. Map main entrypoints, test commands, deployment hints.
3. Tìm CLAUDE.md/README/docs có sẵn.
4. Tạo summary ngắn: architecture, commands, risks, next questions.
Chưa edit file.
```

### 9.2 Sửa bug

```text
Bug: [mô tả]
Expected: [hành vi đúng]
Actual: [hành vi sai]
Evidence: [log/screenshot/test]

Hãy trace root cause trước, sửa scope hẹp nhất, thêm regression test nếu hợp lý, rồi chạy test liên quan.
```

### 9.3 Thêm feature

```text
Feature: [mô tả]
User flow: [ai làm gì]
Acceptance criteria:
- ...
- ...

Hãy inspect current patterns, plan ngắn, implement theo pattern hiện có, verify bằng test/build.
Không thêm dependency mới nếu không cần.
```

### 9.4 Code review

```text
Review diff hiện tại.
Findings first, ordered by severity.
Tập trung vào bug, regression, security, missing tests.
Không nitpick style nếu không ảnh hưởng behavior.
```

### 9.5 Refactor

```text
Refactor [module] để [mục tiêu].
Behavior must stay identical.
Trước khi sửa, tìm test hiện có. Nếu coverage yếu, đề xuất safety check nhỏ.
Sau khi sửa, chạy test liên quan và so sánh public behavior.
```

### 9.6 Automation non-interactive

```bash
claude -p "Review this diff for bugs only" < diff.txt
claude -p "Generate release notes from these commits" --output-format json
claude -p "Summarize this log and flag anomalies" < app.log
```

---

## 10. Anti-patterns cần tránh

| Anti-pattern | Tác hại | Cách sửa |
|---|---|---|
| Paste toàn bộ codebase/log | Tốn token, loãng trọng tâm | Search/đọc section liên quan |
| CLAUDE.md quá dài | Rule quan trọng bị chìm | Prune, import file khi cần |
| Một session làm nhiều task khác nhau | Context rot | `/clear` giữa task |
| "Làm đẹp hơn" không có criteria | Output lệch ý | Đưa audience, constraints, examples |
| Cho Claude refactor khi chỉ cần fix bug | Diff lớn, regression | Scope hẹp, behavior must stay identical |
| Bật quá nhiều MCP | Tăng tool/context overhead | Chỉ bật server cần dùng |
| Không verify | Lỗi ẩn trong diff | Chạy test/build/diff review |
| Tin summary thay vì diff | Bỏ sót thay đổi ngoài ý | Xem `git diff` |

---

## 11. Checklist nhanh

Trước session:

- Repo sạch hoặc biết rõ file nào đang dirty.
- CLAUDE.md ngắn và đúng.
- Secret nằm trong deny rules.
- Prompt có goal, scope, acceptance criteria.

Trong session:

- Search trước khi đọc file.
- Plan nếu task có rủi ro.
- Giữ diff nhỏ.
- Compact khi session dài.
- Clear khi chuyển task.

Trước khi kết thúc:

- Chạy test/build liên quan.
- Review diff.
- Ghi rõ test nào đã chạy, test nào chưa.
- Nếu cần tiếp tục sau, rename/resume session hoặc để lại summary.

---

## 12. Tài liệu chuyên sâu trong repo này

- [deep-dives/context-window-mechanics.md](deep-dives/context-window-mechanics.md)
- [deep-dives/prompt-caching.md](deep-dives/prompt-caching.md)
- [deep-dives/claude-md-patterns.md](deep-dives/claude-md-patterns.md)
- [deep-dives/hook-scripting.md](deep-dives/hook-scripting.md)
- [deep-dives/mcp-servers-guide.md](deep-dives/mcp-servers-guide.md)
- [deep-dives/extended-thinking.md](deep-dives/extended-thinking.md)
- [deep-dives/rag-context-optimization.md](deep-dives/rag-context-optimization.md)

Ghi chú: các deep dive cũ đã được giữ lại, nhưng khi có xung đột với handbook mới và official docs, ưu tiên handbook mới + [references/research-sources.md](references/research-sources.md).

---

## 13. Sources

Danh sách research và những điểm đã cập nhật nằm trong [references/research-sources.md](references/research-sources.md).

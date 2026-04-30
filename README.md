# Claude Code Knowledge System

Tài liệu này giúp bạn dùng Claude Code tối ưu hơn về prompt, context, chi phí và workflow code thực tế.

## Nên bắt đầu từ đâu?

1. [claude-code-handbook.md](claude-code-handbook.md) - playbook chính, đọc từ đầu đến cuối.
2. [checklists/daily-claude-code.md](checklists/daily-claude-code.md) - checklist dùng mỗi ngày.
3. [templates/claude-md-lean-template.md](templates/claude-md-lean-template.md) - template CLAUDE.md ngắn gọn cho project.
4. [templates/prompt-patterns.md](templates/prompt-patterns.md) - prompt mẫu theo loại việc.
5. [deep-dives/rag-context-optimization.md](deep-dives/rag-context-optimization.md) - RAG, setup retrieval và tối ưu context.
6. [templates/rag-setup-template.md](templates/rag-setup-template.md) - template thiết kế RAG cho project.
7. [references/research-sources.md](references/research-sources.md) - nguồn research và điểm cần lưu ý.

## Bản đồ tài liệu

```text
.
|-- README.md
|-- claude-code-handbook.md
|-- checklists/
|   `-- daily-claude-code.md
|-- templates/
|   |-- claude-md-lean-template.md
|   |-- prompt-patterns.md
|   `-- rag-setup-template.md
|-- references/
|   `-- research-sources.md
|-- deep-dives/
|   |-- README.md
|   |-- claude-md-patterns.md
|   |-- context-window-mechanics.md
|   |-- extended-thinking.md
|   |-- hook-scripting.md
|   |-- mcp-servers-guide.md
|   |-- prompt-caching.md
|   `-- rag-context-optimization.md
`-- archive/
    `-- original-2026-04-30/
```

## Cách học nhanh trong 7 ngày

| Ngày | Việc cần làm | Output |
|---|---|---|
| 1 | Đọc handbook phần 1-4 | Nắm mental model prompt-context-cost |
| 2 | Viết lại CLAUDE.md bằng template lean | Project memory gọn hơn |
| 3 | Dùng prompt patterns cho 3 task thật | Prompt có goal/scope/criteria |
| 4 | Luyện `/clear`, `/compact`, `--resume` | Session sạch và rẻ hơn |
| 5 | Tạo 1 subagent reviewer hoặc explorer | Giảm phình context main |
| 6 | Thêm deny rules cho secrets/lệnh nguy hiểm | An toàn hơn |
| 7 | Thiết kế RAG/context strategy cho tài liệu lớn | Không nhồi tài liệu vào prompt |

## Nguyên tắc của bộ tài liệu này

- Thực dụng hơn lý thuyết.
- Giữ prompt ngắn nhưng đủ tiêu chí thành công.
- Tối ưu context trước khi tối ưu model.
- Verification là một phần của prompt, không phải việc phụ.
- Tài liệu dài để trong deep dive; CLAUDE.md chỉ giữ rule cần mỗi session.

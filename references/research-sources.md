# Research Sources và Cập Nhật

Research ngày 2026-04-30. Ưu tiên official docs của Anthropic/Claude Code.

## Nguồn chính

| Chủ đề | Link |
|---|---|
| Claude Code best practices | https://code.claude.com/docs/en/best-practices |
| Memory, CLAUDE.md, `.claude/rules/`, auto memory | https://code.claude.com/docs/en/memory |
| Subagents | https://code.claude.com/docs/en/sub-agents |
| Settings và permissions | https://code.claude.com/docs/en/settings |
| Hooks reference | https://code.claude.com/docs/en/hooks |
| Cost management | https://code.claude.com/docs/en/costs |
| Slash commands | https://docs.anthropic.com/en/docs/claude-code/slash-commands |
| Prompting best practices | https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices |
| Prompt caching | https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching |
| Pricing | https://docs.anthropic.com/en/docs/about-claude/pricing |

## Những điểm đã chỉnh trong handbook mới

1. Thêm mental model prompt-context-execution thay vì chỉ liệt kê feature.
2. Đưa workflow Explore -> Plan -> Implement -> Verify -> Review lên đầu tài liệu.
3. Cập nhật memory theo docs mới: CLAUDE.md, CLAUDE.local.md, `.claude/rules/`, auto memory, import syntax.
4. Chuyển `.claudeignore` thành hướng dẫn ưu tiên `permissions.deny` cho sensitive files, theo settings docs.
5. Cập nhật hooks: hook events hiện tại nhiều hơn PreToolUse/PostToolUse/UserPromptSubmit/Stop.
6. Thêm cách dùng `/clear`, `/compact [focus]`, `--continue`, `--resume` như chiến lược context.
7. Nhấn mạnh subagents như cách giảm context cho exploration/review.
8. Bớt các model name/version quá cụ thể để tránh lỗi thời. Khuyến nghị dùng model picker/current docs cho lựa chọn cụ thể.
9. Thêm checklist và prompt templates để dùng hằng ngày.

## Ghi chú về những nội dung có thể thay đổi

- Model names, pricing và subscription behavior thay đổi thường xuyên. Khi cần tính chi phí thật, kiểm tra trang Pricing và `/cost`/Console của tài khoản hiện tại.
- Hook event schema và settings fields có thể thay đổi theo version Claude Code. Chạy `claude doctor` và đọc docs hiện tại trước khi viết automation quan trọng.
- Prompt caching trong API có TTL 5 phút và tùy chọn 1 giờ, nhưng trong Claude Code bạn thường tối ưu bằng cách giữ context đầu ổn định, compact/clear đúng lúc, và tránh tool/context thừa.

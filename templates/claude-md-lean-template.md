# CLAUDE.md Lean Template

Copy template này vào root project rồi điền ngắn gọn. Mục tiêu là đủ thông tin để Claude làm đúng, không biến CLAUDE.md thành wiki.

```markdown
# Project: [tên project]

## Purpose
[1-3 dòng: project làm gì, user/chức năng chính là ai]

## Stack
- Runtime:
- Framework:
- Database:
- Package manager:
- Deploy:

## Commands
- Install: `[command]`
- Dev: `[command]`
- Test: `[command]`
- Typecheck/lint: `[command]`
- Build: `[command]`

## Architecture notes
- [Decision/rule mà code không tự nói rõ]
- [Module boundary quan trọng]
- [External API/backend contract cần giữ]

## Code conventions
- [Convention bắt buộc 1]
- [Convention bắt buộc 2]
- [Testing convention]

## Safety rules
- YOU MUST search existing patterns before adding new abstraction.
- YOU MUST keep changes scoped to the requested behavior.
- YOU MUST run relevant verification before final when possible.
- NEVER read or print secrets from `.env*`, `secrets/`, or credential files.
- NEVER change public API/database schema without explicit approval.

## Communication
- Answer in [Vietnamese/English].
- Lead with result, then files changed and verification.
- If tests cannot run, explain the blocker and residual risk.

## Compact summary instructions
When compacting, preserve:
- User goal and acceptance criteria.
- Files edited and key decisions.
- Failing/passing tests.
- Remaining next steps.
```

## Khi nào tách ra `.claude/rules/`

Nếu CLAUDE.md vượt khoảng 80-120 dòng, tách các phần theo scope:

```text
.claude/rules/
  frontend.md
  api.md
  database.md
  security.md
```

Rule tốt:

- Cụ thể.
- Có ví dụ đúng/sai nếu convention dễ nhầm.
- Áp dụng cho một scope rõ.

Rule kém:

- "Follow best practices."
- "Write clean code."
- Danh sách dài nhưng không có ưu tiên.

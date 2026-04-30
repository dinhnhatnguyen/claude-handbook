# Claude Code Handbook — Hướng dẫn toàn diện

> **Mục đích**: Tổng hợp kiến thức sử dụng Claude Code tối ưu — từ cài đặt, cấu hình project, viết prompt, quản lý context, chọn model, đến skills, hooks, MCP, và team workflows. Biên soạn dựa trên tài liệu chính thức Anthropic và best practices cộng đồng (cập nhật 04/2026).

---

## Mục lục

1. [Tổng quan Claude Code](#1-tổng-quan-claude-code)
2. [Setup project từ đầu](#2-setup-project-từ-đầu)
3. [CLAUDE.md — Bộ nhớ cố định của project](#3-claudemd--bộ-nhớ-cố-định-của-project)
4. [Skills — Mở rộng năng lực Claude](#4-skills--mở-rộng-năng-lực-claude)
5. [Subagents, Hooks, MCP, Plugins](#5-subagents-hooks-mcp-plugins)
6. [Git Worktrees và Parallel Development](#6-git-worktrees-và-parallel-development)
7. [Chọn Model và Cấu hình Model](#7-chọn-model-và-cấu-hình-model)
8. [Extended Thinking — Reasoning sâu](#8-extended-thinking--reasoning-sâu)
9. [Quản lý Context Window](#9-quản-lý-context-window)
10. [Tối ưu Token và Chi phí](#10-tối-ưu-token-và-chi-phí)
11. [Prompt Engineering cho Claude](#11-prompt-engineering-cho-claude)
12. [Workflow gen code trong dự án](#12-workflow-gen-code-trong-dự-án)
13. [IDE Integration và Keyboard Shortcuts](#13-ide-integration-và-keyboard-shortcuts)
14. [Team Collaboration](#14-team-collaboration)
15. [CI/CD và Automation nâng cao](#15-cicd-và-automation-nâng-cao)
16. [Security Best Practices](#16-security-best-practices)
17. [Debugging và Troubleshooting](#17-debugging-và-troubleshooting)
18. [Checklist và Anti-patterns](#18-checklist-và-anti-patterns)
19. [Tài nguyên tham khảo](#19-tài-nguyên-tham-khảo)

---

## 1. Tổng quan Claude Code

**Claude Code** là CLI agentic chạy trong terminal, hiểu codebase, và thực thi tác vụ phức tạp qua ngôn ngữ tự nhiên. Khác biệt so với chat thông thường:

- Đọc và ghi file trực tiếp
- Chạy lệnh shell và git
- Tích hợp MCP server (database, API, Figma, Jira...)
- Spawn subagents trong context window riêng biệt
- Chạy hooks tự động (lint, test, format)
- Duy trì memory qua các session

### 1.1 Ba sản phẩm Claude — Phân biệt rõ

| Sản phẩm | Mục đích | Truy cập |
|---|---|---|
| Claude.ai | Chat trên web và mobile | claude.ai |
| Claude API | Build application dùng Claude | docs.anthropic.com |
| Claude Code | Coding agent trong terminal và IDE | `npm install -g @anthropic-ai/claude-code` |

### 1.2 CLI flags quan trọng

```bash
# Interactive mode (mặc định)
claude

# Non-interactive: chạy một lệnh rồi thoát
claude --print "Tóm tắt file này" < README.md
claude -p "Review diff này" < diff.txt

# Tiếp tục session gần nhất
claude --continue
claude -c

# Resume session theo ID
claude --resume SESSION_ID

# Bỏ qua permission prompts (chỉ dùng trong môi trường tin cậy)
claude --dangerously-skip-permissions

# Chỉ định model
claude --model claude-sonnet-4-6
claude --model sonnet

# Chạy trong thư mục khác
claude --cd /path/to/project

# Output JSON
claude --output-format json --print "..."

# Xem version và help
claude --version
claude --help
```

### 1.3 Permission modes

| Mode | Mô tả | Dùng khi |
|---|---|---|
| Default | Hỏi trước mỗi action nhạy cảm | Daily development |
| `--dangerously-skip-permissions` | Bỏ qua toàn bộ permission prompts | Trusted CI/CD pipeline |
| Plan mode (Shift+Tab x2) | Chỉ plan, không execute | Review trước khi tốn token |

### 1.4 Architecture tổng quan

```
User
  |
  v
Claude Code CLI
  |-- Main Agent (Sonnet / Opus)
  |     |-- Read / Edit / Write (local files)
  |     |-- Bash (shell commands)
  |     |-- MCP tools (external services)
  |     `-- Spawn subagents
  |
  |-- Subagents (isolated context window)
  |     |-- Explore (Haiku, read-only)
  |     |-- Plan (Sonnet)
  |     `-- Custom agents
  |
  `-- Hooks (deterministic shell scripts)
        |-- PreToolUse
        |-- PostToolUse
        |-- UserPromptSubmit
        |-- Stop
        `-- Notification
```

---

## 2. Setup project từ đầu

### 2.1 Cài đặt

```bash
# Native installer (khuyến nghị — không cần Node.js)
curl -fsSL https://claude.ai/install.sh | sh

# Qua npm
npm install -g @anthropic-ai/claude-code

# Kiểm tra
claude --version

# Authenticate lần đầu
claude auth login
# Mở browser để login với Anthropic account

# Kiểm tra trạng thái
claude auth status
```

### 2.2 Authentication options

```bash
# OAuth — khuyến nghị cho individual developer
claude auth login

# API key — cho CI/CD và automation
export ANTHROPIC_API_KEY=sk-ant-...

# AWS Bedrock — enterprise
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=us-east-1

# Google Vertex AI — enterprise
export CLAUDE_CODE_USE_VERTEX=1
export CLOUD_ML_REGION=us-east5
export ANTHROPIC_VERTEX_PROJECT_ID=your-project
```

### 2.3 Khởi tạo project

```bash
cd your-project/
git init               # Luôn init git trước — đây là safety net
claude                 # Mở Claude Code
> /init                # Tạo CLAUDE.md tự động dựa trên codebase
```

Lệnh `/init` phân tích codebase để detect build system, test framework, code patterns — tạo nền tảng để bạn tinh chỉnh.

### 2.4 Cấu trúc thư mục chuẩn

```
your-project/
|-- CLAUDE.md                    # Persistent memory (đọc mỗi session)
|-- .claude/
|   |-- settings.json            # Hooks, env vars, permissions (commit)
|   |-- settings.local.json      # Cá nhân (không commit)
|   |-- skills/
|   |   |-- deploy/
|   |   |   |-- SKILL.md
|   |   |   |-- scripts/
|   |   |   `-- references/
|   |   `-- api-conventions/
|   |       `-- SKILL.md
|   |-- agents/
|   |   |-- code-reviewer.md
|   |   `-- security-auditor.md
|   `-- commands/
|-- .mcp.json                    # MCP servers (commit để team dùng chung)
|-- .claudeignore                # Files Claude bỏ qua
`-- src/
```

**Global settings** (áp dụng mọi project):

```
~/.claude/
|-- CLAUDE.md            # Global instructions
|-- settings.json        # Global settings
`-- settings.local.json  # Global personal settings
```

### 2.5 Settings.json

```json
{
  "model": "sonnet",
  "env": {
    "MAX_THINKING_TOKENS": "10000",
    "CLAUDE_CODE_SUBAGENT_MODEL": "haiku"
  },
  "permissions": {
    "allow": [
      "Bash(npm:*)",
      "Bash(git:*)",
      "Bash(make:*)",
      "Read",
      "Edit",
      "Write"
    ],
    "deny": [
      "Bash(rm:-rf*)",
      "Bash(curl:*)",
      "Edit(./migrations/**)",
      "Edit(.env)"
    ]
  }
}
```

**Settings.local.json** (cá nhân, không commit):

```json
{
  "model": "opus",
  "env": {
    "ANTHROPIC_API_KEY": "sk-ant-personal..."
  }
}
```

**Settings hierarchy** (sau ghi đè trước):

```
Global settings.json
  -> Global settings.local.json
    -> Project settings.json
      -> Project settings.local.json   (highest priority)
```

### 2.6 .claudeignore

```gitignore
# Dependencies
node_modules/
vendor/
.venv/

# Build outputs
dist/
build/
.next/
__pycache__/

# Sensitive files
.env
.env.*
*.key
*.pem

# Large data files
*.csv
*.sql
*.dump
data/

# Generated
coverage/
*.log
logs/
```

### 2.7 Quy tắc setup ban đầu

- Khởi tạo git trước khi làm bất cứ điều gì — là safety net cho mọi thay đổi
- Giữ permission mặc định — Claude sẽ hỏi trước khi đọc, ghi, hoặc thực thi
- Chạy `/init` để có CLAUDE.md nền tảng
- Tạo `.claudeignore` để loại trừ `node_modules/`, `dist/`, `.env`, log lớn
- Commit `.mcp.json` và `.claude/skills/` để team dùng chung
- Thêm `.claude/settings.local.json` vào `.gitignore`

---

## 3. CLAUDE.md — Bộ nhớ cố định của project

CLAUDE.md là file Claude **đọc tự động ở mỗi session**. Đây là đòn bẩy lớn nhất ảnh hưởng chất lượng output.

### 3.1 Hierarchy của CLAUDE.md

Claude đọc và merge theo thứ tự:

```
1. ~/.claude/CLAUDE.md              Global (áp dụng mọi project)
2. /project-root/CLAUDE.md          Project-level
3. /project-root/src/CLAUDE.md      Subdirectory (chỉ khi làm việc trong src/)
4. /project-root/src/api/CLAUDE.md  Nested (chỉ khi làm việc trong api/)
```

**Global CLAUDE.md** — đặt preferences cá nhân không thay đổi theo project:

```markdown
# ~/.claude/CLAUDE.md

## Style preferences
- Python và TypeScript developer
- Luôn dùng type hints
- Prefer functional style, tránh class khi không cần
- English cho code, Vietnamese cho comments giải thích

## Communication
- Trả lời ngắn gọn, không dài dòng
- Nếu không chắc, hỏi trước khi assume
- Không thêm feature không được yêu cầu
```

### 3.2 Nguyên tắc quan trọng

**CLAUDE.md tốn token mỗi turn.** Một file 5,000 token = 5,000 token tiêu trước khi bạn gõ chữ đầu tiên. Giữ ngắn, súc tích, dưới 60 dòng nếu được.

### 3.3 Import syntax — Modular CLAUDE.md

```markdown
# CLAUDE.md

## Project
SaaS B2B inventory management. Stack: FastAPI + PostgreSQL + React.

## Commands
- `make dev`  — start local stack
- `make test` — run tests

## References
@docs/code-style.md
@docs/api-conventions.md
@docs/architecture.md
```

Các file import chỉ load khi cần — lazy loaded. CLAUDE.md chính giữ ngắn trong khi vẫn accessible khi Claude cần đến.

> Xem thêm: [CLAUDE.md Patterns — Template Collection](deep-dives/claude-md-patterns.md)

### 3.4 Template CLAUDE.md hiệu quả

```markdown
# Project: [Tên project]

## What
SaaS B2B quản lý inventory. Stack: FastAPI + PostgreSQL + Redis + React.

## Why
Real-time inventory sync giữa nhiều cửa hàng. Optimistic locking trong user update flow.

## Code style
- Python: type hints bắt buộc, pydantic v2
- Tests: pytest + factory_boy, AAA pattern
- API: REST, không GraphQL
- Errors: raise CustomException, không return tuple

## Commands
- `make dev`      — start local stack
- `make test`     — chạy tests
- `make lint`     — ruff + mypy
- `make migrate`  — alembic upgrade head

## Model assignment
- Architecture/security review — Opus
- Implementation/refactor     — Sonnet (default)
- File search/exploration     — Haiku
- Plan complex feature        — opusplan

## Rules
- Không sửa file trong `migrations/` — luôn tạo migration mới
- Không commit secrets, dùng `.env.example`
- Luôn viết test trước khi merge
- Dùng skill `api-conventions` khi tạo endpoint mới
```

### 3.5 Không nên đặt trong CLAUDE.md

- Lịch sử debug hay fix recipes — thuộc về git history
- Code patterns dài — dùng skills thay thế
- Thông tin thay đổi thường xuyên (version numbers, dates)
- Nội dung đã có trong README
- Conversation context hay task hiện tại — thông tin ephemeral

---

## 4. Skills — Mở rộng năng lực Claude

Skills là thư mục chứa SKILL.md, mã hóa best practices cho một task cụ thể. Khác CLAUDE.md (luôn load), skills **lazy load** — Claude chỉ đọc khi cần.

### 4.1 Cơ chế trigger

Claude tự động load skill dựa trên mô tả trong frontmatter đối chiếu với user intent:

```
User: "Deploy lên production"
  -> Claude match với description "Use when user asks to deploy, release, push to prod"
  -> Load SKILL.md
  -> Follow skill instructions
```

Nếu user dùng `/skill-name`, Claude load ngay lập tức, bỏ qua matching.

### 4.2 Cấu trúc skill

```
.claude/skills/
`-- deploy/
    |-- SKILL.md          Frontmatter + nội dung chính (<50 dòng)
    |-- scripts/
    |   `-- deploy.sh
    `-- references/       Tài liệu chi tiết (lazy loaded)
        |-- runbook.md
        `-- rollback-guide.md
```

### 4.3 Mẫu SKILL.md

```markdown
---
name: deploy-production
description: Use this skill when the user asks to "deploy", "release", "push to prod",
  "ship it", or mentions production deployment. Handles full deployment workflow
  including pre-flight checks, deployment, and rollback.
allowed-tools: Bash(npm:*), Bash(git:*), Bash(gh:*), Read
model: opus
disable-model-invocation: false
---

# Deploy to Production

## Pre-flight (phải pass hết)
1. Run `make test` — must exit 0
2. Run `make lint` — must exit 0
3. Verify branch là `main`, không có uncommitted changes
4. Check `CHANGELOG.md` đã cập nhật
5. Confirm với user: "Ready to deploy version X.Y.Z?"

## Deployment
1. `git tag v$(cat VERSION)`
2. `git push origin main --tags`
3. `gh workflow run deploy.yml`
4. `gh run watch`
5. `curl https://api.example.com/health`

## Rollback
Xem `references/rollback-guide.md`.
Quick: `git revert HEAD && git push origin main`
```

### 4.4 Skills phổ biến nên có

| Skill | Trigger phrases | Mục đích |
|---|---|---|
| `api-conventions` | "create endpoint", "new route" | REST API standards |
| `db-migration` | "migration", "schema change" | Quy trình migration |
| `code-review` | "review", "check PR" | Checklist review |
| `bug-investigation` | "debug", "trace bug" | Workflow debug có hệ thống |
| `release` | "release", "tag", "deploy" | Tag, changelog, deploy |
| `testing-patterns` | "write test", "test for" | Mocking, fixtures, AAA |
| `security-review` | "security", "audit" | OWASP checklist |

### 4.5 Quy tắc viết skill tốt

- Description quyết định trigger — viết đúng cụm từ user thật sự dùng
- Imperative voice — "Read the file" không phải "You should read"
- Một skill = một expertise area
- SKILL.md dưới 50 dòng, chi tiết để trong `references/`
- Khai báo `allowed-tools` để giảm scope
- `disable-model-invocation: true` cho skills có side effects lớn

---

## 5. Subagents, Hooks, MCP, Plugins

### 5.1 Subagents

Subagent chạy trong context window riêng, report lại summary. Dùng để khám phá codebase mà không làm bẩn main context.

**Built-in subagents:**

| Agent | Model | Tools | Dùng cho |
|---|---|---|---|
| Explore | Haiku | Read, grep, find | Tìm files, grep symbols |
| Plan | Sonnet | Read, Bash (no write) | Lập plan trước khi code |
| General-purpose | Sonnet | Tất cả | Multi-step tasks |

**Custom subagent** (`.claude/agents/security-auditor.md`):

```markdown
---
name: security-auditor
description: Reviews code for security vulnerabilities. Use proactively after
  auth changes, payment flows, or when user asks for security review.
tools: Read, Grep, Bash(grep:*), Bash(find:*)
model: sonnet
---

You are a security-focused code auditor. Check for OWASP Top 10.

For each finding, report:
- Severity: Critical / High / Medium / Low
- Location: file:line
- Issue: what is wrong
- Impact: what an attacker can do
- Fix: concrete remediation
```

### 5.2 Hooks

Hooks chạy shell scripts tự động tại các events, không phụ thuộc vào Claude "nhớ" hay không.

> Xem thêm: [Hook Scripting — Thư viện examples đầy đủ](deep-dives/hook-scripting.md)

#### Tất cả hook events

| Event | Khi nào chạy | Use case |
|---|---|---|
| `PreToolUse` | Trước khi Claude dùng tool | Block nguy hiểm, validate |
| `PostToolUse` | Sau khi tool hoàn thành | Auto-format, auto-test |
| `UserPromptSubmit` | Khi user submit message | Inject context, validate input |
| `Stop` | Khi Claude kết thúc response | Notify, cleanup |
| `Notification` | Khi Claude cần user attention | Custom notifications |

#### Hook exit codes

| Exit code | Nghĩa | Hành vi |
|---|---|---|
| `0` | Success | Claude tiếp tục bình thường |
| `1` | Block | Claude bị chặn, stderr hiển thị cho user |
| `2` | Non-blocking error | Hiển thị warning, Claude tiếp tục |

#### Hook stdout và stderr

- **stdout** — được inject vào context của Claude (Claude đọc được)
- **stderr** — hiển thị cho user nhưng không vào Claude context

```bash
#!/bin/bash
# stdout -> Claude context
echo "Current branch: $(git rev-parse --abbrev-ref HEAD)"

# stderr -> hiển thị cho user
echo "Running lint..." >&2
npm run lint
```

#### Hook environment variables

```bash
$CLAUDE_TOOL_NAME    # "Edit", "Bash", "Write"...
$CLAUDE_TOOL_INPUT   # JSON input của tool
$CLAUDE_FILE_PATH    # Path của file đang edit
$CLAUDE_SESSION_ID   # ID session hiện tại
```

#### Ví dụ hooks cơ bản

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{"type": "command", "command": "npm run lint:fix 2>&1 || true"}]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "filter": "migrations/**",
        "hooks": [{"type": "command", "command": "echo 'Không sửa migrations.' && exit 1"}]
      }
    ],
    "Stop": [
      {
        "hooks": [{"type": "command", "command": "osascript -e 'display notification \"Claude finished\"'"}]
      }
    ]
  }
}
```

### 5.3 MCP Servers

> **Cảnh báo token**: MCP servers preload tool definitions (~100–300 tokens/tool). Cài 7+ servers có thể tiêu 67,000+ tokens trước khi bắt đầu làm việc.

> Xem thêm: [MCP Servers — Hướng dẫn đầy đủ](deep-dives/mcp-servers-guide.md)

```bash
# Add server cho user (cá nhân)
claude mcp add -s user playwright -- npx @playwright/mcp@latest

# Add server cho project (commit, team dùng chung)
claude mcp add -s project github -- npx @modelcontextprotocol/server-github

# Quản lý
claude mcp list
claude mcp remove playwright
claude mcp test github
```

**.mcp.json:**

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {"GITHUB_TOKEN": "${GITHUB_TOKEN}"}
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "${DATABASE_URL}"]
    }
  }
}
```

**Nguyên tắc**: Bắt đầu với 2–3 servers thật sự cần thiết. Không bật tất cả.

---

## 6. Git Worktrees và Parallel Development

### 6.1 Git Worktrees là gì?

Git worktrees cho phép checkout nhiều branches cùng lúc trong các thư mục riêng biệt. Mỗi worktree là isolated workspace, phù hợp để chạy nhiều Claude agents song song mà không conflict.

### 6.2 Setup cơ bản

```bash
# Tạo worktree cho feature branch
git worktree add ../project-feature-auth feature/auth
git worktree add ../project-hotfix-payments hotfix/payments

# List worktrees
git worktree list

# Xóa worktree sau khi merge
git worktree remove ../project-feature-auth
```

### 6.3 Parallel development với Claude

```bash
# Terminal 1 — main workspace
cd ~/project && claude
> Implement feature A trên branch feat/feature-a

# Terminal 2 — isolated worktree
cd ~/project-feature-b && claude
> Implement feature B trên branch feat/feature-b
```

Hai instances chạy độc lập, không conflict files.

### 6.4 Subagent isolation: worktree

```
Spawn subagent với isolation: worktree để:
1. Chạy full test suite
2. Tạo coverage report
3. Report lại kết quả

Main workspace không bị ảnh hưởng.
```

Worktree tự động được cleanup nếu subagent không tạo thay đổi.

### 6.5 Worktree cho code review

```bash
git fetch origin pull/123/head:pr-123
git worktree add ../project-pr-123 pr-123

cd ../project-pr-123 && claude
> Review PR này: logic, security, tests, naming.
> Spawn subagent security-auditor cho auth changes.
```

---

## 7. Chọn Model và Cấu hình Model

### 7.1 So sánh model (2026)

| Model | ID | Input | Output | Tốc độ | Dùng cho |
|---|---|---|---|---|---|
| Haiku 4.5 | claude-haiku-4-5 | $1/1M | $5/1M | Nhanh nhất | File search, simple edits, subagents |
| Sonnet 4.6 | claude-sonnet-4-6 | $3/1M | $15/1M | Trung bình | **Default — 80% tasks** |
| Opus 4.7 | claude-opus-4-7 | $15/1M | $75/1M | Chậm nhất | Architecture, security, complex reasoning |

### 7.2 Decision tree

```
Task là gì?
|
|-- Tìm file, grep, simple read              -> Haiku
|-- Viết code, refactor, bug fix             -> Sonnet
|-- Architecture, security, deep reasoning   -> Opus
`-- Plan phức tạp + implement               -> opusplan
```

### 7.3 Aliases và opusplan

| Alias | Hành vi |
|---|---|
| `sonnet` | claude-sonnet-4-6 |
| `opus` | claude-opus-4-7 |
| `haiku` | claude-haiku-4-5 |
| `opusplan` | Plan với Opus, execute với Sonnet — tiết kiệm ~60% |

### 7.4 Lệnh switch model

```
> /model sonnet
> /model opus
> /model opusplan
> /model haiku
> /model          (mở interactive picker)
> /fast           (toggle Fast mode — Opus 4.6, output nhanh hơn)
```

### 7.5 Model assignment trong CLAUDE.md

```markdown
## Model Assignment
- Architecture decisions   — Opus
- Implementation tasks     — Sonnet
- Simple edits, search     — Haiku
- Security review          — Opus (auto-escalate)
- Plan complex feature     — opusplan
```

### 7.6 Pin version cho enterprise

```bash
export ANTHROPIC_DEFAULT_OPUS_MODEL=claude-opus-4-7
export ANTHROPIC_DEFAULT_SONNET_MODEL=claude-sonnet-4-6
export ANTHROPIC_DEFAULT_HAIKU_MODEL=claude-haiku-4-5-20251001
```

---

## 8. Extended Thinking — Reasoning sâu

> Xem thêm: [Extended Thinking — Hướng dẫn đầy đủ](deep-dives/extended-thinking.md)

### 8.1 Extended Thinking là gì?

Extended Thinking cho phép Claude suy nghĩ nội bộ trước khi trả lời — dùng thinking tokens không hiển thị trong context. Kết quả: accuracy cao hơn cho bài toán phức tạp.

### 8.2 Khi nào dùng và khi nào không

**Nên dùng khi:**
- Multi-step logic phức tạp
- Architecture decisions với nhiều trade-offs
- Security analysis
- Debugging khó, root cause không rõ
- Algorithm và math problems

**Không cần khi:**
- Rename biến hoặc format code
- Simple file edits
- Lookup và search tasks
- Viết boilerplate code

### 8.3 Control Extended Thinking

**Trong Claude Code:**
```
> /model
# Picker hiện effort levels: low, medium, high, xhigh
```

| Level | Thinking tokens | Dùng khi |
|---|---|---|
| low | ~1,000 | Simple reasoning |
| medium | ~5,000 | Standard complex tasks |
| high | ~10,000 | Hard problems |
| xhigh | ~20,000+ | Research-grade reasoning |

**Qua environment variable:**
```bash
export MAX_THINKING_TOKENS=10000

# Disable adaptive thinking (Sonnet/Opus 4.6)
export CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING=1
```

**Opus 4.7** luôn dùng adaptive reasoning, không disable được.

### 8.4 Prompt để kích hoạt reasoning

```xml
<task>
Design authentication system cho multi-tenant SaaS.
Constraints: latency <100ms, GDPR compliant, support SSO.
</task>

Trước khi trả lời, phân tích:
1. Các approach có thể
2. Trade-offs của mỗi approach
3. Potential failure modes
```

---

## 9. Quản lý Context Window

> Xem thêm: [Context Window Mechanics — Kỹ thuật sâu](deep-dives/context-window-mechanics.md)

### 9.1 Context rot

Claude đọc lại toàn bộ conversation mỗi turn. Message thứ 50 tốn nhiều hơn message thứ 5 không phải vì task khó hơn, mà vì 49 messages trước phải re-read. Session dài vừa đắt vừa làm Claude kém chính xác dần.

Target: **dưới 70% capacity**.

### 9.2 Anatomy của context window

```
/context breakdown:

System prompt          ~2,000 tokens
CLAUDE.md              ~1,500 tokens
Tool definitions       ~3,000 tokens
MCP tools              ~5,000 tokens
Skills loaded          ~1,000 tokens
Conversation history  ~45,000 tokens   <- phình lớn nhất
File contents         ~12,000 tokens
---
Total                 ~69,500 / 200,000 (34.75%)
```

### 9.3 Dấu hiệu context cạn

1. Code mâu thuẫn với phần làm trước trong cùng session
2. Claude hỏi lại về structure đã giải thích
3. Bỏ qua quyết định kiến trúc đã thiết lập
4. Breaking changes ignore patterns đã thoả thuận

### 9.4 Lệnh quản lý context

| Lệnh | Tác dụng |
|---|---|
| `/context` | Xem breakdown chi tiết |
| `/compact` | Nén history thành summary, giữ files đã làm |
| `/clear` | Reset hoàn toàn |
| `/rewind` | Undo action cuối (restore checkpoint) |
| `/resume` | Tiếp tục session đã đóng |

### 9.5 Khi nào dùng gì?

| Tình huống | Lệnh |
|---|---|
| Hoàn thành sub-task, chuyển sub-task tiếp | `/compact` |
| Chuyển hẳn task khác (feature A sang bug B) | `/clear` |
| Claude vừa làm sai — muốn quay lại | `/rewind` |
| Context > 70% | `/compact` ngay |
| Session bị ngắt, muốn tiếp tục | `claude -c` hoặc `/resume` |

### 9.6 Compact thông minh

Trước khi compact, chỉ định rõ những gì cần giữ:

```
Trước khi compact, note lại:
- Đã quyết định dùng optimistic locking cho user update
- Schema không được sửa trong session này
- Test suite phải pass với pytest -xvs
- Đang ở bước 3/5 của migration plan
```

### 9.7 Subagents là lá chắn context tốt nhất

```
Spawn subagent Explore để khám phá module auth/, tìm OAuth utilities có sẵn.
Report lại file paths và function signatures.
```

Subagent đọc 30 files, report 200 token summary. Main context không bị ảnh hưởng.

### 9.8 Prompt cache — 5 phút TTL

Anthropic cache conversation context trong 5 phút. Respond trong 5 phút tiết kiệm 80–90% token của messages trước. Để idle quá 5 phút là cache miss, phải re-read toàn bộ.

> Xem thêm: [Prompt Caching — Cơ chế và tối ưu](deep-dives/prompt-caching.md)

---

## 10. Tối ưu Token và Chi phí

> Xem thêm: [Prompt Caching — Cơ chế và tối ưu](deep-dives/prompt-caching.md)

### 10.1 Ba nguồn tiêu token không thấy

1. **Tool output tích lũy** — file đọc một lần, ở context cả session
2. **MCP responses** — JSON đầy đủ stack vào context
3. **Re-read history** — message thứ N = đọc lại N-1 messages trước

### 10.2 Ước tính cost

```
1 token ≈ 0.75 từ tiếng Anh ≈ 4 ký tự

Session 30 messages điển hình:
- ~10,000 tokens input  = $0.03 (Sonnet)
- ~3,000 tokens output  = $0.045
- Tổng ~$0.075 / session
```

### 10.3 Tactics tiết kiệm 40–70%

```bash
# Chọn model đúng
/model sonnet                      # default
/model haiku                       # cho task đơn giản

# Cap thinking tokens
export MAX_THINKING_TOKENS=10000

# Subagent dùng Haiku
export CLAUDE_CODE_SUBAGENT_MODEL=haiku

# Theo dõi usage
ccusage
ccusage --breakdown
```

### 10.4 Anti-patterns tiêu token

| Thay vì... | Hãy... |
|---|---|
| "Reformat that as bullets" | Specify format từ đầu trong prompt |
| Đọc file 500 dòng | Grep tìm function, Read 50 dòng cụ thể |
| Hỏi tới hỏi lui để clarify | Viết task brief đầy đủ ngay message đầu |
| Opus cho mọi task | Sonnet default, Opus khi reason phức tạp |
| Session 100+ messages | `/compact` sau mỗi sub-task |
| Bật nhiều MCP servers | Chỉ cài 2–3 cái thật sự dùng |

### 10.5 Prompt caching

Bật mặc định — giữ nguyên. Cache giảm đáng kể chi phí input tokens cho repeated context.

```bash
# Chỉ disable khi debug
export DISABLE_PROMPT_CACHING=1
```

### 10.6 Plan mode

Trước task expensive, vào Plan mode (Shift+Tab x2) để Claude chỉ plan không execute. Review xong mới cho thực thi.

### 10.7 .claudeignore là token saver quan trọng

Files trong `.claudeignore` không bao giờ đọc vào context. `node_modules/` có thể chứa hàng trăm nghìn files — loại trừ ngay từ đầu.

### 10.8 Batch API cho automation

50% cheaper hơn, turnaround 24 giờ, phù hợp khi không cần real-time:

```python
import anthropic

client = anthropic.Anthropic()
batch = client.messages.batches.create(
    requests=[
        {
            "custom_id": f"review-{i}",
            "params": {
                "model": "claude-sonnet-4-6",
                "max_tokens": 1024,
                "messages": [{"role": "user", "content": f"Review {file}"}]
            }
        }
        for i, file in enumerate(files_to_review)
    ]
)
```

---

## 11. Prompt Engineering cho Claude

### 11.1 Chọn format theo task

| Format | Dùng khi | Lý do |
|---|---|---|
| Plain text | Câu hỏi đơn giản, chat casual | Không cần overhead |
| XML tags | Prompt phức tạp với 3+ thành phần | Claude được fine-tune cho XML |
| JSON | Output schema, function calling | Dễ parse trong code |
| Markdown | Document-style, lightweight | Rõ ràng vừa đủ |

### 11.2 XML là native format của Claude

XML tạo semantic anchoring — boundary rõ ràng giúp Claude duy trì context tốt hơn. JSON gây "nesting anxiety" khi nested sâu. Với Claude, XML tốt hơn JSON cho prompt structure.

Benchmark: XML-scaffolded prompts đạt accuracy cao hơn ~23% trên reasoning tasks so với JSON thuần.

### 11.3 Template chuẩn

```xml
<role>
Senior Python backend engineer, FastAPI + PostgreSQL.
</role>

<context>
Codebase dùng pydantic v2, async SQLAlchemy.
Team ưu tiên code đơn giản, test coverage > 80%.
</context>

<task>
Refactor get_user_orders để fix N+1 query.
</task>

<code>
async def get_user_orders(user_id: int):
    user = await db.get(User, user_id)
    orders = []
    for order_id in user.order_ids:
        order = await db.get(Order, order_id)
        orders.append(order)
    return orders
</code>

<constraints>
- Không đổi function signature
- Không bịa method chưa tồn tại trong SQLAlchemy
- Giữ async/await
</constraints>

<output_format>
<analysis>Phân tích vấn đề N+1</analysis>
<solution>```python\n...\n```</solution>
<test>```python\n...\n```</test>
</output_format>
```

### 11.4 Few-shot examples

3–5 examples là optimal — quá nhiều có diminishing returns:

```xml
<examples>
  <example>
    <input>Sản phẩm tệ quá</input>
    <output>negative</output>
  </example>
  <example>
    <input>Dịch vụ ổn</input>
    <output>neutral</output>
  </example>
  <example>
    <input>Tuyệt vời, sẽ giới thiệu bạn bè!</input>
    <output>positive</output>
  </example>
</examples>
```

### 11.5 Negative constraints

Nói rõ điều không muốn — quan trọng không kém điều muốn:

```xml
<constraints>
- Không thêm error handling không được yêu cầu
- Không refactor code ngoài scope
- Không add dependency mới
- Không thay đổi public API signature
- Không thêm comment giải thích code hiển nhiên
</constraints>
```

### 11.6 Role prompting hiệu quả

```xml
<!-- Tốt: cụ thể và relevant -->
<role>
Staff engineer chuyên Kubernetes và distributed systems,
đã scale hệ thống lên 10M requests/day.
</role>

<!-- Không tốt: vague -->
<role>Expert developer</role>
```

Role tốt nhất là người có exact expertise cần cho task cụ thể đó.

### 11.7 Chain-of-thought

```
Giải thích step by step trước khi đưa ra kết luận.
```

Đặc biệt hiệu quả với Opus và Extended Thinking.

### 11.8 Prefilling (API level)

```python
# Force JSON — không preamble
messages = [
    {"role": "user", "content": "Analyze this..."},
    {"role": "assistant", "content": "{"}
]

# Force code
messages = [
    {"role": "user", "content": "Write a function..."},
    {"role": "assistant", "content": "```python\n"}
]
```

### 11.9 Hybrid XML + JSON

```xml
<thinking>
Reason trong XML
</thinking>

<output_format>
Trả về JSON:
{"answer": "...", "confidence": 0.0-1.0}
</output_format>
```

### 11.10 Error recovery

Khi Claude đi sai hướng, đừng tiếp tục từ đó — sẽ tốn nhiều token hơn để undo về sau:

```
Dừng lại. Approach này sai vì [lý do].

Constraints tôi chưa nêu rõ:
- [Constraint 1]
- [Constraint 2]

Restart từ đầu với constraints mới.
```

---

## 12. Workflow gen code trong dự án

### 12.1 Feature mới

```
1.  cd project && claude
2.  /clear                             context sạch
3.  /model opusplan                    plan Opus, execute Sonnet
4.  Đọc CLAUDE.md và module liên quan
5.  Shift+Tab x2                       vào Plan mode
6.  Describe feature cần implement
7.  Review plan, confirm nếu đồng ý
8.  Shift+Tab x2                       exit plan mode, Claude implement
9.  /compact sau mỗi sub-task lớn
10. Chạy tests, fix nếu fail
11. Generate commit message (conventional commits)
12. git diff -> review -> commit
```

### 12.2 Debug bug

```
1. /clear
2. Spawn subagent Explore để tìm files liên quan đến [bug description]
3. [Subagent report: N files]
4. Đọc files đó, đề xuất 3 root cause hypotheses theo likelihood
5. Implement fix cho hypothesis khả thi nhất
6. Viết test reproduce bug trước (red), sau đó fix để green
7. /compact — note root cause và approach
8. Run full test suite
```

### 12.3 Code review

```
1. /clear
2. Spawn subagent code-reviewer trên diff của branch hiện tại
3. [Subagent report findings: P1/P2/P3]
4. Fix tất cả P1
5. /compact
6. Generate PR description: summary, test plan, breaking changes
```

### 12.4 Onboard codebase mới

```
1. /init                              tạo CLAUDE.md baseline
2. Đọc package.json, README, entry points chính
3. Tạo sơ đồ kiến trúc ASCII, lưu vào ARCHITECTURE.md
4. Liệt kê coding patterns: test pattern, error handling, naming
5. Append patterns vào CLAUDE.md
6. Đâu là 3 module phức tạp nhất cần hiểu trước?
```

### 12.5 Architecture decision

```
1. /model opus                        cần deep reasoning
2. Shift+Tab x2                       Plan mode
3. So sánh N approaches cho [vấn đề]:
   Xét theo: latency, scalability, ops complexity, cost
   Context: [stack hiện tại, scale hiện tại, growth target]
4. [Claude analyze trade-offs]
5. Quyết định: implement approach X vì [lý do]
6. Update CLAUDE.md với quyết định
```

### 12.6 Non-interactive mode

```bash
# Review diff trong CI
git diff HEAD~1 | claude --print "Review diff, liệt kê vấn đề"

# Generate changelog
git log --oneline HEAD~10.. | claude -p "Generate changelog từ commits này"

# JSON output cho pipeline
claude --output-format json --print "Analyze..." < code.py | jq '.issues'
```

### 12.7 Session management

```bash
# List sessions gần đây
claude --list-sessions

# Resume session cụ thể
claude --resume abc123def

# Continue session gần nhất
claude --continue "Tiếp tục nhưng thêm unit tests"
```

---

## 13. IDE Integration và Keyboard Shortcuts

### 13.1 Platforms được hỗ trợ

- **Terminal** — native CLI, full features
- **VS Code** — official extension
- **JetBrains** (IntelliJ, PyCharm, WebStorm...) — official plugin
- **Web** — claude.ai/code

### 13.2 Keyboard shortcuts

| Action | Mac | Windows/Linux |
|---|---|---|
| Submit message | `Enter` | `Enter` |
| Newline trong input | `Shift+Enter` | `Shift+Enter` |
| Enter/Exit Plan mode | `Shift+Tab` x2 | `Shift+Tab` x2 |
| Interrupt Claude | `Ctrl+C` | `Ctrl+C` |
| Clear input | `Ctrl+U` | `Ctrl+U` |
| Previous message | `Up` (input rỗng) | `Up` (input rỗng) |
| History search | `Ctrl+R` | `Ctrl+R` |
| Exit Claude | `/exit` hoặc `Ctrl+D` | `/exit` hoặc `Ctrl+D` |

### 13.3 Slash commands tham khảo nhanh

| Command | Tác dụng |
|---|---|
| `/help` | Xem tất cả commands |
| `/model [name]` | Switch model |
| `/fast` | Toggle Fast mode |
| `/context` | Xem context breakdown |
| `/compact` | Compress history |
| `/clear` | Reset context |
| `/rewind` | Undo last action |
| `/resume` | Resume previous session |
| `/init` | Generate CLAUDE.md |
| `/plugin` | Browse plugins |
| `/mcp` | MCP server status |
| `/exit` | Thoát |

### 13.4 Vi mode

```bash
export CLAUDE_CODE_VI_MODE=1
# hoặc trong settings.json:
# "inputMode": "vi"
```

### 13.5 StatusLine — Real-time monitoring

StatusLine plugin hiển thị trong terminal prompt:

```
[Claude: Sonnet | Context: 34% | $0.023]
```

Hữu ích để biết khi nào cần `/compact`.

---

## 14. Team Collaboration

### 14.1 Cái gì commit, cái gì không

| File | Commit | Lý do |
|---|---|---|
| `CLAUDE.md` | Có | Shared project memory |
| `.claude/settings.json` | Có | Team-wide hooks, permissions |
| `.claude/skills/` | Có | Shared best practices |
| `.claude/agents/` | Có | Shared subagents |
| `.mcp.json` | Có | Team MCP servers |
| `.claudeignore` | Có | Shared ignore rules |
| `.claude/settings.local.json` | Không | Personal preferences |
| `~/.claude/CLAUDE.md` | Không | Individual preferences |

**.gitignore entries:**

```gitignore
.claude/settings.local.json
*.claude-session
```

### 14.2 Team skills — Chuẩn hóa workflow

Skills là cách codify team standards hiệu quả nhất:

```
.claude/skills/
|-- api-conventions/    REST API standards
|-- code-review/        Review checklist
|-- testing/            Test patterns
|-- db-migration/       Migration workflow
|-- release/            Release process
`-- incident-response/  On-call runbook
```

Mỗi khi team agree về convention mới — thêm vào skill tương ứng.

### 14.3 CLAUDE.md làm onboarding doc

```markdown
## For new team members
1. Đọc ARCHITECTURE.md để hiểu hệ thống
2. Chạy `make dev` để start local
3. Đọc `docs/coding-standards.md`
4. Dùng skill `api-conventions` khi tạo endpoint
5. Mọi PR phải pass `make test && make lint`
```

### 14.4 Settings permissions cho team

```json
{
  "permissions": {
    "allow": [
      "Bash(make:*)",
      "Bash(git:*)",
      "Bash(npm:*)",
      "Read", "Edit", "Write"
    ],
    "deny": [
      "Edit(./migrations/**)",
      "Edit(.env)",
      "Bash(rm:-rf*)"
    ]
  }
}
```

Mỗi thành viên thêm personal allows trong `settings.local.json` (không commit).

---

## 15. CI/CD và Automation nâng cao

### 15.1 GitHub Actions — AI code review

```yaml
name: AI Code Review
on: [pull_request]

jobs:
  ai-review:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Run review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          git diff origin/${{ github.base_ref }}...HEAD > diff.txt
          claude --model sonnet --output-format json \
            --print "Review diff. Return JSON: {breaking_changes, security_issues, missing_tests}" \
            < diff.txt > review.json

      - name: Comment PR
        uses: actions/github-script@v7
        with:
          script: |
            const r = JSON.parse(require('fs').readFileSync('review.json'));
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## AI Review\n\n**Issues:** ${JSON.stringify(r)}`
            });
```

### 15.2 Pre-commit hook

```bash
#!/bin/bash
# .git/hooks/pre-commit

STAGED=$(git diff --cached --name-only --diff-filter=ACM | grep '\.py$')
[ -z "$STAGED" ] && exit 0

git diff --cached -- $STAGED | \
  claude --model haiku --print \
    "Scan nhanh: syntax error, obvious bug, hardcoded secret?
     Chỉ flag critical. Format: PASS hoặc FAIL: [reason]" \
  > /tmp/claude-review.txt

if grep -q "^FAIL" /tmp/claude-review.txt; then
  echo "Claude pre-commit check failed:"
  cat /tmp/claude-review.txt
  exit 1
fi
```

### 15.3 Automated changelog

```bash
#!/bin/bash
LAST_TAG=$(git describe --tags --abbrev=0)
COMMITS=$(git log $LAST_TAG..HEAD --oneline)

claude --model sonnet --print "Generate CHANGELOG entry từ commits:
$COMMITS
Group by: Features, Bug Fixes, Breaking Changes"
```

### 15.4 Docker integration

```dockerfile
FROM node:20-slim
RUN npm install -g @anthropic-ai/claude-code
ENV CI=true
WORKDIR /workspace
ENTRYPOINT ["claude", "--dangerously-skip-permissions", "--print"]
```

```bash
docker run --rm \
  -e ANTHROPIC_API_KEY=sk-ant-... \
  -v $(pwd):/workspace \
  claude-code \
  "Review tất cả Python files trong /workspace/src"
```

---

## 16. Security Best Practices

### 16.1 Trust model

Claude Code không có internet access trừ khi:
- Bạn cài MCP server cho service cụ thể
- Bạn cho phép `Bash(curl:*)` hoặc tương tự

Mặc định, Claude Code chỉ đọc/ghi files trong working directory và chạy Bash commands bạn cho phép. Code không gửi đến đâu ngoài Anthropic API.

### 16.2 Bảo vệ secrets

```gitignore
# .claudeignore — loại trừ khỏi Claude context
.env
.env.*
secrets/
*.key
*.pem
credentials.json
service-account.json
```

```bash
# Dùng environment variables thay vì paste trực tiếp
export DATABASE_URL="postgresql://..."
# Claude thấy tên biến, không phải giá trị
```

### 16.3 Least privilege permissions

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Edit(src/**)",
      "Bash(npm test)"
    ],
    "deny": [
      "Edit(.env)",
      "Bash(curl:*)",
      "Bash(wget:*)",
      "Bash(rm:-rf*)"
    ]
  }
}
```

Mở rộng permissions khi thật sự cần — không bật tất cả từ đầu.

### 16.4 Prompt injection awareness

MCP servers và external data có thể chứa prompt injection. Bảo vệ:

- Review MCP server source trước khi tin dùng
- Không dùng MCP servers không rõ nguồn gốc
- Deny `Bash(curl:*)` nếu không cần internet
- Review tool calls bất thường trước khi approve

### 16.5 Audit trail với git

```bash
# Tạo checkpoint trước khi Claude thay đổi nhiều
git add -A && git commit -m "checkpoint before Claude session"

# Sau session, review tất cả thay đổi
git diff HEAD
git log --oneline -10
```

### 16.6 Review Bash commands trước khi approve

Không bao giờ approve blindly. Đọc commands đủ kỹ như đọc code vào production. Với destructive commands, kiểm tra kỹ hơn.

---

## 17. Debugging và Troubleshooting

### 17.1 Context quality issues

**Triệu chứng**: Claude đưa ra code mâu thuẫn hoặc ignore constraints.

```
/context       check % used

Nếu > 70%:
/compact       nén với note về decisions quan trọng

Nếu vẫn kém:
/clear         reset, start fresh với context sạch
```

Root cause thường gặp: session quá dài, file lớn đã đọc vào context, CLAUDE.md quá dài.

### 17.2 Model behavior issues

**Claude không follow instructions:**
1. Check CLAUDE.md có conflict không
2. Check skills có auto-load và override không
3. Thử `/clear` và repeat với context sạch hơn
4. Dùng `<constraints>` XML tags thay vì plain text

**Claude "quên" decisions:**
1. Restate decisions rõ ràng sau `/compact`
2. Lưu decisions vào CLAUDE.md ngay lập tức

**Claude hallucinate methods/APIs:**
```xml
<constraints>
Không bịa API methods. Nếu không chắc, đọc source code
hoặc hỏi trước khi viết.
</constraints>
```

### 17.3 Hook failures

```bash
# Debug hook bằng cách chạy trực tiếp
chmod +x .claude/hooks/post-edit.sh
./.claude/hooks/post-edit.sh

# Kiểm tra syntax JSON
cat .claude/settings.json | python3 -m json.tool
```

Common bugs:
- Thiếu `#!/bin/bash` shebang
- Script không có execute permission (`chmod +x`)
- Path relative thay vì absolute
- Exit code sai (1 = block, 2 = warning, không phải ngược lại)

### 17.4 MCP connection issues

```bash
claude mcp list
claude mcp test github
claude mcp restart
claude mcp logs github

# Reset nếu bị stuck
claude mcp remove github
claude mcp add -s project github -- npx @modelcontextprotocol/server-github
```

Common issues: token expired, npx package không tìm thấy, SSE timeout (dùng stdio thay thế).

### 17.5 Authentication issues

```bash
claude auth status
claude auth logout && claude auth login

# Verify API key trực tiếp
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-haiku-4-5","max_tokens":10,"messages":[{"role":"user","content":"hi"}]}'
```

### 17.6 Restore từ git

```bash
git diff                        xem tất cả thay đổi
git checkout -- file.py         restore một file
git checkout -- .               restore tất cả (cẩn thận)

# Hoặc trong Claude:
/rewind                         undo last action
```

### 17.7 Performance issues

**Claude chậm:**
- Dùng `/fast` (Fast mode)
- Switch sang Haiku cho task đơn giản
- Disable extended thinking: `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING=1`

**Cost cao hơn expected:**
```bash
ccusage --breakdown --session SESSION_ID
# Identify: MCP servers không dùng, CLAUDE.md dài, compact không đủ
```

---

## 18. Checklist và Anti-patterns

### 18.1 Daily checklist

**Project setup:**
- [ ] `git init` trước session đầu
- [ ] CLAUDE.md dưới 60 dòng, có model assignment rules
- [ ] `.claudeignore` loại trừ node_modules, dist, .env
- [ ] Commit `.mcp.json` và `.claude/skills/` cho team

**Mỗi session:**
- [ ] `/clear` nếu chuyển sang task mới
- [ ] Default model = Sonnet, escalate Opus khi cần
- [ ] Subagent dùng Haiku cho exploration
- [ ] `/compact` sau mỗi sub-task lớn
- [ ] Monitor `/context` dưới 70%
- [ ] Plan mode trước task tốn token

**Khi code:**
- [ ] Git checkpoint trước khi Claude thay đổi lớn
- [ ] Review `git diff` sau mỗi session
- [ ] Đọc kỹ Bash commands trước khi approve

### 18.2 Anti-patterns cần tránh

**Model:**
- Dùng Opus cho rename biến hoặc format code
- Bật nhiều MCP servers không cần thiết

**Context:**
- Session dài 100+ messages mà không compact
- CLAUDE.md trên 200 dòng
- Đọc file lớn khi chỉ cần một hàm

**Prompting:**
- Prompt vague dẫn đến 5 turns clarify
- Không dùng `<constraints>` — Claude tự ý thêm features
- Plain text cho prompt phức tạp với nhiều thành phần

**Security:**
- Commit `.claude/settings.local.json`
- Để secrets trong files Claude có thể đọc
- Approve Bash commands mà không đọc kỹ

### 18.3 Checklist trước PR

```
Spawn subagent code-reviewer cho diff này:
1. Logic correctness
2. Security vulnerabilities (OWASP top 10)
3. Missing tests
4. Breaking changes
5. Performance concerns

Report: P1 (must fix), P2 (should fix), P3 (nice to have)
```

---

## 19. Tài nguyên tham khảo

### 19.1 Tài liệu chính thức Anthropic

- [Claude Code Overview](https://docs.anthropic.com/en/docs/claude-code/overview)
- [Claude Code CLI Reference](https://docs.anthropic.com/en/docs/claude-code/cli-reference)
- [Claude Code Best Practices](https://docs.anthropic.com/en/docs/claude-code/best-practices)
- [Models Overview](https://docs.anthropic.com/en/docs/about-claude/models/overview)
- [Prompt Engineering Guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)
- [Use XML Tags](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/use-xml-tags)
- [Extended Thinking](https://docs.anthropic.com/en/docs/build-with-claude/extended-thinking)
- [MCP Documentation](https://modelcontextprotocol.io/docs)
- [Anthropic Courses (GitHub)](https://github.com/anthropics/courses)

### 19.2 Tài liệu cộng đồng

- [Claude Code Token Optimization](https://buildtolaunch.substack.com/p/claude-code-token-optimization)
- [18 Token Management Hacks](https://www.mindstudio.ai/blog/claude-code-token-management-hacks)
- [Advisor Strategy: Opus + Sonnet/Haiku](https://www.mindstudio.ai/blog/claude-code-advisor-strategy-opus-sonnet-haiku)
- [Pick the Right Model for Every Task](https://dev.to/klement_gunndu/pick-the-right-claude-code-model-for-every-task-1p6a)
- [Context Window Optimization](https://claudefa.st/blog/guide/mechanics/context-management)
- [54% Context Reduction Case Study](https://gist.github.com/johnlindquist/849b813e76039a908d962b2f0923dc9a)
- [Understanding Full Stack: MCP, Skills, Subagents, Hooks](https://alexop.dev/posts/understanding-claude-code-full-stack/)
- [XML vs JSON Prompts Benchmark](https://www.nexailabs.com/blog/cracking-the-code-json-or-xml-for-better-prompts)

### 19.3 Templates và Examples

- [Anthropic Claude Code (Official)](https://github.com/anthropics/claude-code)
- [Claude Code Showcase](https://github.com/ChrisWiles/claude-code-showcase)
- [Everything Claude Code](https://github.com/affaan-m/everything-claude-code)
- [Claude Code Skills Suite](https://github.com/levnikolaevich/claude-code-skills)

### 19.4 Tools hỗ trợ

| Tool | Mục đích |
|---|---|
| ccusage | CLI xem token spend theo session |
| Claude Code Setup Plugin | Auto-recommend automations |
| StatusLine | Real-time context monitoring |

### 19.5 Pricing (04/2026)

| Plan | Giá | Hạn mức |
|---|---|---|
| Free | $0 | Haiku, Sonnet (giới hạn) |
| Pro | $20/tháng | Opus, ~45 messages/5h |
| Max 5x | $100/tháng | 5x Pro limits |
| Max 20x | $200/tháng | 20x Pro, full Opus |
| API | Pay-per-use | Sonnet $3/$15, Opus $15/$75 / 1M tokens |

Batch API: 50% cheaper cho non-realtime workloads.  
Prompt caching: 80–90% cheaper cho repeated context.

---

## Lời kết

**Ba nguyên tắc cốt lõi:**

1. **CLAUDE.md ngắn gọn + skills lazy load** — bộ nhớ thông minh tốt hơn bộ nhớ lớn
2. **Sonnet default, Opus khi cần, Haiku cho subagent** — match model với task
3. **Subagents + /compact + /clear** — context sạch là context tốt

**Quy tắc 80/20**: 80% chất lượng đến từ CLAUDE.md tốt, chọn model đúng, và prompt rõ ràng. 20% còn lại đến từ skills, hooks, MCP — chỉ thêm khi thật sự cần.

**Stack tối ưu cho một project:**

```
git init
  -> /init — CLAUDE.md baseline
  -> .claudeignore — bảo vệ context
  -> settings.json — permissions + hooks
  -> 2-3 skills — codify team standards
  -> 1-2 MCP servers — context thật sự cần
  -> opusplan — plan Opus, execute Sonnet
```

---

### Deep Dives — Tài liệu chuyên sâu

| File | Nội dung |
|---|---|
| [Prompt Caching](deep-dives/prompt-caching.md) | Cơ chế hoạt động, tối ưu cache hit rate |
| [Context Window Mechanics](deep-dives/context-window-mechanics.md) | Token economics, compaction, monitoring |
| [Hook Scripting](deep-dives/hook-scripting.md) | Thư viện 20+ hook examples thực tế |
| [MCP Servers Guide](deep-dives/mcp-servers-guide.md) | Tất cả servers, custom MCP, debugging |
| [CLAUDE.md Patterns](deep-dives/claude-md-patterns.md) | Templates cho 8 loại project khác nhau |

---

*Cập nhật 04/2026 — Claude Code Handbook*

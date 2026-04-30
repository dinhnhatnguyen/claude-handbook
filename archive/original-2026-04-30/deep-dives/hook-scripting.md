# Hook Scripting — Thư viện examples đầy đủ

> Tập hợp hooks thực tế cho các use cases phổ biến. Mỗi hook kèm theo giải thích, cách cài, và cách test.

---

## 1. Nền tảng cần biết

### 1.1 Cách hooks hoạt động

```
Claude muốn dùng tool Edit
  -> PreToolUse hook chạy
  -> Nếu exit 0: tool chạy
  -> Nếu exit 1: tool bị block, Claude nhận thông báo từ stderr
  -> Nếu exit 2: warning, tool vẫn chạy

Tool Edit hoàn thành
  -> PostToolUse hook chạy
  -> stdout của hook được inject vào context Claude
  -> stderr của hook hiển thị cho user
```

### 1.2 Cấu trúc settings.json

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Tên tool hoặc regex",
        "filter": "File path pattern (tùy chọn)",
        "hooks": [
          {
            "type": "command",
            "command": "shell command here"
          }
        ]
      }
    ],
    "PostToolUse": [...],
    "UserPromptSubmit": [...],
    "Stop": [...],
    "Notification": [...]
  }
}
```

### 1.3 Matchers

```json
"matcher": "Edit"           // Exact tool name
"matcher": "Edit|Write"     // OR — nhiều tools
"matcher": "Bash"           // Tất cả Bash calls
"matcher": ".*"             // Mọi tool (cẩn thận)
```

### 1.4 Filters (chỉ áp dụng cho file-based tools)

```json
"filter": "migrations/**"      // Glob pattern
"filter": "**/*.test.ts"       // Wildcard
"filter": "src/core/**"        // Directory
```

### 1.5 Environment variables trong hook

```bash
$CLAUDE_TOOL_NAME      # "Edit", "Write", "Bash"...
$CLAUDE_FILE_PATH      # "/project/src/auth.py" (khi edit file)
$CLAUDE_TOOL_INPUT     # JSON string của tool input
$CLAUDE_SESSION_ID     # Session ID hiện tại
```

### 1.6 Test hook trực tiếp

```bash
# Trước khi cài, test script trực tiếp
chmod +x .claude/hooks/my-hook.sh
CLAUDE_FILE_PATH="src/auth.py" CLAUDE_TOOL_NAME="Edit" .claude/hooks/my-hook.sh
echo "Exit code: $?"
```

---

## 2. PostToolUse Hooks — Sau khi tool chạy xong

### 2.1 Auto-format sau khi edit

**Mục đích**: Tự động format code sau mỗi lần Claude edit file.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "if [[ \"$CLAUDE_FILE_PATH\" == *.py ]]; then ruff format \"$CLAUDE_FILE_PATH\" 2>&1; fi"
          }
        ]
      }
    ]
  }
}
```

**Phiên bản phân loại theo file type:**

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "#!/bin/bash\ncase \"$CLAUDE_FILE_PATH\" in\n  *.py) ruff format \"$CLAUDE_FILE_PATH\" 2>&1 ;;\n  *.ts|*.tsx|*.js|*.jsx) npx prettier --write \"$CLAUDE_FILE_PATH\" 2>&1 ;;\n  *.go) gofmt -w \"$CLAUDE_FILE_PATH\" 2>&1 ;;\nesac"
          }
        ]
      }
    ]
  }
}
```

### 2.2 Auto-lint và type check

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "filter": "**/*.py",
        "hooks": [
          {
            "type": "command",
            "command": "ruff check \"$CLAUDE_FILE_PATH\" --fix 2>&1 || true; mypy \"$CLAUDE_FILE_PATH\" --ignore-missing-imports 2>&1 | tail -5"
          }
        ]
      }
    ]
  }
}
```

Lưu ý: `|| true` để không block nếu lint có warnings nhỏ. Bỏ `|| true` nếu muốn block khi có error.

### 2.3 Run tests liên quan sau khi edit

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "filter": "src/**/*.py",
        "hooks": [
          {
            "type": "command",
            "command": "pytest tests/ -x --tb=short -q 2>&1 | tail -20"
          }
        ]
      }
    ]
  }
}
```

**Phiên bản thông minh hơn — chỉ chạy test file tương ứng:**

```bash
#!/bin/bash
# .claude/hooks/run-related-tests.sh

SOURCE_FILE="$CLAUDE_FILE_PATH"

# Xác định test file tương ứng
if [[ "$SOURCE_FILE" == src/auth.py ]]; then
  TEST_FILE="tests/test_auth.py"
elif [[ "$SOURCE_FILE" == src/users.py ]]; then
  TEST_FILE="tests/test_users.py"
else
  # Guess: src/foo.py -> tests/test_foo.py
  BASENAME=$(basename "$SOURCE_FILE" .py)
  TEST_FILE="tests/test_${BASENAME}.py"
fi

if [ -f "$TEST_FILE" ]; then
  echo "Running $TEST_FILE..." >&2
  pytest "$TEST_FILE" -x --tb=short -q 2>&1 | tail -15
fi
```

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "filter": "src/**/*.py",
        "hooks": [{"type": "command", "command": "bash .claude/hooks/run-related-tests.sh"}]
      }
    ]
  }
}
```

### 2.4 Inject git status sau mỗi edit

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "echo \"Files changed: $(git diff --name-only | wc -l | tr -d ' ')\"; git diff --stat 2>/dev/null | tail -5"
          }
        ]
      }
    ]
  }
}
```

stdout này được inject vào context — Claude biết được trạng thái git hiện tại.

### 2.5 Check secrets sau khi write

```bash
#!/bin/bash
# .claude/hooks/check-secrets.sh
# Kiểm tra file mới viết có chứa secrets không

FILE="$CLAUDE_FILE_PATH"
[ -z "$FILE" ] && exit 0
[ ! -f "$FILE" ] && exit 0

# Patterns nguy hiểm
PATTERNS=(
  "sk-ant-"
  "AKIA[0-9A-Z]{16}"
  "ghp_[a-zA-Z0-9]{36}"
  "xoxb-[0-9]+"
  "-----BEGIN RSA PRIVATE KEY-----"
  "password\s*=\s*['\"][^'\"]{8,}"
)

for pattern in "${PATTERNS[@]}"; do
  if grep -qE "$pattern" "$FILE" 2>/dev/null; then
    echo "SECURITY: Potential secret found in $FILE (pattern: $pattern)" >&2
    echo "WARNING: Possible secret detected in $FILE" # -> Claude context
    exit 2  # Non-blocking: warn nhưng không block
  fi
done
```

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [{"type": "command", "command": "bash .claude/hooks/check-secrets.sh"}]
      }
    ]
  }
}
```

---

## 3. PreToolUse Hooks — Trước khi tool chạy

### 3.1 Block edit migrations

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "filter": "migrations/**",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'ERROR: Không được sửa migration files. Tạo migration mới bằng alembic revision.' >&2 && exit 1"
          }
        ]
      }
    ]
  }
}
```

### 3.2 Block rm -rf

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "if echo \"$CLAUDE_TOOL_INPUT\" | python3 -c \"import sys,json; cmd=json.load(sys.stdin).get('command',''); exit(0 if 'rm -rf' not in cmd else 1)\"; then exit 0; else echo 'ERROR: rm -rf bị block. Xóa file cụ thể thay thế.' >&2; exit 1; fi"
          }
        ]
      }
    ]
  }
}
```

**Phiên bản đơn giản hơn:**

```bash
#!/bin/bash
# .claude/hooks/block-dangerous-commands.sh

COMMAND=$(echo "$CLAUDE_TOOL_INPUT" | python3 -c "import sys,json; print(json.load(sys.stdin).get('command',''))" 2>/dev/null)

BLOCKED_PATTERNS=(
  "rm -rf"
  "DROP TABLE"
  "DROP DATABASE"
  "truncate"
  "> /dev/null 2>&1 && rm"
)

for pattern in "${BLOCKED_PATTERNS[@]}"; do
  if echo "$COMMAND" | grep -qi "$pattern"; then
    echo "ERROR: Command contains blocked pattern '$pattern'" >&2
    echo "Blocked command: $COMMAND" >&2
    exit 1
  fi
done
```

### 3.3 Require confirmation cho production commands

```bash
#!/bin/bash
# .claude/hooks/confirm-production.sh
# Chỉ áp dụng khi có dấu hiệu production

COMMAND=$(echo "$CLAUDE_TOOL_INPUT" | python3 -c "import sys,json; print(json.load(sys.stdin).get('command',''))" 2>/dev/null)

PROD_PATTERNS=("kubectl.*prod" "helm.*production" "gcloud.*--project=prod" "aws.*production")

for pattern in "${PROD_PATTERNS[@]}"; do
  if echo "$COMMAND" | grep -qE "$pattern"; then
    echo "WARNING: Production command detected: $COMMAND" >&2
    echo "Đang chạy production command — hãy verify trước khi approve." >&2
    # Không block (exit 0) nhưng warn user và inject cảnh báo vào context
    echo "NOTE: Production command was executed: $COMMAND"
    exit 0
  fi
done
```

### 3.4 Inject git branch context trước mỗi edit

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "echo \"Branch: $(git rev-parse --abbrev-ref HEAD) | Uncommitted: $(git diff --name-only | wc -l | tr -d ' ') files\""
          }
        ]
      }
    ]
  }
}
```

stdout này được inject vào context — Claude biết mình đang ở branch nào.

---

## 4. UserPromptSubmit Hooks — Khi user submit message

### 4.1 Inject project context tự động

```bash
#!/bin/bash
# .claude/hooks/inject-context.sh
# Inject context hữu ích trước mỗi message của user

echo "=== Auto Context ==="
echo "Branch: $(git rev-parse --abbrev-ref HEAD 2>/dev/null || echo 'no git')"
echo "Uncommitted files: $(git diff --name-only 2>/dev/null | head -5)"
echo "Last commit: $(git log --oneline -1 2>/dev/null)"
echo "=== End Context ==="
```

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [{"type": "command", "command": "bash .claude/hooks/inject-context.sh"}]
      }
    ]
  }
}
```

### 4.2 Inject environment info

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo \"Python: $(python --version 2>&1)\"; echo \"Node: $(node --version 2>/dev/null || echo 'N/A')\"; echo \"Docker: $(docker ps --format 'table {{.Names}}' 2>/dev/null | wc -l) containers running\""
          }
        ]
      }
    ]
  }
}
```

### 4.3 Validate prompt không chứa sensitive data

```bash
#!/bin/bash
# .claude/hooks/validate-prompt.sh
# Non-blocking check — chỉ warn

# Đọc prompt từ stdin hoặc env (tùy implementation)
PROMPT="${CLAUDE_USER_PROMPT:-}"

SENSITIVE_PATTERNS=("api_key" "secret_key" "password" "token" "credential")

for pattern in "${SENSITIVE_PATTERNS[@]}"; do
  if echo "$PROMPT" | grep -qi "$pattern"; then
    echo "NOTICE: Prompt có thể chứa sensitive data ($pattern). Đảm bảo không paste credentials thực." >&2
    break
  fi
done

exit 0  # Không block
```

---

## 5. Stop Hooks — Khi Claude kết thúc response

### 5.1 Desktop notification (macOS)

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "osascript -e 'display notification \"Claude đã hoàn thành\" with title \"Claude Code\" sound name \"Glass\"'"
          }
        ]
      }
    ]
  }
}
```

### 5.2 Desktop notification (Linux — notify-send)

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "notify-send 'Claude Code' 'Đã hoàn thành' --icon=dialog-information 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

### 5.3 Chạy test suite sau khi Claude xong

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "if git diff --name-only | grep -q '\.py$'; then echo 'Running tests...' >&2; pytest --tb=short -q 2>&1 | tail -10; fi"
          }
        ]
      }
    ]
  }
}
```

### 5.4 Log session summary

```bash
#!/bin/bash
# .claude/hooks/log-session.sh

LOG_FILE=".claude/session-log.md"
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null || echo 'no-git')
CHANGED=$(git diff --name-only 2>/dev/null | head -10)

cat >> "$LOG_FILE" << EOF

## Session: $TIMESTAMP
- Branch: $BRANCH
- Files changed: 
$(echo "$CHANGED" | sed 's/^/  - /')
EOF
```

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [{"type": "command", "command": "bash .claude/hooks/log-session.sh 2>/dev/null || true"}]
      }
    ]
  }
}
```

---

## 6. Hooks phức tạp — Script files riêng

Với hooks phức tạp, tạo script file và reference từ settings.json:

```
.claude/
`-- hooks/
    |-- post-edit.sh
    |-- pre-bash.sh
    |-- check-secrets.sh
    `-- notify.sh
```

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{"type": "command", "command": "bash $PWD/.claude/hooks/post-edit.sh"}]
      }
    ]
  }
}
```

**Lưu ý**: Dùng `$PWD/.claude/hooks/...` để đảm bảo path tuyệt đối.

---

## 7. Tất cả hooks trong một settings.json

Ví dụ project Python hoàn chỉnh:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "filter": "**/*.py",
        "hooks": [
          {
            "type": "command",
            "command": "ruff format \"$CLAUDE_FILE_PATH\" 2>&1 | head -3; ruff check \"$CLAUDE_FILE_PATH\" --fix --quiet 2>&1 | head -5"
          }
        ]
      },
      {
        "matcher": "Edit",
        "filter": "src/**/*.py",
        "hooks": [
          {
            "type": "command",
            "command": "bash $PWD/.claude/hooks/run-related-tests.sh"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "filter": "migrations/**",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Không sửa migrations — dùng alembic revision để tạo mới.' >&2 && exit 1"
          }
        ]
      },
      {
        "matcher": "Edit|Write",
        "filter": ".env",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Không edit .env trực tiếp — dùng .env.example.' >&2 && exit 1"
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo \"Git: $(git rev-parse --abbrev-ref HEAD) | Changed: $(git diff --name-only 2>/dev/null | wc -l | tr -d ' ') files\""
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "osascript -e 'display notification \"Claude done\" with title \"Claude Code\"' 2>/dev/null || notify-send 'Claude Code' 'Done' 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

---

## 8. Debugging hooks

### 8.1 Test hook script trực tiếp

```bash
# Set mock env vars
export CLAUDE_TOOL_NAME="Edit"
export CLAUDE_FILE_PATH="src/auth.py"
export CLAUDE_TOOL_INPUT='{"file_path": "src/auth.py", "content": "..."}'

# Chạy hook
bash .claude/hooks/post-edit.sh
echo "Exit: $?"
```

### 8.2 Debug log

```bash
#!/bin/bash
# Thêm debug logging vào hook
exec 2>> /tmp/claude-hook-debug.log
echo "=== Hook triggered: $CLAUDE_TOOL_NAME on $CLAUDE_FILE_PATH ===" >&2

# ... rest of hook
```

### 8.3 Common errors

| Error | Nguyên nhân | Fix |
|---|---|---|
| `Permission denied` | Script không có execute bit | `chmod +x .claude/hooks/script.sh` |
| `No such file` | Path relative thay vì absolute | Dùng `$PWD/.claude/hooks/...` |
| Hook không trigger | Matcher sai | Test matcher với `"matcher": ".*"` trước |
| Block không hoạt động | Exit code sai | Verify exit 1 thật sự return 1 |
| stdout không vào context | Script dùng `>&2` cho tất cả | Dùng stdout cho context, stderr cho user |

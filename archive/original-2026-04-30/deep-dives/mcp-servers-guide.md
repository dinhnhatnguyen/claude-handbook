# MCP Servers — Hướng dẫn đầy đủ

> Model Context Protocol (MCP) cho phép Claude kết nối với các service bên ngoài. File này giải thích kiến trúc, tất cả official servers, cách xây dựng custom server, và debug.

---

## 1. MCP là gì?

MCP (Model Context Protocol) là giao thức mở cho phép Claude sử dụng **tools từ external services** — database, GitHub, browser, file system, APIs — thông qua một interface chuẩn hóa.

```
Claude Code
    |
    | MCP Protocol
    |
    v
MCP Server (process chạy local hoặc remote)
    |
    v
External Service (GitHub API, PostgreSQL, Figma...)
```

Mỗi MCP server expose một tập tools. Claude nhìn thấy tools đó như built-in tools và có thể gọi chúng.

---

## 2. Token cost của MCP

Mỗi tool được MCP server expose tốn token trong context window:

```
Tool definition trung bình:    100–300 tokens
Server với 10 tools:          1,000–3,000 tokens
5 servers × 10 tools:         5,000–15,000 tokens (trước khi bắt đầu)
```

**Rule of thumb**: Mỗi MCP server thêm khoảng 1,000–2,000 tokens vào context. Với 5+ servers, overhead đáng kể.

---

## 3. Transport types

| Transport | Mô tả | Dùng khi |
|---|---|---|
| `stdio` | Server chạy như subprocess, giao tiếp qua stdin/stdout | Local tools, fastest, most common |
| `sse` | Server-Sent Events qua HTTP, persistent connection | Remote servers, real-time |
| `http` | HTTP request/response thông thường | Remote servers, stateless |

Hầu hết servers dùng `stdio`. SSE và HTTP dùng cho remote/cloud servers.

---

## 4. Official servers của Anthropic

### 4.1 server-filesystem

```bash
claude mcp add -s project filesystem -- npx -y @modelcontextprotocol/server-filesystem /allowed/path
```

Tools: `read_file`, `write_file`, `list_directory`, `search_files`, `get_file_info`

Dùng khi cần truy cập files **ngoài** project directory. Không cần cho files trong project (Claude tự đọc được).

### 4.2 server-github

```bash
claude mcp add -s project github -- npx -y @modelcontextprotocol/server-github
# Cần: GITHUB_TOKEN environment variable
```

Tools: `search_repositories`, `get_file_contents`, `create_issue`, `list_pull_requests`, `create_pull_request`, `get_pull_request`, `merge_pull_request`, `list_issues`, `add_comment`

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

### 4.3 server-postgres

```bash
claude mcp add -s project postgres -- npx -y @modelcontextprotocol/server-postgres "postgresql://user:pass@localhost/db"
```

Tools: `query` (read-only SQL), `list_tables`, `describe_table`

**Lưu ý**: Chỉ read-only theo mặc định — không thể INSERT/UPDATE qua tool này.

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "${DATABASE_URL}"]
    }
  }
}
```

### 4.4 server-sqlite

```bash
claude mcp add -s project sqlite -- npx -y @modelcontextprotocol/server-sqlite /path/to/database.db
```

Tools: `read_query`, `write_query`, `list_tables`, `describe_table`, `create_table`

Hỗ trợ cả read và write cho SQLite.

### 4.5 server-memory

```bash
claude mcp add -s user memory -- npx -y @modelcontextprotocol/server-memory
```

Tools: `create_entities`, `create_relations`, `add_observations`, `search_nodes`, `read_graph`

Persistence memory graph — Claude có thể lưu và truy xuất knowledge graph qua sessions.

---

## 5. Community servers phổ biến

### 5.1 Playwright (Browser automation)

```bash
claude mcp add -s user playwright -- npx @playwright/mcp@latest
```

Tools: `browser_navigate`, `browser_click`, `browser_fill`, `browser_screenshot`, `browser_wait`

Dùng khi: testing UI, scraping, automation.

```
> Mở trang login của app, điền credentials, verify dashboard loads
[Claude sẽ dùng Playwright tools]
```

### 5.2 Slack

```json
{
  "mcpServers": {
    "slack": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-slack"],
      "env": {
        "SLACK_BOT_TOKEN": "${SLACK_BOT_TOKEN}",
        "SLACK_TEAM_ID": "${SLACK_TEAM_ID}"
      }
    }
  }
}
```

Tools: `list_channels`, `post_message`, `get_channel_history`, `reply_to_thread`

### 5.3 Linear

```json
{
  "mcpServers": {
    "linear": {
      "command": "npx",
      "args": ["-y", "@linear/mcp-server"],
      "env": {
        "LINEAR_API_KEY": "${LINEAR_API_KEY}"
      }
    }
  }
}
```

Tools: `list_issues`, `create_issue`, `update_issue`, `list_projects`

### 5.4 Sentry

```json
{
  "mcpServers": {
    "sentry": {
      "command": "npx",
      "args": ["-y", "@sentry/mcp-server-sentry"],
      "env": {
        "SENTRY_AUTH_TOKEN": "${SENTRY_AUTH_TOKEN}",
        "SENTRY_ORG": "your-org"
      }
    }
  }
}
```

Tools: `list_issues`, `get_issue`, `list_events`, `resolve_issue`

### 5.5 AWS

```json
{
  "mcpServers": {
    "aws": {
      "command": "npx",
      "args": ["-y", "mcp-server-aws"],
      "env": {
        "AWS_ACCESS_KEY_ID": "${AWS_ACCESS_KEY_ID}",
        "AWS_SECRET_ACCESS_KEY": "${AWS_SECRET_ACCESS_KEY}",
        "AWS_REGION": "us-east-1"
      }
    }
  }
}
```

---

## 6. Remote MCP servers (SSE/HTTP)

Với servers chạy remote (cloud, team shared):

```json
{
  "mcpServers": {
    "team-tools": {
      "url": "https://mcp.internal.company.com/sse",
      "transport": "sse",
      "headers": {
        "Authorization": "Bearer ${INTERNAL_MCP_TOKEN}"
      }
    }
  }
}
```

---

## 7. Xây dựng custom MCP server

### 7.1 Khi nào nên build custom server?

- Internal APIs mà không có public MCP server
- Tools cần business logic đặc thù
- Wrapper cho legacy systems

### 7.2 Custom server bằng Python

```bash
pip install mcp
```

```python
# my_mcp_server.py
import asyncio
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp import types

server = Server("my-company-tools")

@server.list_tools()
async def list_tools() -> list[types.Tool]:
    return [
        types.Tool(
            name="get_deploy_status",
            description="Get deployment status for a service",
            inputSchema={
                "type": "object",
                "properties": {
                    "service": {
                        "type": "string",
                        "description": "Service name (e.g., api, worker)"
                    },
                    "environment": {
                        "type": "string",
                        "enum": ["staging", "production"]
                    }
                },
                "required": ["service", "environment"]
            }
        ),
        types.Tool(
            name="list_feature_flags",
            description="List all feature flags and their current state",
            inputSchema={
                "type": "object",
                "properties": {},
                "required": []
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    if name == "get_deploy_status":
        service = arguments["service"]
        env = arguments["environment"]
        
        # Gọi internal API
        import httpx
        async with httpx.AsyncClient() as client:
            resp = await client.get(
                f"https://deploy.internal.com/status/{env}/{service}",
                headers={"Authorization": f"Bearer {os.environ['DEPLOY_TOKEN']}"}
            )
            data = resp.json()
        
        return [types.TextContent(
            type="text",
            text=f"Service: {service}\nEnv: {env}\nStatus: {data['status']}\nVersion: {data['version']}\nDeployed: {data['deployed_at']}"
        )]
    
    elif name == "list_feature_flags":
        # ... internal flag service call
        flags = {"dark_mode": True, "new_checkout": False, "beta_api": True}
        return [types.TextContent(
            type="text",
            text="\n".join(f"{k}: {v}" for k, v in flags.items())
        )]
    
    raise ValueError(f"Unknown tool: {name}")

async def main():
    async with stdio_server() as streams:
        await server.run(streams[0], streams[1], server.create_initialization_options())

if __name__ == "__main__":
    asyncio.run(main())
```

### 7.3 Đăng ký custom server

```json
{
  "mcpServers": {
    "company-tools": {
      "command": "python",
      "args": ["/path/to/my_mcp_server.py"],
      "env": {
        "DEPLOY_TOKEN": "${DEPLOY_TOKEN}"
      }
    }
  }
}
```

### 7.4 Custom server bằng TypeScript

```typescript
// src/server.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  ListToolsRequestSchema,
  CallToolRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

const server = new Server(
  { name: "company-tools", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "get_pr_metrics",
      description: "Get metrics for a pull request",
      inputSchema: {
        type: "object",
        properties: {
          pr_number: { type: "number", description: "PR number" }
        },
        required: ["pr_number"]
      }
    }
  ]
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;
  
  if (name === "get_pr_metrics") {
    const { pr_number } = args as { pr_number: number };
    // ... fetch metrics
    return {
      content: [{ type: "text", text: `PR #${pr_number} metrics: ...` }]
    };
  }
  
  throw new Error(`Unknown tool: ${name}`);
});

const transport = new StdioServerTransport();
await server.connect(transport);
```

---

## 8. Debug MCP connections

### 8.1 Lệnh debug

```bash
# Xem tất cả servers đang active
claude mcp list

# Test kết nối của server cụ thể
claude mcp test github

# Xem logs
claude mcp logs github
claude mcp logs --tail 50 postgres

# Restart server bị stuck
claude mcp restart github

# Xóa và re-add
claude mcp remove github
claude mcp add -s project github -- npx -y @modelcontextprotocol/server-github
```

### 8.2 Test server thủ công

```bash
# Stdio server có thể test bằng pipe
echo '{"jsonrpc":"2.0","method":"tools/list","id":1}' | \
  GITHUB_TOKEN=ghp_xxx npx -y @modelcontextprotocol/server-github

# Xem tools được expose
```

### 8.3 Common errors

| Error | Nguyên nhân | Fix |
|---|---|---|
| `Connection refused` | Server process không start được | Check command, args, và env vars |
| `Tool not found` | Tool name sai | `claude mcp test` để xem tools available |
| `Authentication failed` | Token sai hoặc expired | Renew token, check env var name |
| `Timeout` | Server quá chậm | Dùng stdio thay SSE, hoặc tăng timeout |
| `Token budget exceeded` | Quá nhiều MCP tools | Bỏ servers không cần thiết |

### 8.4 MCP Inspector (community tool)

```bash
npx @modelcontextprotocol/inspector npx @modelcontextprotocol/server-github
```

Mở web UI để test MCP server tương tác trực tiếp.

---

## 9. Security considerations

### 9.1 Kiểm tra server source trước khi cài

MCP server chạy với quyền của user. Một server độc hại có thể:
- Đọc files trong home directory
- Gọi APIs với credentials của bạn
- Exfiltrate code

**Chỉ cài servers từ:**
- Official Anthropic repositories
- Vendors đáng tin (GitHub, Slack, Linear...)
- Code bạn tự đọc và hiểu

### 9.2 Principle of least privilege cho MCP

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", 
               "/specific/allowed/path"]
                           // Chỉ allow một thư mục cụ thể
    }
  }
}
```

### 9.3 Không commit credentials

```json
// .mcp.json — commit này
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"  // Reference env var, không hardcode
      }
    }
  }
}
```

Token được đọc từ environment — không bao giờ hardcode vào `.mcp.json`.

---

## 10. Chiến lược chọn servers

### 10.1 Bắt đầu tối thiểu

```
Session 1: Không có MCP server
-> Xem Claude cần tool gì không có
-> Thêm server khi thật sự gặp giới hạn
```

### 10.2 Stack đề xuất theo loại project

**Web application:**
```json
{"github": "...", "postgres": "..."}
```

**Data engineering:**
```json
{"postgres": "...", "sqlite": "...", "filesystem": "..."}
```

**DevOps/Platform:**
```json
{"github": "...", "slack": "...", "aws": "..."}
```

**Product team:**
```json
{"github": "...", "linear": "...", "sentry": "..."}
```

### 10.3 Khi nào KHÔNG dùng MCP

- Chỉ cần đọc file local — Claude tự làm được
- Chạy git commands — `Bash(git:*)` đủ rồi
- Gọi API đơn giản — `Bash(curl:*)` hoặc `Bash(python:*)` nhanh hơn
- Task một lần — overhead cài MCP không xứng

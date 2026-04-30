# Context Window Mechanics — Kỹ thuật sâu

> Giải thích kỹ thuật về cách context window hoạt động, cách đo lường, và các chiến lược tối ưu cụ thể.
>
> Ghi chú 2026-04-30: đây là deep dive từ bản gốc. Các con số về model/context/pricing có thể thay đổi; kiểm tra official docs trước khi dùng cho quyết định chi phí.

---

## 1. Context window là gì?

Context window là lượng text tối đa mà Claude có thể "nhìn thấy" trong một lần xử lý. Năm 2026, tất cả model Claude đều có context window **200,000 tokens**.

**Token là gì?**
```
1 token ≈ 4 ký tự tiếng Anh
1 token ≈ 0.75 từ tiếng Anh
1 token ≈ 2-3 ký tự tiếng Việt/Hán

"Hello world" = 2 tokens
"async def get_user_orders(user_id: int):" = ~10 tokens
File Python 500 dòng ≈ 3,000-5,000 tokens
```

---

## 2. Anatomy chi tiết của context window

### 2.1 Các thành phần

```
200,000 tokens tổng
|
|-- System layer (~10,000 tokens cố định)
|   |-- System prompt của Claude Code
|   |-- Built-in tool definitions (Read, Edit, Write, Bash...)
|   `-- MCP tool definitions (100-300 tokens/tool)
|
|-- Project layer (~3,000 tokens, load mỗi session)
|   |-- CLAUDE.md root
|   |-- Global ~/.claude/CLAUDE.md
|   `-- Subdirectory CLAUDE.md (khi relevant)
|
|-- Skills layer (~1,000 tokens, lazy loaded)
|   `-- SKILL.md đang active
|
|-- Conversation layer (tăng mỗi turn)
|   |-- User messages
|   |-- Assistant responses
|   `-- Tool call results (file reads, bash outputs...)
|
`-- Current turn (~variable)
    |-- Input đang xử lý
    `-- Thinking tokens (extended thinking)
```

### 2.2 Ví dụ breakdown thực tế

```
/context output sau 15 turns làm việc:

Component                    Tokens      % of total
-----------                  ------      ----------
System prompt                 2,000        1.0%
CLAUDE.md                     1,500        0.75%
Built-in tools                3,000        1.5%
MCP tools (3 servers)         4,500        2.25%
Skills (1 loaded)               800        0.4%
Conversation history         65,000       32.5%    <-- tăng nhất
File reads (tích lũy)         12,000        6.0%
-----------                  ------      ----------
Total                        88,800       44.4%
```

### 2.3 Tại sao conversation history phình nhanh

Mỗi turn thêm vào:
- User message: ~100-500 tokens
- Assistant response: ~200-2,000 tokens
- Tool calls + results: ~500-5,000 tokens (mỗi file read)

Turn 15 không phải chứa 15 messages — nó chứa tất cả nội dung từ turn 1 đến 15.

```
Turn 1:   +600 tokens
Turn 2:   +800 tokens  (bao gồm file read)
Turn 5:   +3,000 tokens (nhiều tool calls)
...
Turn 15:  tổng tích lũy ~65,000 tokens
```

---

## 3. Cơ chế re-read

Claude không có "short-term memory" — mỗi turn, nó đọc **toàn bộ context từ đầu**.

```
Turn 1:  Claude đọc [system + CLAUDE.md + turn1]
Turn 2:  Claude đọc [system + CLAUDE.md + turn1 + turn2]
Turn 10: Claude đọc [system + CLAUDE.md + turn1 + ... + turn10]
```

**Hệ quả:**
- Chi phí tăng tuyến tính với số turns
- Độ chính xác giảm khi context quá đầy (attention dilution)
- Thông tin đầu session bị "đẩy xa" khỏi điểm chú ý

### 3.1 Attention dilution

Nghiên cứu cho thấy LLM chú ý tốt nhất vào:
- Phần **đầu** của context (system prompt, CLAUDE.md)
- Phần **cuối** của context (messages gần nhất)
- Phần **giữa** bị giảm chú ý khi context quá dài

```
Context dài:
[System][CLAUDE.md] ... [middle messages - ít được chú ý] ... [recent messages]
   ^                                                                     ^
   Chú ý cao                                                    Chú ý cao
```

Đây là lý do tại sao, ở session dài, Claude "quên" những quyết định thiết lập từ đầu.

---

## 4. Compaction — Cơ chế và chiến lược

### 4.1 /compact hoạt động thế nào?

Khi chạy `/compact`, Claude:
1. Đọc toàn bộ conversation history hiện tại
2. Tạo ra một **summary** nén lại những gì quan trọng
3. Thay thế conversation history bằng summary đó
4. Giữ nguyên: system prompt, CLAUDE.md, tool definitions

```
Trước compact:                          Sau compact:
[System][CLAUDE.md][file reads]         [System][CLAUDE.md]
[turn1][turn2]...[turn15]               [summary ~2,000 tokens]
= 88,800 tokens                         = ~23,800 tokens
```

### 4.2 Compact không giữ lại tất cả

Summary là một compression — không phải verbatim copy. Thông tin có thể bị mất:
- Chi tiết cụ thể của đoạn code được thảo luận
- Lý do chi tiết đằng sau các quyết định
- File paths và line numbers chính xác

**Giải pháp:** Trước khi compact, nói rõ cho Claude biết cần giữ gì:

```
Trước khi compact, note lại:
1. Đã quyết định dùng Redis cho session storage vì [lý do cụ thể]
2. Schema bảng users có thêm column `preferences JSONB`
3. Đang ở bước 3/5: đã làm auth, đang làm profile, chưa làm settings
4. Bug #123 đã reproduce được ở file auth.py:145
5. Test command: pytest tests/auth/ -xvs --tb=short
```

### 4.3 Khi nào compact vs clear?

```
/compact — dùng khi:
  - Vẫn đang trong cùng một task/feature
  - Cần giữ context về decisions đã làm
  - Context 60-80% full
  - Muốn tiếp tục với "clean" history nhưng giữ essential info

/clear — dùng khi:
  - Chuyển sang task hoàn toàn khác
  - Context bị "nhiễm" vì nhiều trial-and-error
  - Muốn fresh start hoàn toàn
  - Context > 85% và compact không đủ
```

### 4.4 Auto-compaction

Claude Code có thể tự compact khi context gần đầy. Để kiểm soát:

```bash
# Xem khi context đạt ngưỡng nào thì auto-compact
claude --context-threshold 0.8  # compact ở 80%
```

---

## 5. Đo lường và monitoring

### 5.1 /context command

```
> /context

Context Window Usage
====================
Total capacity:  200,000 tokens

System prompt:     2,000 tokens   (1.0%)
CLAUDE.md:         1,500 tokens   (0.75%)
Tool definitions:  7,500 tokens   (3.75%)
  Built-in:        3,000
  MCP servers:     4,500 (3 servers × avg 1,500)
Skills:              800 tokens   (0.4%)
Conversation:     65,000 tokens  (32.5%)
  User messages:  18,000
  AI responses:   28,000
  Tool results:   19,000
Current turn:      2,000 tokens   (1.0%)
====================
Total used:       78,800 tokens  (39.4%)
Remaining:       121,200 tokens  (60.6%)
```

### 5.2 Token counts thực tế

Các thành phần hay bị underestimate:

| Thành phần | Token estimate |
|---|---|
| 1 MCP server trung bình | 1,000–2,000 tokens |
| File Python 100 dòng | 300–500 tokens |
| File Python 500 dòng | 1,500–2,500 tokens |
| package.json trung bình | 200–500 tokens |
| CLAUDE.md 60 dòng | 400–600 tokens |
| Một turn conversation | 500–2,000 tokens |
| Bash output 100 dòng | 500–800 tokens |

### 5.3 Xác định điều gì phình context nhất

```
> /context
# Xem phần "Tool results" trong Conversation
# Thường là nguồn lớn nhất khi Claude đọc nhiều files

# Hoặc dùng ccusage để xem breakdown chi tiết
ccusage --breakdown --session CURRENT_SESSION_ID
```

---

## 6. Chiến lược tối ưu theo loại task

### 6.1 Exploration task (tìm hiểu codebase)

**Vấn đề**: Đọc nhiều files → tích lũy trong context → phình nhanh.

**Giải pháp**: Spawn subagent Explore:
```
Spawn subagent Explore để:
- Tìm tất cả files liên quan đến authentication
- Map relationships giữa các modules
- List functions public của module auth/

Report lại: file paths, function signatures, key patterns.
Không cần code chi tiết.
```

Subagent đọc 30 files trong context window riêng, chỉ báo cáo ~500 tokens summary. Main context không bị ảnh hưởng.

### 6.2 Long implementation task

**Pattern**: Feature → sub-task → compact → sub-task tiếp theo

```
Turn 1-5:  Implement auth module
  -> /compact ["đã xong auth, schema X, pattern Y"]

Turn 6-10: Implement profile module dựa trên auth
  -> /compact ["đã xong profile, endpoint Z"]

Turn 11-15: Implement settings module
```

Mỗi compact ~65-70% context được giải phóng.

### 6.3 Debug session

**Vấn đề**: Thử nhiều hypothesis → nhiều file reads → context phình.

**Giải pháp**: Structure rõ ràng:
```
Turn 1: Spawn subagent để tìm files liên quan đến bug
Turn 2: Đọc chỉ N files quan trọng nhất (không phải tất cả)
Turn 3: Propose hypotheses
Turn 4: Test hypothesis 1
  -> Nếu đúng: implement fix
  -> Nếu sai: /compact ["H1 không phải root cause vì X, tiếp tục H2"]
Turn 5: Test hypothesis 2
```

### 6.4 Large file processing

**Vấn đề**: File 1,000 dòng = 5,000+ tokens vào context.

**Giải pháp**: Đọc theo phần, không đọc cả file:
```
# Thay vì:
Đọc toàn bộ file auth.py

# Hãy:
Grep trong auth.py tìm hàm validate_token và các hàm nó gọi.
Đọc chỉ những dòng đó.
```

---

## 7. Context và độ chính xác

### 7.1 Khi nào context đầy ảnh hưởng accuracy

Nghiên cứu nội bộ của Anthropic và cộng đồng cho thấy:
- Context < 50%: accuracy baseline
- Context 50-70%: nhẹ giảm trên tasks phức tạp
- Context > 80%: có thể bỏ qua instructions đầu session
- Context > 90%: risk cao bỏ qua constraints và patterns đã thiết lập

**Target**: Giữ dưới 70%.

### 7.2 Dấu hiệu accuracy đang giảm

1. Claude viết code mâu thuẫn với phần làm trước
2. Bỏ qua naming conventions đã thiết lập
3. Dùng error handling pattern khác với pattern agreed
4. Hỏi lại về thông tin đã được giải thích

### 7.3 Recovery khi accuracy giảm

```
Option 1: /compact
  + Giữ được progress
  - Summary có thể mất chi tiết

Option 2: /clear + restart với brief
  + Context hoàn toàn sạch
  - Phải recap lại context cần thiết

Option 3: /rewind đến trước điểm vấn đề
  + Giữ được conversation đến điểm tốt
  - Mất work sau điểm đó
```

---

## 8. File reads và context management

### 8.1 Mỗi file read tích lũy trong context

```python
# Một session điển hình:
Claude reads auth.py      (+2,000 tokens)
Claude reads models.py    (+1,500 tokens)
Claude reads utils.py     (+800 tokens)
Claude edits auth.py      (+diff output ~500 tokens)
Claude runs tests         (+test output ~1,000 tokens)
# Total tool results: ~5,800 tokens accumulated
```

Files không tự biến mất sau khi đọc — chúng ở lại context cả session.

### 8.2 Chiến lược đọc file hiệu quả

```
# Không tốt: đọc cả file mỗi khi cần
Đọc auth.py để tìm validate_token

# Tốt: grep trước, đọc chỉ phần cần
Grep trong auth.py tìm "def validate_token"
Đọc auth.py từ dòng 145 đến 180

# Tốt hơn: dùng subagent cho exploration
Spawn subagent để map module auth/ và report function signatures
```

### 8.3 /compact sau file-heavy exploration

Sau khi đọc nhiều files để tìm hiểu, compact ngay trước khi bắt đầu implement:

```
[Đã đọc 10 files để hiểu codebase]

Compact với note: "đã map module auth/ — key files là auth.py:145 và
middleware.py:89 — pattern dùng JWT với RS256, refresh via Redis"

[Bắt đầu implement với context sạch]
```

---

## Tóm tắt nhanh

| Vấn đề | Nguyên nhân | Giải pháp |
|---|---|---|
| Context phình nhanh | File reads tích lũy | Grep trước, đọc chính xác |
| Context phình nhanh | Many exploration turns | Subagent Explore |
| Accuracy giảm | Context > 70-80% | /compact với explicit notes |
| "Quên" decisions | Attention dilution | Update CLAUDE.md ngay |
| Cost tăng | Session dài | /compact sau mỗi sub-task |
| Cache miss | Idle > 5 phút | Làm việc liên tục hoặc /clear |

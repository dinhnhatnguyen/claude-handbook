# Prompt Caching — Cơ chế và tối ưu

> Phần này giải thích sâu cơ chế prompt caching của Anthropic, cách nó ảnh hưởng đến chi phí và tốc độ, và cách tối ưu cache hit rate trong Claude Code.
>
> Ghi chú 2026-04-30: đây là deep dive từ bản gốc. TTL/pricing/model support có thể thay đổi; kiểm tra prompt caching docs và pricing hiện tại trước khi tính chi phí.

---

## 1. Cơ chế hoạt động

### 1.1 Cache là gì?

Mỗi khi Claude nhận một request, nó phải xử lý (tokenize và encode) toàn bộ context — bao gồm system prompt, CLAUDE.md, tool definitions, conversation history, và file contents. Quá trình này tốn thời gian và chi phí.

**Prompt caching** lưu lại phần xử lý của một số đoạn text nhất định trong một khoảng thời gian. Khi request tiếp theo dùng lại đoạn text đó, Anthropic không cần xử lý lại — chỉ cần đọc từ cache.

### 1.2 Cache TTL — 5 phút

Cache có thời gian tồn tại (TTL) là **5 phút**. Sau 5 phút không có request mới dùng đến đoạn text đó, cache bị xóa.

```
Turn 1  (t=0:00)   → Xử lý toàn bộ context — full cost
Turn 2  (t=0:30)   → Cache hit trên system prompt + CLAUDE.md
Turn 3  (t=2:00)   → Cache hit
Turn 4  (t=4:45)   → Cache hit (vẫn trong 5 phút)
...
Idle    (t=5:10)   → Cache expired
Turn 5  (t=5:30)   → Cache miss — xử lý lại toàn bộ
```

### 1.3 Cái gì được cache?

Cache áp dụng theo **prefix** — đoạn text cố định ở đầu context:

```
[Cached]   System prompt
[Cached]   CLAUDE.md
[Cached]   Tool definitions (built-in + MCP)
[Cached]   Skills đang loaded
[Not cached] Conversation history (thay đổi mỗi turn)
[Not cached] File contents được đọc trong turn hiện tại
```

Conversation history **không được cache** vì nó thay đổi mỗi turn. Đây là lý do chính khiến session dài tốn nhiều tiền — phần history phình lên liên tục.

### 1.4 Cost khi cache hit

| Token type | Cache miss | Cache hit |
|---|---|---|
| Input tokens | $3.00/1M (Sonnet) | $0.30/1M (Sonnet) — giảm 90% |
| Output tokens | $15.00/1M | $15.00/1M — không đổi |

Chỉ **input tokens** được giảm khi cache hit. Output tokens luôn tính full price.

---

## 2. Cache hit rate thực tế

### 2.1 Tính toán cache benefit

Ví dụ session điển hình (Sonnet 4.6):

```
Context cố định (cached):
  System prompt:       2,000 tokens
  CLAUDE.md:           1,500 tokens
  Tool definitions:    3,000 tokens
  Total cached:        6,500 tokens

Session 20 turns:
  Turn 1 (cache miss):  6,500 × $3/1M = $0.0195
  Turn 2–20 (cache hit): 6,500 × $0.30/1M × 19 = $0.0370
  
  Tanh vì không cache: 6,500 × $3/1M × 20 = $0.39
  Với cache: $0.0195 + $0.0370 = $0.057
  Tiết kiệm: ~85%
```

### 2.2 Tại sao idle làm mất cache?

```
active session:    [----][----][----]  < 5 phút giữa các turn
                   hit   hit   hit     cache warm

idle session:      [----]        [----]
                   hit      x    miss   cache expired
                         5+ phút
```

Đây là lý do nên **làm việc liên tục** hơn là để session idle.

---

## 3. Tối ưu cache hit rate

### 3.1 Giữ phần đầu context ổn định

Cache hoạt động theo prefix — phần đầu context càng ổn định, càng nhiều được cache.

**Tốt** (phần đầu cố định):
```
System prompt (không đổi)
CLAUDE.md (không đổi)
Tool definitions (không đổi)
---
[turn 1 conversation]
[turn 2 conversation]  <- chỉ phần này thay đổi
```

**Xấu** (thay đổi prefix):
```
[Thông tin ngày giờ động] <- làm invalidate cache của mọi thứ phía sau
System prompt
CLAUDE.md
```

### 3.2 CLAUDE.md ngắn là cache-efficient

CLAUDE.md được cache, nhưng nó cũng tốn cache slots. File ngắn:
- Được cache nhanh hơn
- Ít ảnh hưởng đến 5-minute TTL bucket
- Để lại nhiều cache space cho conversation

### 3.3 Không thay đổi tool definitions giữa session

Mỗi lần thêm/xóa MCP server, tool definitions thay đổi → cache miss cho toàn bộ phần sau.

Cấu hình MCP servers ổn định trước khi bắt đầu session.

### 3.4 Compact thay vì clear khi có thể

```
/compact  -> nén history, giữ phần đầu (system prompt, CLAUDE.md) nguyên
            -> phần đầu vẫn cached, chỉ mất cache trên history

/clear    -> reset hoàn toàn
            -> toàn bộ cache invalidated, turn tiếp theo là full miss
```

---

## 4. Monitoring cache performance

### 4.1 Dùng ccusage

```bash
# Xem cache hit rate theo session
ccusage

# Chi tiết theo từng turn
ccusage --breakdown --session SESSION_ID

# Output:
# Session abc123
# Total input:    45,230 tokens   ($0.136)
# Cache hits:     38,400 tokens   ($0.012)  [85% hit rate]
# Cache misses:    6,830 tokens   ($0.020)
# Total output:   12,100 tokens   ($0.182)
# Total cost:     $0.214
```

### 4.2 API response headers

Khi dùng API trực tiếp, cache metrics có trong response:

```python
response = client.messages.create(...)

# Cache metrics
print(response.usage.cache_creation_input_tokens)  # tokens mới được cache
print(response.usage.cache_read_input_tokens)       # tokens đọc từ cache
print(response.usage.input_tokens)                  # tokens không cached
```

---

## 5. Prompt caching trong Claude API

### 5.1 Cache control headers

Khi dùng API, bạn có thể chủ động mark cache breakpoints:

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": "You are a senior engineer...",
            "cache_control": {"type": "ephemeral"}  # Cache điểm này
        }
    ],
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": large_document,           # Tài liệu lớn
                    "cache_control": {"type": "ephemeral"}  # Cache
                },
                {
                    "type": "text",
                    "text": "Summarize the above."   # Query thực sự
                }
            ]
        }
    ]
)
```

### 5.2 Pattern: Cache document, query thay đổi

```python
def query_document(document: str, questions: list[str]) -> list[str]:
    """Cache document một lần, query nhiều lần."""
    answers = []
    
    for i, question in enumerate(questions):
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=512,
            messages=[
                {
                    "role": "user",
                    "content": [
                        {
                            "type": "text",
                            "text": document,
                            "cache_control": {"type": "ephemeral"}  # Cached sau turn 1
                        },
                        {
                            "type": "text",
                            "text": question  # Thay đổi mỗi turn, không cached
                        }
                    ]
                }
            ]
        )
        answers.append(response.content[0].text)
        # Turn 1: full cost cho document
        # Turn 2+: cache hit cho document — chỉ trả cho question
    
    return answers
```

### 5.3 Minimum cacheable size

Cache chỉ áp dụng cho đoạn text đủ dài. Minimum:
- Claude 3.5 Sonnet: 1,024 tokens
- Claude 3.5 Haiku: 2,048 tokens

Đoạn text ngắn hơn sẽ không được cache dù có `cache_control`.

---

## 6. Khi caching không giúp được

### 6.1 Tasks một lần (one-shot)

```bash
# Mỗi lần chạy là fresh request — không có gì để cache
claude --print "Review file này" < code.py
```

Cache chỉ có lợi khi có nhiều turns trong cùng session, hoặc khi document lớn được query nhiều lần.

### 6.2 Pipeline với fresh sessions

```yaml
# CI/CD: mỗi job là fresh session
- name: Claude review
  run: claude --print "Review..." < diff.txt
```

Không có cache benefit. Nhưng vẫn có thể dùng Batch API (50% cheaper) thay thế.

### 6.3 Sessions idle quá 5 phút

Nếu workflow của bạn thường xuyên để session idle hơn 5 phút, cache benefit giảm đáng kể. Cân nhắc:
- Làm việc liên tục hơn
- `/clear` và start fresh (rẻ hơn là re-read expired cache)

---

## 7. Disable caching — khi nào và tại sao

```bash
export DISABLE_PROMPT_CACHING=1
```

Chỉ disable khi:
- Debug cache behavior — muốn thấy full cost không có cache
- Testing exact token counts
- Profiling performance

Không disable trong production — tốn tiền không cần thiết.

---

## Tóm tắt

| Situation | Cache benefit |
|---|---|
| Session tích cực, turn trong 5 phút | Cao — 80–90% trên system/tools |
| Session idle > 5 phút | Không có — cache expired |
| One-shot non-interactive | Không có |
| Document lớn query nhiều lần | Rất cao — cache document sau turn 1 |
| CI/CD fresh sessions | Không có — dùng Batch API thay thế |

**Rule of thumb**: Với session interactive thông thường, prompt caching tự động tiết kiệm 70–85% chi phí input tokens cho phần context cố định. Không cần cấu hình thêm — bật mặc định.

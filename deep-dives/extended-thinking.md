# Extended Thinking — Hướng dẫn đầy đủ

> Extended Thinking cho phép Claude suy nghĩ sâu trước khi trả lời. File này giải thích cơ chế, khi nào nên dùng, cách kiểm soát, và API usage.
>
> Ghi chú 2026-04-30: đây là deep dive từ bản gốc. Model name, thinking behavior và pricing trong file này có thể lỗi thời; kiểm tra official docs hiện tại trước khi áp dụng.

---

## 1. Cơ chế hoạt động

### 1.1 Standard response vs Extended Thinking

**Standard response:**
```
User: "Design auth system"
                |
                v
Claude: [trả lời ngay]
```

**Extended Thinking:**
```
User: "Design auth system"
                |
                v
Claude: [thinking block — nội bộ]
  - Xem xét approach A: pros/cons
  - Xem xét approach B: pros/cons
  - Identify risks của mỗi approach
  - Chọn approach tốt nhất
  - Draft câu trả lời
                |
                v
Claude: [câu trả lời dựa trên reasoning trên]
```

Thinking block **không hiển thị** trong conversation thông thường. Chỉ có output cuối được trả về.

### 1.2 Thinking tokens là gì?

Thinking tokens là token riêng biệt dùng cho quá trình reasoning — không phải output tokens. Chúng:
- Không tính vào output token cost theo cách thông thường
- Có giá riêng (xem pricing)
- Không được cache
- Không hiển thị trong conversation history

### 1.3 Adaptive vs Fixed thinking

**Adaptive thinking** (Opus 4.7, Sonnet 4.6): Claude tự quyết định khi nào cần think nhiều hay ít dựa trên độ phức tạp của task.

**Fixed thinking**: Bạn set budget cố định — Claude sẽ dùng nhiều nhất budget đó.

---

## 2. Khi nào Extended Thinking thực sự có giá trị

### 2.1 Tasks mà thinking giúp được nhiều

**Multi-step reasoning với dependency:**
```
"Design database schema cho social network với:
- Users có thể follow nhau
- Posts có comments và reactions
- Notification system real-time
- Cần optimize cho 1M users"

-> Thinking: diagram relationships, identify N+1 risks, 
   consider denormalization tradeoffs, design indexes
```

**Architecture decisions với nhiều constraints:**
```
"Nên dùng message queue hay event streaming?
Constraints: team nhỏ, budget thấp, latency < 100ms, 
eventual consistency được phép cho non-critical paths"

-> Thinking: weigh Kafka vs RabbitMQ vs SQS vs Redis Streams,
   match constraints với capabilities của từng option
```

**Security analysis:**
```
"Review auth flow này tìm vulnerabilities"

-> Thinking: step through each auth state, consider
   attack vectors (CSRF, timing attacks, session fixation...),
   check token lifecycle
```

**Complex debugging:**
```
"User báo cáo: sau khi deploy, 0.1% requests bị timeout
nhưng chỉ sau 3 giờ uptime. Logs không rõ ràng."

-> Thinking: brainstorm causes (connection pool exhaustion,
   memory leak, cron job interference, GC pauses...),
   prioritize by probability
```

### 2.2 Tasks mà thinking không giúp (và tốn tiền thêm)

- Rename biến hay function
- Format code
- Add type hints
- Simple CRUD endpoints theo pattern đã có
- Copy-paste với small modification
- Fix obvious syntax errors
- Tìm file hoặc grep

**Rule of thumb**: Nếu một senior dev có thể làm task trong < 2 phút mà không cần "dừng lại nghĩ", extended thinking không cần thiết.

---

## 3. Kiểm soát Extended Thinking trong Claude Code

### 3.1 Effort levels qua /model picker

```
> /model
```

Picker hiện danh sách models và effort levels:

| Level | Thinking budget | Use case |
|---|---|---|
| None | 0 | Simple tasks |
| Low | ~1,024 tokens | Light reasoning |
| Medium | ~5,000 tokens | Standard complex tasks |
| High | ~10,000 tokens | Hard problems |
| Max (xhigh) | ~32,000 tokens | Research-grade |

### 3.2 Environment variable

```bash
# Set globally
export MAX_THINKING_TOKENS=10000

# Trong settings.json
{
  "env": {
    "MAX_THINKING_TOKENS": "10000"
  }
}
```

### 3.3 Disable adaptive thinking

```bash
# Sonnet 4.6 và các models cũ hơn — có thể disable
export CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING=1

# Opus 4.7 — luôn dùng adaptive, không disable được
```

---

## 4. Prompt engineering cho Extended Thinking

### 4.1 Prompts kích hoạt thinking hiệu quả

Không cần prompt đặc biệt — Claude tự quyết định khi nào cần think. Nhưng một số patterns giúp:

**Yêu cầu phân tích trước:**
```
Trước khi đề xuất solution, hãy:
1. Liệt kê tất cả approaches có thể
2. Với mỗi approach, nêu pros và cons cụ thể
3. Chọn approach tốt nhất với justification rõ ràng
```

**Yêu cầu xem xét edge cases:**
```
Design solution cho [problem].
Đặc biệt chú ý đến:
- What happens khi [edge case 1]?
- How does it behave under [edge case 2]?
- Failure modes?
```

**Explicit thinking request:**
```xml
<thinking>
Phân tích trade-offs của các approaches trước khi trả lời.
</thinking>
```

### 4.2 Khi thinking không đủ

Nếu output vẫn không đạt yêu cầu sau thinking:
- Thêm constraints cụ thể hơn trong prompt
- Chia task thành subtasks nhỏ hơn, mỗi task một turn
- Switch sang Opus nếu đang dùng Sonnet
- Tăng thinking budget (effort level cao hơn)

---

## 5. Extended Thinking qua API

### 5.1 Basic usage

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=16000,
    thinking={
        "type": "enabled",
        "budget_tokens": 10000
    },
    messages=[{
        "role": "user",
        "content": "Design a distributed rate limiter for 100K RPS with <1ms latency overhead."
    }]
)

for block in response.content:
    if block.type == "thinking":
        print("=== THINKING ===")
        print(block.thinking)
        print("=== END THINKING ===\n")
    elif block.type == "text":
        print("=== ANSWER ===")
        print(block.text)
```

### 5.2 Streaming thinking

```python
with client.messages.stream(
    model="claude-opus-4-7",
    max_tokens=16000,
    thinking={"type": "enabled", "budget_tokens": 8000},
    messages=[{"role": "user", "content": "..."}]
) as stream:
    for event in stream:
        if hasattr(event, 'type'):
            if event.type == 'content_block_start':
                if event.content_block.type == 'thinking':
                    print("[Thinking started...]")
                elif event.content_block.type == 'text':
                    print("[Answer started...]")
            elif event.type == 'content_block_delta':
                if hasattr(event.delta, 'thinking'):
                    print(event.delta.thinking, end='', flush=True)
                elif hasattr(event.delta, 'text'):
                    print(event.delta.text, end='', flush=True)
```

### 5.3 Multi-turn với thinking

Thinking từ turn trước được preserve trong conversation history:

```python
messages = []

# Turn 1 với thinking
turn1_response = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=8000,
    thinking={"type": "enabled", "budget_tokens": 5000},
    messages=[{"role": "user", "content": "Design auth system architecture"}]
)

# Thêm response vào history (bao gồm thinking blocks)
messages.append({"role": "user", "content": "Design auth system architecture"})
messages.append({"role": "assistant", "content": turn1_response.content})

# Turn 2 — có thể reference thinking từ turn 1
turn2_response = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=8000,
    thinking={"type": "enabled", "budget_tokens": 5000},
    messages=messages + [
        {"role": "user", "content": "Now detail the token refresh flow"}
    ]
)
```

### 5.4 Budget sizing guidance

```python
THINKING_BUDGETS = {
    "simple_reasoning": 1024,       # ~$0.003 extra
    "medium_analysis": 5000,        # ~$0.015 extra
    "complex_design": 10000,        # ~$0.030 extra
    "research_grade": 32000,        # ~$0.096 extra
}

def choose_budget(task_complexity: str) -> int:
    return THINKING_BUDGETS.get(task_complexity, 5000)
```

**Pricing thinking tokens** (Opus 4.7):
- Input: $15/1M tokens
- Thinking: $15/1M tokens (same as input)
- Output: $75/1M tokens

10,000 thinking tokens = $0.15 — thường đáng cho complex tasks.

---

## 6. Ví dụ thực tế: So sánh output

### Prompt: "Implement distributed lock trong Python với Redis"

**Không có Extended Thinking:**
```python
import redis
import uuid

def acquire_lock(client, lock_name, timeout=10):
    identifier = str(uuid.uuid4())
    return client.set(lock_name, identifier, nx=True, ex=timeout)
```

Đúng nhưng thiếu nhiều edge cases.

**Với Extended Thinking (budget 5000):**

Thinking process (không hiển thị cho user):
```
- SET NX là atomic — tốt
- Nhưng nếu process crash sau khi acquire? timeout handle được
- Nếu timeout quá ngắn và operation chưa xong? timeout drift
- Nếu Redis primary crash trước khi replicate? split-brain
- Redlock algorithm giải quyết split-brain với N Redis instances
- Cần implement: acquire, release, renewal
- Release cần Lua script để atomic check + delete
```

Output:
```python
import redis
import uuid
import time
from contextlib import contextmanager

class RedisLock:
    """
    Distributed lock với Redis.
    
    Dùng SET NX EX atomic operation để acquire.
    Release qua Lua script đảm bảo atomic check-and-delete.
    Tự động renew để tránh timeout drift với long operations.
    """
    
    RELEASE_SCRIPT = """
    if redis.call('get', KEYS[1]) == ARGV[1] then
        return redis.call('del', KEYS[1])
    else
        return 0
    end
    """
    
    def __init__(self, client: redis.Redis, name: str, timeout: int = 30):
        self.client = client
        self.name = f"lock:{name}"
        self.timeout = timeout
        self.identifier = str(uuid.uuid4())
        self._release_script = client.register_script(self.RELEASE_SCRIPT)
    
    def acquire(self, blocking: bool = True, blocking_timeout: float = -1) -> bool:
        start = time.monotonic()
        while True:
            if self.client.set(self.name, self.identifier, nx=True, ex=self.timeout):
                return True
            if not blocking:
                return False
            if blocking_timeout >= 0 and time.monotonic() - start > blocking_timeout:
                return False
            time.sleep(0.1)
    
    def release(self) -> bool:
        return bool(self._release_script(keys=[self.name], args=[self.identifier]))
    
    def renew(self, additional_time: int = None) -> bool:
        """Extend lock TTL nếu vẫn đang hold."""
        return bool(self.client.expire(self.name, additional_time or self.timeout))

@contextmanager
def distributed_lock(client: redis.Redis, name: str, timeout: int = 30):
    lock = RedisLock(client, name, timeout)
    acquired = lock.acquire(blocking=True, blocking_timeout=timeout)
    if not acquired:
        raise TimeoutError(f"Could not acquire lock '{name}'")
    try:
        yield lock
    finally:
        lock.release()
```

Thinking tạo ra output đầy đủ hơn nhiều với cùng một prompt.

---

## 7. Limitations

### 7.1 Thinking không được cache

Thinking tokens không được prompt cache. Mỗi turn mới, thinking phải được generate lại từ đầu. Chi phí thinking không giảm theo cache.

### 7.2 Thinking không luôn visible

Trong Claude Code interactive mode, thinking process không hiển thị cho user. Chỉ output cuối được hiện.

Qua API (ví dụ 5.1), thinking content có thể đọc được.

### 7.3 Minimum budget

Budget phải ≥ 1,024 tokens. Budget nhỏ hơn sẽ bị ignore hoặc gây error.

### 7.4 Không phải silver bullet

Extended thinking giúp với reasoning tasks. Nó không giúp với:
- Thông tin Claude không có trong training data
- Real-time information
- Tasks cần tool calls chứ không phải reasoning
- Giảm hallucination về facts cụ thể

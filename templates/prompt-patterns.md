# Prompt Patterns cho Claude Code

## 1. Explore-only

```text
Hãy explore codebase cho task này, chưa sửa file.

Task: [mô tả]

Cần trả về:
1. Entry points/file liên quan.
2. Current behavior.
3. Proposed change.
4. Risks.
5. Tests/commands nên chạy.

Hãy search trước, chỉ đọc đoạn cần thiết.
```

## 2. Bug fix

```text
Bug: [actual behavior]
Expected: [expected behavior]
Evidence: [log/screenshot/test/user report]

Hãy:
1. Trace root cause.
2. Sửa scope hẹp nhất.
3. Thêm regression test nếu hợp lý.
4. Chạy verification liên quan.
5. Tóm tắt file changed + test result.
```

## 3. Feature

```text
Feature: [mô tả]
User flow: [ai làm gì]
Acceptance criteria:
- [criterion 1]
- [criterion 2]

Constraints:
- Giữ API hiện tại nếu có thể.
- Không thêm dependency mới nếu không cần.
- Theo pattern hiện có trong codebase.

Hãy plan ngắn, implement, verify.
```

## 4. Refactor safe

```text
Refactor [module/file] để [mục tiêu].

Rules:
- Behavior must stay identical.
- Public API must stay identical.
- Diff should be small and mechanical.
- If tests are weak, add or propose safety checks first.

Sau khi sửa, chạy test liên quan và review diff for regressions.
```

## 5. Code review

```text
Review diff hiện tại như senior reviewer.

Findings first, ordered by severity.
Focus:
- Bugs/regressions
- Security/data-loss risk
- Missing tests
- Edge cases

Skip style nitpicks unless they affect correctness.
```

## 6. Context recovery

```text
Hãy tạo state summary ngắn để tôi có thể /clear hoặc /compact.

Include:
- Goal
- Files changed
- Important decisions
- Current failing/passing tests
- Next 3 steps
- Anything explicitly out of scope
```

## 7. Cost-saving instruction

```text
Làm task này với token tối ưu:
- Search trước khi đọc file.
- Không paste output dài; tóm tắt những phần quan trọng.
- Chỉ đọc/sửa file liên quan.
- Final ngắn: result, files, verification, risks.
```

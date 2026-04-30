# Daily Claude Code Checklist

## 1. Trước khi bắt đầu

- [ ] Chạy `git status` để biết workspace đang sạch hay có thay đổi sẵn.
- [ ] Nếu task mới khác task cũ, dùng `/clear`.
- [ ] Prompt có đủ goal, scope và acceptance criteria.
- [ ] Nếu task rủi ro cao, yêu cầu explore/plan trước khi edit.
- [ ] Nếu có secret/local env, đảm bảo đã deny trong settings.

## 2. Prompt khởi động nhanh

```text
Task: [mô tả ngắn]

Goal:
- [kết quả cần đạt]

Scope:
- Chỉ sửa [file/module]
- Không sửa [ngoài scope]

Acceptance criteria:
- [hành vi đúng]
- [test/build/check]

Working rules:
- Search/doc code trước khi edit.
- Giữ diff nhỏ.
- Chạy verification liên quan trước final.
```

## 3. Trong lúc làm

- [ ] Bảo Claude search trước khi đọc file lớn.
- [ ] Nếu output bắt đầu dài và lan man, yêu cầu "state summary + next step only".
- [ ] Nếu context dài, dùng `/compact Focus on files edited, decisions, failing tests, next steps`.
- [ ] Nếu Claude sửa rộng quá scope, dừng lại và yêu cầu review diff.
- [ ] Không approve lệnh nguy hiểm nếu chưa hiểu lệnh đó làm gì.

## 4. Trước khi kết thúc

- [ ] Chạy test/build liên quan.
- [ ] Review `git diff`.
- [ ] Yêu cầu Claude review diff theo stance code reviewer.
- [ ] Ghi lại test đã chạy và test chưa chạy.
- [ ] Nếu cần quay lại sau, dùng `/rename` cho session hoặc `claude --continue`.

## 5. Dấu hiệu cần reset context

- Claude quên yêu cầu lặp lại.
- Claude đọc nhầm module.
- Claude sửa unrelated files.
- Session chuyển sang task mới.
- Test/log cũ không còn liên quan.

Dùng `/clear` nếu là task mới. Dùng `/compact` nếu vẫn là cùng task.

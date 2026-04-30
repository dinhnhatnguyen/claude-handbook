# RAG Setup Template

Copy template này vào `docs/rag-strategy.md` hoặc issue thiết kế trước khi build retrieval.

## 1. Mục tiêu

- Use case:
- Người dùng hỏi gì?
- Câu trả lời cần citation không?
- RAG phục vụ Claude Code, Claude.ai Project, API app hay internal tool?

## 2. Corpus

| Nguồn | Owner | Update cadence | Độ tin cậy | Ghi chú |
|---|---|---|---|---|
| `docs/` | | | High/Medium/Low | |
| Confluence/Notion | | | | |
| Tickets/Issues | | | | |
| API specs | | | | |

## 3. Metadata bắt buộc

- `source`
- `title`
- `heading_path`
- `chunk_id`
- `updated_at`
- `version`
- `owner`
- `tags`

## 4. Chunking

- Strategy: theo heading/section/code block.
- Target chunk size:
- Overlap:
- Những phần loại bỏ: nav/footer/boilerplate/generated text.
- Quy tắc cho code blocks:

## 5. Contextual Retrieval

Prompt contextualizer:

```text
<document>
{{WHOLE_DOCUMENT}}
</document>

<chunk>
{{CHUNK_CONTENT}}
</chunk>

Hãy viết 1-3 câu ngắn để đặt chunk này vào đúng ngữ cảnh của tài liệu.
Chỉ trả về context ngắn, không giải thích thêm.
```

Lưu:

```text
contextualized_chunk = contextual_note + "\n\n" + original_chunk
```

## 6. Retrieval

- Semantic embeddings model:
- Keyword/BM25 engine:
- Initial top-N:
- Reranker:
- Final top-K:
- Dedup key:
- Filters: version, product, language, owner, source type.

## 7. Prompt assembly

Format trả về cho Claude:

```text
Source: [source#heading]
Title: [title]
Updated: [updated_at]
Score: [score]
Content:
[chunk]
```

Rules:

- Chỉ đưa top-K chunks.
- Luôn đưa source/title.
- Nếu nguồn không đủ, nói rõ thiếu gì.
- Không trộn docs khác version nếu không có lý do.

## 8. Claude Code integration

Chọn một:

- [ ] Local script: `./scripts/search-knowledge "query"`
- [ ] MCP tool: `search_knowledge(query, filters)`
- [ ] Claude.ai Project RAG
- [ ] API tool returning `search_result` content blocks

Prompt dùng trong CLAUDE.md hoặc command:

```text
Khi task cần kiến thức ngoài codebase, hãy dùng retrieval trước.
Chỉ dùng top chunks có source/title rõ. Nếu retrieval thiếu bằng chứng, nói rõ thay vì suy đoán.
```

## 9. Evals

Tạo 20-50 câu hỏi thật:

```yaml
- question: ""
  expected_sources:
    - ""
  expected_facts:
    - ""
```

Metrics:

- Retrieval recall@K.
- Answer faithfulness.
- Citation accuracy.
- Latency.
- Tokens injected per answer.

## 10. Maintenance

- Re-index trigger:
- Owner review cadence:
- Stale docs policy:
- Access control:
- Secret/sensitive content policy:

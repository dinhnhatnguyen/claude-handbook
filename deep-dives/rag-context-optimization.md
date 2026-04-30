# RAG và Context Optimization cho Claude Code

> Mục tiêu: dùng RAG để đưa đúng kiến thức vào Claude, không nhồi toàn bộ tài liệu vào context.

## 1. RAG là gì?

RAG là Retrieval-Augmented Generation: hệ thống tìm các đoạn tài liệu liên quan rồi đưa chúng vào prompt để Claude trả lời hoặc hành động dựa trên nguồn cụ thể.

RAG gồm 2 pha:

1. Offline/indexing: chuẩn hóa tài liệu, chia chunk, tạo embeddings/BM25 index, lưu metadata.
2. Online/retrieval: nhận query, tìm top chunks, rerank, đưa top-K vào Claude.

## 2. Khi nào không cần RAG?

Không cần build RAG riêng nếu:

- Tài liệu nhỏ và ổn định.
- Claude Code có thể search repo trực tiếp bằng `rg`.
- Bạn chỉ dùng vài file docs cho một task ngắn.
- Dùng Claude.ai Projects và RAG tự động đã đủ.

Nếu knowledge base nhỏ hơn context window và dùng lặp lại, đưa trực tiếp vào prompt + prompt caching có thể đơn giản hơn RAG.

## 3. Khi nào nên dùng RAG?

Nên dùng RAG nếu:

- Docs quá lớn để đưa vào context.
- Docs thay đổi thường xuyên.
- Cần truy vấn qua nhiều nguồn: repo, Notion, Google Drive, tickets, database.
- Cần citation/source để kiểm chứng.
- Cần tìm exact identifiers như error code, API endpoint, config key.
- Nhiều người dùng hỏi nhiều câu khác nhau trên cùng knowledge base.

## 4. Setup tối thiểu

```text
docs/
  -> parser
  -> chunker
  -> contextualizer
  -> embeddings + BM25
  -> vector/search store
  -> retriever
  -> reranker
  -> Claude prompt/tool result
```

Với prototype local:

- Parser: đọc Markdown/PDF/text.
- Chunker: chia theo heading.
- Keyword: SQLite FTS, Tantivy, Meilisearch hoặc Postgres full-text.
- Vector: pgvector, LanceDB, Chroma, Qdrant.
- Rerank: dùng model reranker hoặc Claude với top-N nhỏ nếu volume thấp.
- Integration: local script hoặc MCP server.

## 5. Chunking

Chunk tốt phải vừa tìm được, vừa đủ nghĩa khi đứng riêng.

Khuyến nghị:

- Chia theo cấu trúc tài liệu trước: heading, section, code block.
- Giữ metadata đầy đủ.
- Tránh chunk quá nhỏ kiểu một câu không có chủ thể.
- Tránh chunk quá lớn khiến retrieval trả về nhiều noise.
- Với tài liệu kỹ thuật, giữ code và giải thích gần nhau.

Metadata tối thiểu:

```json
{
  "source": "docs/payments/retry-policy.md",
  "title": "Payment retry policy",
  "heading_path": "Checkout > Timeout",
  "chunk_id": "payments-retry-timeout-001",
  "updated_at": "2026-04-30",
  "version": "v2",
  "tags": ["payments", "checkout", "timeout"]
}
```

## 6. Contextual Retrieval

Traditional RAG thường làm mất context vì chunk bị tách khỏi tài liệu gốc. Contextual Retrieval thêm một đoạn mô tả ngắn vào chunk trước khi index.

Ví dụ:

```text
Original chunk:
"The default timeout is 30 seconds."

Contextualized chunk:
"This chunk is from checkout service v2 retry policy, section Timeout. It defines the default gateway timeout for payment authorization. The default timeout is 30 seconds."
```

Contextualized chunk được dùng cho cả embeddings và BM25 để query dễ tìm đúng hơn.

## 7. Hybrid retrieval

Embeddings giỏi bắt nghĩa, nhưng có thể miss exact match. BM25/keyword giỏi tìm mã lỗi, tên hàm, endpoint, config key.

Pipeline tốt:

1. Semantic search lấy top-N.
2. BM25 lấy top-N.
3. Merge + deduplicate.
4. Rerank theo query.
5. Đưa top-K vào Claude.

Gợi ý ban đầu:

- Initial top-N: 50-150.
- Final top-K: 5-20.
- Với coding task: bắt đầu top 5-8.
- Với research/legal/spec comparison: có thể top 10-20.

## 8. Reranking

Rerank giúp giảm noise trước khi tốn context.

Prompt rerank đơn giản:

```text
Query: {{query}}

Chunks:
{{chunks_with_source}}

Hãy chấm mỗi chunk từ 0-3:
0 = không liên quan
1 = hơi liên quan
2 = liên quan
3 = trực tiếp trả lời query

Trả về danh sách chunk_id theo thứ tự tốt nhất, không giải thích.
```

Với volume lớn, nên dùng reranker model riêng thay vì Claude để tiết kiệm.

## 9. Prompt assembly

Đừng đưa raw chunk vô tổ chức. Format đều:

```text
<search_result>
Source: docs/payments/retry-policy.md#timeout
Title: Payment retry policy
Updated: 2026-04-30
Content:
...
</search_result>
```

System/task instruction:

```text
Chỉ dùng search results bên dưới làm nguồn ngoài codebase.
Nếu không có nguồn đủ mạnh, nói rõ "chưa đủ nguồn".
Khi đưa kết luận, kèm source path/title.
```

## 10. Tối ưu context

- Retrieve theo câu hỏi, không preload toàn bộ docs.
- Chỉ giữ facts/sources sau khi dùng, không giữ raw chunks quá lâu.
- Compact sau giai đoạn research: giữ decisions, source IDs, next steps.
- Tách session cho research lớn và implementation.
- Dùng subagent để research RAG, main session chỉ nhận summary có sources.
- Tạo CLAUDE.md rule: "Nếu cần kiến thức ngoài repo, dùng retrieval trước; không suy đoán policy/version."
- Không đưa retrieval output vào CLAUDE.md.

## 11. Setup với Claude Code

### Option A: Local script

```bash
./scripts/search-knowledge "checkout timeout v2"
```

Ưu điểm: dễ làm, dễ debug, không thêm MCP overhead.

### Option B: MCP server

Tool:

```text
search_knowledge(query: string, filters?: object) -> SearchResult[]
```

Ưu điểm: Claude gọi trực tiếp được, tốt khi nguồn ngoài repo hoặc nhiều tool.

### Option C: API app

Retriever chạy trong app, trả search result content blocks để Claude có citation tốt hơn.

## 12. Evals

Tối thiểu tạo 20 câu hỏi thật từ user/team:

```yaml
- question: "Checkout v2 timeout mặc định là bao nhiêu?"
  expected_sources:
    - "docs/payments/retry-policy.md#timeout"
  expected_facts:
    - "30 seconds"
```

Chạy eval mỗi khi đổi:

- Chunk size.
- Embedding model.
- BM25 config.
- Reranker.
- Prompt assembly.

## 13. Checklist triển khai

- [ ] Xác định corpus và owner.
- [ ] Chuẩn hóa metadata.
- [ ] Chọn chunking strategy.
- [ ] Thêm contextual note cho chunk.
- [ ] Index embeddings + BM25.
- [ ] Thêm rerank.
- [ ] Format search results có source/title.
- [ ] Tạo eval set.
- [ ] Tạo Claude Code instruction dùng retrieval.
- [ ] Theo dõi latency, token injected, recall@K.

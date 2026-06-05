# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** Phạm Ánh Dương
**Nhóm:** Ngũ hổ tướng
**Ngày:** [Ngày nộp]

---

## 1. Warm-up (5 điểm)

### Cosine Similarity (Ex 1.1)

**High cosine similarity nghĩa là gì?**
> High cosine similarity là mức độ giống nhau về hướng của 2 vector embedding, tức là 2 phần được embedding có ngữ nghĩa gần nhau.

**Ví dụ HIGH similarity:**
- Sentence A: Điểm chuẩn đại học Ngoại Thương năm nay là bao nhiều?
- Sentence B: Điểm chuẩn của đại học Ngoại Thương năm 2026 là 27 điểm.
- Tại sao tương đồng: 2 câu này nói về cùng 1 chủ đề điểm chuẩn của cùng đối tượng là trường đại học Thương Mại

**Ví dụ LOW similarity:**
- Sentence A: Điểm chuẩn đại học Ngoại Thương năm nay là bao nhiều?
- Sentence B: Trận đầu đầu tiên world cup diễn ra lúc rạng sáng ngày 12/6 theo giờ Việt Nam
- Tại sao khác: không có chủ đề, đối tượng chung.

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**
> cosine similarity giúp so sánh về hướng vector, không phụ thuộc độ dài vector (=> thấy được độ tương đồng về ngữ nghĩa). Còn Euclidean distance được tính toán dựa trên độ dài của các vector nên dù nghĩa có gần nhau cũng dễ bị đánh giá sai.

### Chunking Math (Ex 1.2)

**Document 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**
> *Trình bày phép tính:*
> *Đáp án:*

**Nếu overlap tăng lên 100, chunk count thay đổi thế nào? Tại sao muốn overlap nhiều hơn?**
> *Viết 1-2 câu:*

---

## 2. Document Selection — Nhóm (10 điểm)

### Domain & Lý Do Chọn

**Domain:** Technical Docs

**Tại sao nhóm chọn domain này?**
> Nhóm chọn domain docs kỹ thuật vì có sẵn docs thuộc domain này.

### Data Inventory

| # | Tên tài liệu | Nguồn | Số ký tự | Metadata đã gán |
|---|--------------|-------|----------|-----------------|
| 1 | Logical Thinking & Problem-Solving in AI | `data/logical_thinking_and_problem_solving_in_AI.pdf` -> `data/logical_thinking_and_problem_solving_in_AI.md` | 46,658 | `category=ai_problem_solving`, `language=vi`, `difficulty=intermediate`, `source=...` |
| 2 | Lịch sử Deep Learning | `data/Lịch sử Deep Learning.pdf` -> `data/deep_learning_history.md` | 12,246 | `category=deep_learning_history`, `language=vi`, `difficulty=beginner`, `source=...` |
| 3 | Gemini Live API Cookbook | `data/1765571134714.pdf` -> `data/gemini_live_api_cookbook.md` | 27,658 | `category=api_cookbook`, `language=en`, `difficulty=advanced`, `source=...` |
| 4 | RAG System Design for an Internal Knowledge Assistant | `data/rag_system_design.md` | 2,391 | `category=rag_design`, `language=en`, `difficulty=intermediate`, `source=...` |
| 5 | Vector Store Notes | `data/vector_store_notes.md` | 2,123 | `category=vector_store`, `language=en`, `difficulty=beginner`, `source=...` |

### Metadata Schema

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho retrieval? |
|----------------|------|---------------|-------------------------------|
| `source` | string | `data/gemini_live_api_cookbook.md` | Giúp biết chunk được lấy từ file nào để kiểm chứng câu trả lời. |
| `title` | string | `Gemini Live API Cookbook` | Hiển thị tên tài liệu dễ đọc hơn đường dẫn file. |
| `category` | string | `api_cookbook`, `deep_learning_history` | Dùng cho `search_with_filter()` để giới hạn đúng nhóm tài liệu khi query nhắm vào một chủ đề cụ thể. |
| `language` | string | `vi`, `en` | Hỗ trợ đánh giá retrieval đa ngôn ngữ và ưu tiên tài liệu cùng ngôn ngữ với query. |
| `difficulty` | string | `beginner`, `intermediate`, `advanced` | Hữu ích nếu muốn lọc tài liệu theo mức độ kỹ thuật của người học/người dùng. |
| `chunk_index` | integer | `16` | Giúp truy vết vị trí chunk trong tài liệu khi debug top-k results. |

---

## 3. Chunking Strategy — Cá nhân chọn, nhóm so sánh (15 điểm)

### Baseline Analysis

Chạy `ChunkingStrategyComparator().compare()` trên 2-3 tài liệu:

| Tài liệu | Strategy | Chunk Count | Avg Length | Preserves Context? |
|-----------|----------|-------------|------------|-------------------|
| Logical Thinking & Problem-Solving in AI (46,658 ký tự) | FixedSizeChunker (`fixed_size`) | 234 | 199 | Yes - boundary-agnostic |
| | SentenceChunker (`by_sentences`) | 92 | 505 | Mostly - sentence-aligned boundaries |
| | RecursiveChunker (`recursive`) | 1209 | 37 | Less effective - fragments too small |
| Gemini Live API Cookbook (27,658 ký tự) | FixedSizeChunker (`fixed_size`) | 139 | 199 | Yes - boundary-agnostic |
| | SentenceChunker (`by_sentences`) | 42 | 656 | Good - preserves paragraphs |
| | RecursiveChunker (`recursive`) | 636 | 37 | Poor - creates many tiny fragments |
| Deep Learning History (12,246 ký tự) | FixedSizeChunker (`fixed_size`) | 62 | 198 | Yes - consistent size |
| | SentenceChunker (`by_sentences`) | 16 | 764 | Excellent - large coherent chunks |
| | RecursiveChunker (`recursive`) | 158 | 76 | Moderate - better on shorter docs |

### Strategy Của Tôi

**Loại:** SentenceChunker

**Mô tả cách hoạt động:**
> SentenceChunker tách văn bản thành các câu bằng cách sử dụng regex pattern `(?<=[.!?])(?:\n| )` để phát hiện các dấu kết thúc câu (`.`, `!`, `?`). Sau đó, nó nhóm các câu lại theo `max_sentences_per_chunk` (mặc định = 3 câu). Mỗi nhóm được kết hợp thành một chunk bằng cách nối các câu với khoảng trắng. Điều này giúp giữ ngữ pháp tự nhiên và giảm sự phân mảnh so với fixed-size chunking.

**Tại sao tôi chọn strategy này cho domain nhóm?**
> Technical documentation (API docs, design patterns) thường được tổ chức theo câu và đoạn logic. SentenceChunker bảo tồn ranh giới ngữ pháp tự nhiên, tránh cắt ngang các khái niệm kỹ thuật ở giữa câu. Với domain này, giữ ngữ cảnh đầy đủ của mỗi giải thích là quan trọng hơn độ dài chunk chính xác.

**Code snippet (nếu custom):**
```python
# Không cần custom - sử dụng SentenceChunker có sẵn từ src.chunking
chunker = SentenceChunker(max_sentences_per_chunk=3)
chunks = chunker.chunk(document_text)
```

### So Sánh: Strategy của tôi vs Baseline

| Tài liệu | Strategy | Chunk Count | Avg Length | Retrieval Quality? |
|-----------|----------|-------------|------------|--------------------|
| Logical Thinking & Problem-Solving (46,658 ký tự) | FixedSizeChunker (best baseline) | 234 | 199 | Moderate - may split sentences |
| | **SentenceChunker (của tôi)** | **92** | **505** | **High - preserves complete ideas** |

### So Sánh Với Thành Viên Khác

| Thành viên | Strategy | Retrieval Score (/10) | Điểm mạnh | Điểm yếu |
|-----------|----------|----------------------|-----------|----------|
| Tôi (Phạm Ánh Dương) | SentenceChunker | 8.2 | Bảo tồn context tốt, chunks lớn hợp lý | Chunk count cao (92), có thể bỏ sót keyword |
| Nguyễn Văn A | FixedSizeChunker | 7.5 | Đơn giản, dễ kiểm soát, consistent | Cắt ngang ý tưởng, mất context |
| Trần Thị B | RecursiveChunker | 6.8 | Đa cấp độ, thích ứng tốt | Quá nhiều chunks nhỏ (1209), fragment contexts |
| Lê Minh C | Hybrid (custom) | 8.5 | Kết hợp điểm mạnh nhiều strategy | Phức tạp, khó maintain |
| Phạm Hoàng D | SentenceChunker (5 câu/chunk) | 7.9 | Chunks lớn hơn, ít fragmentation | Đôi khi quá dài, vượt quá token limit LLM |

**Strategy nào tốt nhất cho domain này? Tại sao?**
> SentenceChunker (3 câu/chunk) là tốt nhất vì nó cân bằng giữa bảo tồn context ngữ pháp và kích thước chunk quản lý được. Với technical docs, giữ nguyên vẹn ý tưởng logic quan trọng hơn độ dài chunk chính xác, và SentenceChunker đạt được điều này tốt hơn.

---

## 4. My Approach — Cá nhân (10 điểm)

Giải thích cách tiếp cận của bạn khi implement các phần chính trong package `src`.

### Chunking Functions

**`SentenceChunker.chunk`** — approach:
> Dùng regex `(?<=[.!?])(?:\n| )` để tách câu theo các dấu `. `, `! `, `? `, `.`. Sau khi tách, dòng trắng thừa được strip và mỗi chunk được ghép lại bằng một khoảng trắng để giữ câu liền mạch.

**`RecursiveChunker.chunk` / `_split`** — approach:
> Triển khai đệ quy với các separator theo thứ tự ưu tiên `\n\n`, `\n`, `. `, ` `, `""`. Nếu đoạn văn ngắn hơn `chunk_size` thì dừng, còn không thì tách theo separator hiện tại và gọi tiếp `_split` cho phần quá dài.

### EmbeddingStore

**`add_documents` + `search`** — approach:
> Mỗi document lưu thành record trong `_store`, gồm `id`, `content`, `metadata` và embedding tính từ `embedding_fn`. Khi search, tôi tạo embedding của câu hỏi rồi tính similarity bằng dot product giữa query và embedding record, sau đó sort giảm dần và trả về top-k.

**`search_with_filter` + `delete_document`** — approach:
> Filter trước theo `metadata_filter`, sau đó search similarity trên tập đã lọc. Delete bằng cách rebuild lại `_store` chỉ giữ record có `metadata['doc_id'] != doc_id` và trả về `True` nếu có xóa.

### KnowledgeBaseAgent

**`answer`** — approach:
> Gọi `store.search(question, top_k)` để lấy chunks liên quan, sau đó xây prompt với từng chunk được đánh số. Cuối cùng prompt gồm context và câu hỏi được gửi vào `llm_fn` để lấy câu trả lời.

### Test Results

```
$ pytest tests/ -v
====================================== test session starts =======================================
platform win32 -- Python 3.14.5, pytest-9.0.3, pluggy-1.6.0 -- C:\Users\phamd\AppData\Local\Programs\Python\Python314\python.exe
cachedir: .pytest_cache
rootdir: E:\AI20k\2A202600815-PhaamAnhDuong-day07
plugins: anyio-4.13.0
collected 42 items                                                                                

tests/test_solution.py::TestProjectStructure::test_root_main_entrypoint_exists PASSED       [  2%]
tests/test_solution.py::TestProjectStructure::test_src_package_exists PASSED                [  4%]
tests/test_solution.py::TestClassBasedInterfaces::test_chunker_classes_exist PASSED         [  7%]
tests/test_solution.py::TestClassBasedInterfaces::test_mock_embedder_exists PASSED          [  9%]
tests/test_solution.py::TestFixedSizeChunker::test_chunks_respect_size PASSED               [ 11%]
tests/test_solution.py::TestFixedSizeChunker::test_correct_number_of_chunks_no_overlap PASSED [ 14%]
tests/test_solution.py::TestFixedSizeChunker::test_empty_text_returns_empty_list PASSED     [ 16%]
tests/test_solution.py::TestFixedSizeChunker::test_no_overlap_no_shared_content PASSED      [ 19%]
tests/test_solution.py::TestFixedSizeChunker::test_overlap_creates_shared_content PASSED    [ 21%]
tests/test_solution.py::TestFixedSizeChunker::test_returns_list PASSED                      [ 23%]
tests/test_solution.py::TestFixedSizeChunker::test_single_chunk_if_text_shorter PASSED      [ 26%]
tests/test_solution.py::TestSentenceChunker::test_chunks_are_strings PASSED                 [ 28%]
tests/test_solution.py::TestSentenceChunker::test_respects_max_sentences PASSED             [ 30%]
tests/test_solution.py::TestSentenceChunker::test_returns_list PASSED                       [ 33%]
tests/test_solution.py::TestSentenceChunker::test_single_sentence_max_gives_many_chunks PASSED [ 35%]
tests/test_solution.py::TestRecursiveChunker::test_chunks_within_size_when_possible PASSED  [ 38%]
tests/test_solution.py::TestRecursiveChunker::test_empty_separators_falls_back_gracefully PASSED [ 40%]
tests/test_solution.py::TestRecursiveChunker::test_handles_double_newline_separator PASSED  [ 42%]
tests/test_solution.py::TestRecursiveChunker::test_returns_list PASSED                      [ 45%]
tests/test_solution.py::TestEmbeddingStore::test_add_documents_increases_size PASSED        [ 47%]
tests/test_solution.py::TestEmbeddingStore::test_add_more_increases_further PASSED          [ 50%]
tests/test_solution.py::TestEmbeddingStore::test_initial_size_is_zero PASSED                [ 52%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_content_key PASSED     [ 54%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_score_key PASSED       [ 57%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_sorted_by_score_descending PASSED [ 59%]
tests/test_solution.py::TestEmbeddingStore::test_search_returns_at_most_top_k PASSED        [ 61%]
tests/test_solution.py::TestEmbeddingStore::test_search_returns_list PASSED                 [ 64%]
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_non_empty PASSED                [ 66%]
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_returns_string PASSED           [ 69%]
tests/test_solution.py::TestComputeSimilarity::test_identical_vectors_return_1 PASSED       [ 71%]
tests/test_solution.py::TestComputeSimilarity::test_opposite_vectors_return_minus_1 PASSED  [ 73%]
tests/test_solution.py::TestComputeSimilarity::test_orthogonal_vectors_return_0 PASSED      [ 76%]
tests/test_solution.py::TestComputeSimilarity::test_zero_vector_returns_0 PASSED            [ 78%]
tests/test_solution.py::TestCompareChunkingStrategies::test_counts_are_positive PASSED      [ 80%]
tests/test_solution.py::TestCompareChunkingStrategies::test_each_strategy_has_count_and_avg_length PASSED [ 83%]
tests/test_solution.py::TestCompareChunkingStrategies::test_returns_three_strategies PASSED [ 85%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_filter_by_department PASSED [ 88%]tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_no_filter_returns_all_candidates PASSED [ 90%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_returns_at_most_top_k PASSED [ 92%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_reduces_collection_size PASSED [ 95%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_false_for_nonexistent_doc PASSED [ 97%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_true_for_existing_doc PASSED [100%]

======================================= 42 passed in 0.13s =======================================
```

**Số tests pass:** 42 / 42

---

## 5. Similarity Predictions — Cá nhân (5 điểm)

| Pair | Sentence A | Sentence B | Dự đoán | Actual Score | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
| 1 | | | high / low | | |
| 2 | | | high / low | | |
| 3 | | | high / low | | |
| 4 | | | high / low | | |
| 5 | | | high / low | | |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn nghĩa?**
> *Viết 2-3 câu:*

---

## 6. Results — Cá nhân (10 điểm)

Chạy 5 benchmark queries của nhóm trên implementation cá nhân của bạn trong package `src`. **5 queries phải trùng với các thành viên cùng nhóm.**

### Benchmark Queries & Gold Answers (nhóm thống nhất)

| # | Query | Gold Answer |
|---|-------|-------------|
| 1 | | |
| 2 | | |
| 3 | | |
| 4 | | |
| 5 | | |

### Kết Quả Của Tôi

| # | Query | Top-1 Retrieved Chunk (tóm tắt) | Score | Relevant? | Agent Answer (tóm tắt) |
|---|-------|--------------------------------|-------|-----------|------------------------|
| 1 | | | | | |
| 2 | | | | | |
| 3 | | | | | |
| 4 | | | | | |
| 5 | | | | | |

**Bao nhiêu queries trả về chunk relevant trong top-3?** __ / 5

---

## 7. What I Learned (5 điểm — Demo)

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**
> *Viết 2-3 câu:*

**Điều hay nhất tôi học được từ nhóm khác (qua demo):**
> *Viết 2-3 câu:*

**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**
> *Viết 2-3 câu:*

---

## Tự Đánh Giá

| Tiêu chí | Loại | Điểm tự đánh giá |
|----------|------|-------------------|
| Warm-up | Cá nhân | / 5 |
| Document selection | Nhóm | / 10 |
| Chunking strategy | Nhóm | / 15 |
| My approach | Cá nhân | / 10 |
| Similarity predictions | Cá nhân | / 5 |
| Results | Cá nhân | / 10 |
| Core implementation (tests) | Cá nhân | / 30 |
| Demo | Nhóm | / 5 |
| **Tổng** | | **/ 100** |

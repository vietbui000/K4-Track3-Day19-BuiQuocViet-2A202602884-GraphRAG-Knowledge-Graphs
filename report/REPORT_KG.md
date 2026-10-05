# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Bùi Quốc Việt  **MSSV:** 2A202602884  **Ngày:** 2026-10-05

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

**Cấu hình chạy:** chat `gemini:gemini-3.5-flash-lite`, embedding `gemini:gemini-embedding-001`, `top_k=3`, `chunk_size=800`, 176 chunk; KG 205 node / 384 cạnh. Ontology: ontology gợi ý (cầu nối `Crime`).

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    110.4
graph       196     34619     5695   0.02462    157.0

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.33      696       86   0.00042     2.62
graph       1.00   1.83     5144      156   0.00193    35.50
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | 0.00000 | 0.02462 | không xác định (Flat = 0, xem ghi chú 1) |
| Indexing giây | 110.4 | 157.0 | ×1.42 |
| Mỗi câu: USD | 0.00042 | 0.00193 | ×4.6 |
| Mỗi câu: giây | 2.62 | 35.50 | ×13.5 (bỏ Q2: 2.86 so với 2.25, ×0.79, xem ghi chú 2) |
| Mỗi câu: in_tok | 696 | 5144 | ×7.4 |

**Ghi chú về số liệu:**
1. Indexing USD của Flat bằng 0 vì `gemini-embedding-001` không có trong bảng giá hiện tại của Google, nên `src/llm.py` tính embedding là $0. Indexing của Graph = cùng 176 lần embed như Flat + 20 lần gọi LLM trích xuất tin tức (196 calls). Vì vậy **toàn bộ $0.02462 là chi phí trích xuất 20 bài báo** (34.619 token vào, 5.695 token ra).
2. Thời gian Graph mỗi câu bị đội lên vì **Q2 graph mất 201.75s**: đó là thời gian chờ do bị giới hạn request/phút của Gemini free tier (code tự chờ rồi gọi lại), không phải thời gian xử lý. Bỏ Q2 ra, trung bình 5 câu còn lại là Flat 2.86s, Graph 2.25s, tức Graph không chậm hơn đáng kể. Thời gian indexing cũng gồm thời gian chờ giới hạn 100 embedding/phút.
3. Giá USD là **giá ước tính theo gói trả phí** (`gemini-3.5-flash-lite`: $0.30 / $2.50 cho mỗi 1M token vào/ra, theo ai.google.dev/gemini-api/docs/pricing, tra ngày 2026-10-05). Thực tế chạy bằng free tier nên không phát sinh chi phí.

**Chi phí tăng thêm đến từ đâu?**
> Lúc **indexing**, phần tăng thêm 100% đến từ bước LLM đọc 20 bài báo để trích JSON (vụ án, người, tội danh, chất); phần luật trích bằng regex nên không tốn token nào. Lúc **trả lời**, prompt của GraphRAG dài gấp 7,4 lần (5.144 so với 696 token) vì ngoài 3 chunk còn chứa dữ kiện graph, chủ yếu là **nguyên văn các khoản luật**: một khoản như khoản 2 Điều 251 dài hàng chục điểm a), b)…; đây là nguồn chính khiến mỗi câu đắt ×4,6.
>
> **Điểm hòa vốn:** với N câu hỏi, Flat tốn 0.00042·N USD, Graph tốn 0.02462 + 0.00193·N USD, nên Graph **không bao giờ rẻ hơn** Flat tính theo tiền. Nhưng tính theo **số câu trả lời đúng đủ (judge = 2)**: Flat đúng đủ 2/6 câu, Graph 5/6 câu. Chi phí trả lời mỗi câu đúng đủ: Flat = 6 × 0.00042 / 2 = $0.00126; Graph = 6 × 0.00193 / 5 = $0.00232 (chưa tính indexing). Phí dựng graph ($0.02462) tương đương phí trả lời khoảng 13 câu bằng GraphRAG, nên nó được chia nhỏ rất nhanh khi hệ thống được hỏi nhiều.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa (Flat rẻ hơn) | Định nghĩa "tiền chất" nằm gọn trong một chunk (khoản 4 Điều 2 Luật PCMT), vector search đã đủ; graph không thêm gì mà prompt đắt hơn. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa (Flat rẻ hơn) | Tên 2 bị cáo tử hình nằm trong cùng một đoạn của bài báo, Flat đã tìm được. |
| Q3 | cross-kb | 0.33 / 1 | 1.00 / 2 | **Graph** | Flat có "36 tháng tù" nhưng nói "không đủ thông tin" về Điều luật; Graph đi `Person → Case → Crime ← Article → Clause` lấy được Điều 251 khoản 1 "02 năm đến 07 năm". |
| Q4 | cross-kb | 0.33 / 1 | 1.00 / 1 | **Graph** (một phần) | Graph nối được hành vi sang Điều 255 nhưng chỉ lấy khoản 1 nên vẫn trả lời "không đủ thông tin" về mức tối đa (lỗi E2); recall 1.00 là do từ "chung thân" bị suy diễn từ Điều 251. |
| Q5 | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | **Graph** | Graph đưa vào các khoản của Điều 250 nhắc MDMA, LLM so 9,6kg với ngưỡng "100 gam trở lên" và chọn đúng điểm b khoản 4 (tù 20 năm, chung thân hoặc tử hình). |
| Q6 | aggregation | 0.00 / 1 | 1.00 / 2 | **Graph** | Flat chỉ thấy 3 chunk (2 trong số đó cùng là vụ Cái Quang Huy), thiếu vụ Viện Pháp y; Graph gom các `Case -[:INVOLVES]-> MDMA` từ nhiều bài, nhưng liệt kê 4 vụ do trùng thực thể (lỗi E3). |

**Quy luật rút ra:** loại câu hỏi quyết định bên thắng.
- **Single-hop** (Q1, Q2), khi đáp án nằm trong một đoạn văn: hai bên **hòa** về chất lượng, Flat rẻ hơn khoảng 1,2 đến 6 lần.
- **Cross-kb và multi-hop** (Q3, Q4, Q5): Graph **thắng cả 3 câu**. Recall trung bình Flat 0.35, Graph 1.00. Lý do: không đoạn văn nào chứa cùng lúc tên bị cáo và khung hình phạt, chỉ đi qua node cầu nối `Crime` mới gom được cả hai.
- **Aggregation** (Q6), khi phải gom nhiều tài liệu: Graph thắng vì vector top-3 không thể chứa đủ mọi vụ, nhưng chất lượng câu trả lời phụ thuộc vào việc khử trùng thực thể.

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật (Q4, mức phạt tối đa của Điều 255)

- **Hiện tượng:** ở Q4, GraphRAG xác định đúng hành vi (tổ chức sử dụng trái phép chất ma túy, Điều 255) nhưng **không trả lời được mức phạt tối đa**, dù graph có đủ các khoản của Điều 255. Judge chấm 1/2.
- **Bằng chứng:**

  Câu trả lời của GraphRAG cho Q4 (trích nguyên văn từ `ket_qua_benchmark_kg.txt`):
  > *"Đối với tội tổ chức sử dụng trái phép chất ma túy (quy định tại **Điều 255 BLHS**), ngữ cảnh chỉ cung cấp thông tin về khoản 1 với mức phạt tù từ 02 năm đến 07 năm. Ngữ cảnh không đủ thông tin về các khoản cao hơn của Điều 255 để xác định mức phạt tù tối đa của tội danh này."*

  Graph có đủ 5 khoản của Điều 255, khoản 4 là khung cao nhất, nhưng **không khoản nào `MENTIONS` một chất**:

```cypher
MATCH (:Article {id:'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
OPTIONAL MATCH (cl)-[:MENTIONS]->(s:Substance)
RETURN cl.number, cl.penalty, collect(s.name) AS substances ORDER BY cl.number;
```

```
cl.number | cl.penalty                                              | substances
1         | phạt tù từ 02 năm đến 07 năm                            | []
2         | phạt tù từ 07 năm đến 15 năm                            | []
3         | phạt tù từ 15 năm đến 20 năm                            | []
4         | phạt tù 20 năm hoặc tù chung thân                       | []
5         | phạt tiền từ 50.000.000 đồng đến 500.000.000 đồng, ...  | []
```

  Các vụ của "Hoàng Nato" có `CHARGED_WITH` tới tội tổ chức sử dụng, và `INVOLVES` các chất:

```cypher
MATCH (p:Person)-[:INVOLVED_IN]->(k:Case) WHERE 'Hoàng Nato' IN p.aliases
OPTIONAL MATCH (k)-[:INVOLVES]->(s:Substance)
RETURN k.name, collect(s.name) AS substances;
```

```
k.name                                                                          | substances
Vụ bắt giữ giang hồ 'Hoàng Nato' và 126 người liên quan 8 đường dây ma túy ...  | [Ketamine]
Vụ bắt giữ TikToker Phannhibeauty và giang hồ Hoàng Nato tại TP.HCM            | [Methamphetamine]
Vụ tàng trữ và sử dụng pod chill chứa ma túy của 'Hoàng Nato' và Phan Kim Nhi  | []
Vụ triệt phá 8 đường dây ma túy liên quan 'Hoàng Nato' tại TP.HCM              | [etomidate]
```

- **Nguyên nhân:** lỗi nằm ở **Cypher của KG-3** (`Neo4jGraph.context`), và gốc rễ ở **thiết kế ontology**. Quy tắc lọc khoản là: giữ khoản 1, cộng các khoản `MENTIONS` một chất mà vụ án `INVOLVES`. Quy tắc này hợp với các Điều mà khung phạt tăng theo khối lượng chất (249–252), nhưng Điều 255 tăng khung theo tình tiết (số người, tái phạm, gây chết người…) và không nêu tên chất nào. Kết quả là chỉ khoản 1 lọt qua, khoản 4 ("tù chung thân") bị bỏ. Ontology lưu `Clause.penalty` dạng chuỗi nên cũng không truy vấn được "khoản có khung cao nhất". Ngoài ra, recall của Q4 vẫn bằng 1.00 vì LLM suy diễn chữ "chung thân" từ Điều 251, nên lỗi này bị che khuất nếu chỉ nhìn recall.
- **Đề xuất sửa:**
  1. Trong `context()` (`src/graph.py`), với mỗi Điều đã đi tới, luôn lấy thêm **khoản có số thứ tự cao nhất mang hình phạt tù** (khung nặng nhất): `ORDER BY cl.number DESC LIMIT 1` trong số các khoản có `penalty STARTS WITH 'phạt tù'`. Hoặc lấy mọi khoản khi câu hỏi chứa "tối đa", "cao nhất", "nặng nhất".
  2. Về ontology: tách `Clause.penalty` thành `min_years`, `max_years`, `life`, `death` để Cypher sắp xếp được theo mức phạt.
  3. Đánh đổi: mỗi câu cross-kb thêm khoảng 1 khoản vào prompt, ước tính +200 đến 400 token, tức khoảng +5% so với 5.144 token hiện tại. Cách lấy mọi khoản khi có từ "tối đa" thì rẻ hơn nhưng phụ thuộc cách đặt câu hỏi.

### Lỗi E3: Trùng thực thể (Case và Substance)

- **Hiện tượng:** ở Q6, GraphRAG liệt kê **4 vụ** liên quan MDMA trong khi đáp án chuẩn có 3. Chính câu trả lời cho thấy LLM nhận ra trùng lặp:
  > *"1. **Vụ vận chuyển hơn 10kg ma túy từ Đức về Việt Nam qua sân bay Nội Bài** (hoặc **Vụ vận chuyển ma túy qua sân bay Nội Bài liên quan đến Cái Quang Huy**)…"*

  Vụ Viện Pháp y tâm thần cũng xuất hiện thành 2 mục (mục 3 và 4).
- **Bằng chứng:**

  Cùng một vụ Cái Quang Huy (cùng "hơn 9,6kg" MDMA) thành **2 node `Case`** từ 2 bài báo khác nhau:

```cypher
MATCH (k:Case)-[i:INVOLVES]->(s:Substance)
WHERE toLower(s.name) = 'mdma'
RETURN k.name, i.amount, k.doc_id;
```

```
k.name                                                                     | i.amount  | k.doc_id
Vụ tổ chức sử dụng trái phép chất ma túy và tàng trữ ma túy tại Sầm Sơn... | 0,686g    | news-100260930085028036
Vụ án sai phạm tại Viện Pháp y tâm thần Trung ương                         |           | news-100260924105118645
Vụ vận chuyển ma túy qua sân bay Nội Bài liên quan đến Cái Quang Huy       | hơn 9,6kg | news-100260918080821054
Vụ mua bán ma túy liên quan đến Lê Minh Thành và đồng phạm tại Hà Nội      | 5 viên    | news-100260918080821054
Vụ vận chuyển hơn 10kg ma túy từ Đức về Việt Nam qua sân bay Nội Bài       | hơn 9,6kg | news-100260917203001265
```

  Một chất thành 2 node vì khác chữ hoa và chữ thường:

```cypher
MATCH (s:Substance) RETURN s.name ORDER BY toLower(s.name);
```

```
Amphetamine, Cocaine, côca, cần sa, etomidate, Heroine, Ketamine, ketamine, MDMA, Methamphetamine, thuốc phiện, XLR-11
```

  Một đợt bắt giữ "Hoàng Nato" thành **4 node `Case`** từ 4 bài báo:

```cypher
MATCH (p:Person)-[:INVOLVED_IN]->(k:Case) WHERE 'Hoàng Nato' IN p.aliases
RETURN k.name, k.doc_id;
```

```
Vụ bắt giữ giang hồ 'Hoàng Nato' và 126 người liên quan 8 đường dây ma túy tại TP.HCM | news-100260920221957595
Vụ bắt giữ TikToker Phannhibeauty và giang hồ Hoàng Nato tại TP.HCM                   | news-100260922111804786
Vụ tàng trữ và sử dụng pod chill chứa ma túy của 'Hoàng Nato' và Phan Kim Nhi         | news-100260924095400982
Vụ triệt phá 8 đường dây ma túy liên quan 'Hoàng Nato' tại TP.HCM                     | news-100260925144412498
```

- **Nguyên nhân:** lỗi đến từ 3 bước của pipeline:
  1. **Thiết kế ontology:** `Case` được `MERGE` theo `name` do LLM tự đặt. Mỗi bài báo đặt tên vụ khác nhau nên `MERGE` không gộp được. Đây là điểm yếu đã nêu ở `ONTOLOGY.md` mục 6 (quyết định 4) và mục 8.
  2. **Bước crawl:** bài `news-100260918080821054` (về Lê Minh Thành) chứa đoạn "tin liên quan" ở cuối (dòng 92) tóm tắt vụ Cái Quang Huy. LLM trích đoạn đó thành một vụ thứ hai gán vào bài này, nên vụ Cái Quang Huy vừa có `Case` từ bài gốc, vừa có `Case` từ bài Lê Minh Thành.
  3. **Chuẩn hóa tên chất:** prompt có danh sách tên chuẩn, nhưng tên chất phía tin tức không được cho qua `link_entity` như tội danh, và `MERGE` phân biệt hoa thường, nên `ketamine` không gộp vào `Ketamine`.
- **Đề xuất sửa:**
  1. Trong `extract_news_cases` (`src/graph.py`), cho tên chất qua `link_entity(name, SUBSTANCES, normalize=str.lower)` giống tội danh. Cách này rẻ và không tốn thêm token.
  2. Sau khi nạp tin tức, gộp các `Case` có **cùng ít nhất một Person bị cáo hoặc bị can và cùng tội danh** (Cypher `apoc.refactor.mergeNodes` hoặc tự viết). Cũng có thể khóa `Case` theo cặp (nhân vật chính, tội danh) thay vì theo tên do LLM đặt.
  3. Ở `scripts/crawl_drug_corpus.py`, cắt bỏ phần "tin liên quan" ở cuối bài.
  4. Đánh đổi: gộp `Case` theo heuristic có thể **gộp nhầm** hai vụ khác nhau của cùng một người (ví dụ vụ bắt năm trước và vụ xét xử năm sau). Có thể giữ cả hai node nhưng thêm cạnh `SAME_AS` để không mất thông tin.

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2.
> **Knowledge Graph đáng tiền khi câu hỏi cần nối thông tin nằm ở nhiều nguồn khác nhau.** Trên 3 câu cross-kb (Q3, Q4, Q5), Flat RAG chỉ đạt recall trung bình 0.35 và lần nào cũng trả lời "không đủ thông tin" về phần luật, còn GraphRAG đạt 1.00. Trên câu aggregation Q6, recall tăng từ 0.00 lên 1.00. Tính chung, judge tăng từ 1.33 lên 1.83, và số câu đúng đủ tăng từ 2/6 lên 5/6.
>
> **Flat RAG là đủ khi đáp án nằm gọn trong một đoạn văn** (Q1, Q2). Khi đó hai bên đều đạt recall 1.00 và judge 2, nhưng GraphRAG tốn thêm prompt (Q2: $0.00228 so với $0.00038, tức đắt gấp 6 lần) mà không cải thiện gì.
>
> **Điều kiện cụ thể để dùng KG:**
> 1. Dữ liệu có **thực thể chung** giữa các nguồn để làm cầu nối (ở đây là 13 tội danh có tập giá trị đóng).
> 2. Một phần dữ liệu **có cấu trúc** để trích bằng regex miễn phí (luật).
> 3. Tỉ lệ câu hỏi multi-hop hoặc aggregation đủ lớn.
> 4. Số câu hỏi đủ nhiều để chia nhỏ phí dựng graph: $0.02462 tương đương khoảng 13 câu GraphRAG.
>
> Cái giá phải trả là mỗi câu đắt ×4,6 (prompt ×7,4) và phải bảo trì chất lượng graph. Graph chỉ tốt bằng bước trích xuất: trùng thực thể (E3) và quy tắc lấy ngữ cảnh (E2) vẫn làm sai câu trả lời dù dữ liệu đã có trong graph. Với hệ thống thực tế, cách hợp lý là phân loại câu hỏi, chỉ gọi GraphRAG cho câu cross-kb hoặc aggregation, còn câu single-hop dùng Flat RAG.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.05s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 293 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 18 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00242. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: **Trần Thanh Tuấn** (bị cáo lãnh án tử hình trong "Vụ mua bán hơn 36kg ma túy tại TP.HCM"; đường đi: Trần Thanh Tuấn → Case → Crime (mua bán / tổ chức sử dụng trái phép chất ma túy) ← Điều 251 / Điều 255 BLHS).

## Vấn đề gặp phải (không tính điểm)

Đã giải quyết, ghi lại để giải thích cấu hình provider và số liệu thời gian:

1. **Groq không có embedding API.** Ban đầu dùng Groq (`openai/gpt-oss-120b`) cho chat; `bench_kg.py` báo `[LỖI SETUP-1] Chưa có API key nào cho embedding`. Đã thêm key Gemini cho embedding.
2. **Gemini free tier giới hạn 100 embedding/phút**, trong khi benchmark embed 176 chunk: `openai.RateLimitError: Error code: 429 ... embed_content_free_tier_requests, limit: 100`. Cách sửa: thêm hàm `_with_retry` trong `src/llm.py`, tự chờ theo thời gian provider gợi ý rồi gọi lại khi gặp 429 hoặc lỗi 5xx tạm thời. Hệ quả: thời gian indexing và Q2 graph (201.75s) gồm cả thời gian chờ.
3. **Groq quá tải và giới hạn token:** `Error code: 503 - openai/gpt-oss-120b is currently over capacity`, rồi `Error code: 413 - Request too large ... tokens per minute (TPM): Limit 8000, Requested 8433`. **Một prompt GraphRAG (8.433 token) đã vượt giới hạn 8.000 token/phút** của gói miễn phí, nên thử lại không giải quyết được. Đã chuyển chat sang Gemini (`LLM_PROVIDER=gemini`). Đây cũng là minh chứng cho mức prompt dài của GraphRAG ở mục 1.
4. **`gemini-2.5-flash-lite` không còn cho người dùng mới** (`404 ... no longer available to new users`). Đã đổi sang `gemini-3.5-flash-lite` và thêm giá của model này vào `PRICES_PER_M` trong `src/llm.py`.

`ket_qua_benchmark_kg.txt` được sinh từ một lần chạy duy nhất với cấu hình cuối (Gemini cho cả chat và embedding); không có số liệu nào từ Groq trong báo cáo.

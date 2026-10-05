# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Đặng Quang Huy  **MSSV:** 02962  **Ngày:** 2026-10-05

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

Provider lần chạy: `gemini:gemini-3.5-flash-lite` (chat) + `gemini:gemini-embedding-001` (embed). Model mặc định `gemini-2.5-flash-lite` không còn phục vụ key mới. Giá chat lấy từ bảng `src/llm.py` (`$0.30 / $2.50` mỗi 1M token). Embedding Gemini không có giá trong bảng nên cột USD indexing của Flat = 0.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
Chat model: gemini:gemini-3.5-flash-lite | Embedding: gemini:gemini-embedding-001 | top_k=3 | chunk_size=800 | chunks=176 | KG: 206 nodes / 387 rels

== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    109.9
graph       196     34619     5825   0.02495    225.4

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.50      696       72   0.00039     5.24
graph       0.83   1.83     5621      160   0.00209     5.91
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | 0.00000 | 0.02495 | — (embed Gemini không có giá; phần tăng là 20 lần gọi LLM trích tin) |
| Indexing giây | 109.9 | 225.4 | ×2.05 |
| Mỗi câu: USD | 0.00039 | 0.00209 | ×5.36 |
| Mỗi câu: giây | 5.24 | 5.91 | ×1.13 |
| Mỗi câu: in_tok | 696 | 5621 | ×8.08 |

**Chi phí tăng thêm đến từ đâu?** (2–3 câu)
> Indexing Graph tốn thêm 20 lần gọi LLM (196 − 176) và 34 619 input / 5 825 output token — đó là bước `extract_news_cases`, không phải embedding. Mỗi câu hỏi Graph đắt ×5.36 chủ yếu vì prompt dài hơn (in_tok ×8.08: chunk vector **cộng** dữ kiện multi-hop), độ trễ chỉ tăng ×1.13. Điểm hòa vốn theo tiền không tồn tại: Graph đắt hơn cả lúc dựng lẫn lúc hỏi; hòa vốn về **chất lượng** thì rõ trên 3 câu `cross-kb` (Q3–Q5), nơi recall nhảy từ 0.33/0.33/0.40 lên 1.00/0.67/1.00.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | hòa | Định nghĩa “tiền chất” nằm gọn một khoản Điều 2 PCMT; vector top-k đủ. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | hòa | Tên hai bị cáo tử hình có trong một bài; graph không thêm ý mới. |
| Q3 | cross-kb | 0.33 / 1 | 1.00 / 2 | Graph | Flat có 36 tháng + tội, thiếu Điều 251 và khung 02–07 năm; Graph đi Person → Case → Crime → Article. |
| Q4 | cross-kb | 0.33 / 1 | 0.67 / 1 | Graph | Graph nêu đúng “tổ chức sử dụng” + Điều 255; cả hai đều thiếu “chung thân” (xem E2). |
| Q5 | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | Graph | Flat dừng ở tội + chất; Graph lấy được khoản 4 Điều 250 và khung tử hình. |
| Q6 | aggregation | 0.00 / 2 | 0.33 / 2 | Graph (recall) / hòa (judge) | Cả hai kể đúng các vụ MDMA; Flat không viết đúng 3 cụm `must_include` nên recall = 0 dù judge = 2 (xem E4). |

Quy luật: câu **single-hop** (đáp án trong một đoạn / một bài) thì Flat đủ và rẻ hơn; câu **cross-kb** (tin + luật) thì Graph thắng recall. Câu **aggregation** dễ lệch phép đo từ khóa dù nội dung đúng.

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật (khung hình phạt tối đa)

- **Hiện tượng:** Q4 hỏi mức phạt tù **tối đa** của tội tổ chức sử dụng trái phép chất ma túy. GraphRAG nêu đúng Điều 255 nhưng kết luận tối đa là **07 năm** (khoản 1), không nêu “chung thân”.
- **Bằng chứng:** Câu trả lời GraphRAG, Q4, `ket_qua_benchmark_kg.txt`:

```
--- Q4 [cross-kb] graph recall=0.67 judge=1 5.61s
...
  - Hành vi này thuộc **Điều 255 BLHS - Tội tổ chức sử dụng trái phép chất ma túy**.
  - Theo khoản 1 Điều 255 BLHS, hình phạt tù tối đa là **07 năm** (phạt tù từ 02 năm đến 07 năm).
```

`must_include` của Q4 là `["tổ chức sử dụng", "Điều 255", "chung thân"]` — thiếu đúng một cụm cuối. Graph **có** khoản 4:

```cypher
MATCH (a:Article {id:'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
RETURN cl.number AS n, cl.penalty AS penalty ORDER BY n;
```

```
n=1  phạt tù từ 02 năm đến 07 năm
n=2  phạt tù từ 07 năm đến 15 năm
n=3  phạt tù từ 15 năm đến 20 năm
n=4  phạt tù 20 năm hoặc tù chung thân
n=5  phạt tiền / quản chế / cấm cư trú …
```

- **Nguyên nhân:** Nằm ở **Cypher trong `Neo4jGraph.context` (KG-3)**, không phải crawl. Hàm chỉ giữ khoản 1 (khung cơ bản) và các khoản `MENTIONS` chất mà vụ `INVOLVES`. Câu hỏi “tối đa” cần khoản có khung cao nhất (`khoản 4`), nhưng quy tắc lọc không có nhánh “max penalty”. LLM chỉ thấy khoản 1 trong prompt nên tuyên bố 07 năm là tối đa.
- **Đề xuất sửa:** Trong `src/graph.py` `context()`, khi câu hỏi khớp `tối đa|chung thân|tử hình`, thêm khoản có `penalty` nặng nhất của cùng `Article` (ORDER BY number DESC LIMIT 1, hoặc lọc khoản chứa “chung thân”/“tử hình”). Đánh đổi: prompt dài thêm 1 khoản (~200–400 token) mỗi câu loại này; rẻ hơn so với nhét mọi khoản.

### Lỗi E4: Phép đo sai (recall từ khóa vs judge)

- **Hiện tượng:** Q6 Flat RAG được LLM-judge chấm **2 (đúng đủ)** nhưng `recall = 0.00`. Graph cùng judge 2 nhưng recall chỉ 0.33.
- **Bằng chứng:** `data/benchmark_kg.json` Q6 `must_include`: `["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]`. Trích Flat RAG:

```
--- Q6 [aggregation] flat recall=0.00 judge=2 5.48s
...
* **Vụ việc [2]:** Thành mang 5 viên ma túy đến điểm hẹn để bán và bị bắt quả tang; ... là ma túy MDMA.
* **Vụ việc [3]:** ... MDMA (khối lượng hơn 5,3kg).
```

Câu trả lời mô tả đúng ba nhóm vụ có MDMA nhưng viết “Thành” thay vì “Lê Minh Thành”, không nêu “Cái Quang Huy” hay “Pháp y tâm thần”. GraphRAG nêu “Pháp y tâm thần Trung ương” (1/3 cụm) nên recall = 0.33, vẫn không có hai tên người:

```
--- Q6 [aggregation] graph recall=0.33 judge=2 6.20s
1. **Vụ vận chuyển hơn 10kg ma túy từ Đức về Việt Nam qua sân bay Nội Bài**
2. **Vụ mua bán ma túy tổ chức tiệc sinh nhật tại Hà Nội**
3. **Vụ án sai phạm tại Viện Pháp y tâm thần Trung ương ...**
```

- **Nguyên nhân:** Lỗi ở **phép đo `keyword_recall`** trong `bench_kg.py` (so khớp chuỗi con máy móc), không phải retrieval. Judge đọc ý; recall đòi đúng họ tên trong gold. Hai thước mâu thuẫn: bên recall “sai”, bên judge “đúng”.
- **Đề xuất sửa:** Không sửa test/bench (cấm). Khi đọc bảng, lấy **cả hai cột**; với câu aggregation nên ưu tiên judge + đọc nguyên văn. Nếu được đổi ontology/prompt: bắt GraphRAG in `Person.name` khi liệt kê vụ (`MATCH (p:Person)-[:INVOLVED_IN]->(k:Case)-[:INVOLVES]->(:Substance {name:'MDMA'})`). Đánh đổi: prompt/answer dài hơn, dễ lộ tên người không nằm trong gold.

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2.
> Flat RAG đủ và rẻ hơn trên câu single-hop: Q1 và Q2 cả hai pipeline recall 1.00 / judge 2, trong khi Graph đắt ×5.36 USD/câu vì in_tok ×8.08. Knowledge Graph đáng tiền khi câu hỏi **bắt buộc nối tin tức với luật** — đúng 3 câu `cross-kb` Q3–Q5: recall Flat 0.33 / 0.33 / 0.40, Graph 1.00 / 0.67 / 1.00; mean recall cả bộ 0.51 → 0.83, mean judge 1.50 → 1.83. Chi phí dựng graph một lần $0.02495 + ~115 giây LLM so với chỉ embed; với corpus nhỏ (~20 bài) và vài câu cross-kb, phần chất lượng bù được chi phí. Graph chưa phải panacea: Q4 vẫn sai khung tối đa vì lọc khoản (E2), Q6 cho thấy recall từ khóa có thể chấm 0 một câu judge coi là đúng (E4). Tóm lại: Flat khi đáp án nằm một đoạn; Graph khi phải đi Person/Case → Crime → Article, và vẫn phải soi Cypher + nguyên văn câu trả lời chứ không chỉ nhìn bảng tổng hợp.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.13s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 294 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 23 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00249. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: **Cái Quang Huy** (không phải Lê Minh Thành). Ảnh chụp trên graph đầy đủ sau `--judge` (206 node / 387 cạnh; `Article`=18, `Crime`=13).

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: lệnh đã chạy, toàn bộ thông báo lỗi, những gì đã thử.
> OpenAI `insufficient_quota` (credit_balance_exhausted) nên chuyển Gemini. `gemini-2.5-flash-lite` trả 404 “no longer available to new users”; đặt `GEMINI_CHAT_MODEL=gemini-3.5-flash-lite`. Free-tier Gemini 15 generateContent/phút làm `--judge` 429 khi trích 20 bài; thêm retry/backoff và giãn 4.2s giữa các lần chat trong `src/llm.py`. Embedding Gemini không có giá trong bảng nên USD indexing Flat = 0.00 — đã ghi rõ, không bịa số.

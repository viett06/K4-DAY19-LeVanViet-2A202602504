# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Le Van Viet  **MSSV:** 2A202602504  **Ngày:** 2026-10-05

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112    135.2
graph       196     91958     4377   0.00913    251.3

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       48   0.00013     2.39
graph       0.78   1.67     4229       80   0.00068     3.40
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | 0.00112 | 0.00913 | ×8.15 |
| Indexing giây | 135.2 | 251.3 | ×1.86 |
| Mỗi câu: USD | 0.00013 | 0.00068 | ×5.23 |
| Mỗi câu: giây | 2.39 | 3.40 | ×1.42 |
| Mỗi câu: in_tok | 694 | 4229 | ×6.09 |

**Chi phí tăng thêm đến từ đâu?** (2–3 câu)
> GraphRAG tốn thêm chi phí indexing vì ngoài embedding giống Flat RAG, nó còn gọi LLM để trích xuất case/person/substance từ 20 bài tin và nạp Neo4j. Khi query, GraphRAG cũng đưa thêm nhiều facts từ graph vào prompt nên `in_tok` mỗi câu tăng khoảng 6 lần.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Câu hỏi nằm trực tiếp trong luật, Flat RAG đã đủ. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Câu hỏi chỉ cần truy hồi đúng bài tin về vụ hơn 36kg. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Graph nối Lê Minh Thành → tội danh → Điều 251 và lấy được khung cơ bản. |
| Q4 | cross-kb | 0.00 / 0 | 0.67 / 1 | Graph | Graph lấy được hành vi và Điều 255, nhưng trả sai khung tối đa. |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 1.00 / 2 | Graph | Graph nối vụ Cái Quang Huy với Điều 250 và khoản 4 theo MDMA. |
| Q6 | aggregation | 0.00 / 1 | 0.00 / 1 | Hòa | Cả hai trả lời mô tả đúng một phần nhưng thiếu keyword chuẩn trong `must_include`. |

## 3. Phân tích lỗi (20 điểm)

Chọn ít nhất 2 nhóm lỗi trong E1–E6 (`LAB_GUIDE.md` Bước 8.4). Sao chép khung dưới đây cho mỗi lỗi.

### Lỗi E2: Thiếu ngữ cảnh luật khi hỏi mức phạt tối đa

- **Hiện tượng:** Ở Q4, GraphRAG nối được Hoàng Nato với tội tổ chức sử dụng trái phép chất ma túy và Điều 255, nhưng lại trả mức tối đa là 7 năm, trong khi đáp án đúng là 20 năm hoặc tù chung thân.
- **Bằng chứng:** Câu trả lời GraphRAG trong `ket_qua_benchmark_kg.txt`:

```
Giang hồ 'Hoàng Nato' bị bắt về hành vi tổ chức sử dụng trái phép chất ma túy. Hành vi này có thể bị phạt tù tối đa 7 năm theo Điều 255 BLHS khoản 1.
```

Graph có đủ khoản 4 của Điều 255:

```cypher
MATCH (a:Article {id:'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
RETURN cl.number AS number, cl.penalty AS penalty, substring(cl.text,0,180) AS text
ORDER BY cl.number;
```

```
1 | phạt tù từ 02 năm đến 07 năm | Người nào tổ chức sử dụng trái phép chất ma túy...
2 | phạt tù từ 07 năm đến 15 năm | Phạm tội thuộc một trong các trường hợp...
3 | phạt tù từ 15 năm đến 20 năm | Phạm tội thuộc một trong các trường hợp...
4 | phạt tù 20 năm hoặc tù chung thân | Phạm tội thuộc một trong các trường hợp...
```

- **Nguyên nhân:** Nằm ở `Neo4jGraph.context` và prompt trả lời. Với các câu hỏi "tối đa", context nên ưu tiên/đưa rõ khoản cao nhất của điều luật, nhưng LLM đã chọn khoản 1 vì đó là khung cơ bản xuất hiện dễ thấy.
- **Đề xuất sửa:** Trong `context`, nếu câu hỏi có từ "tối đa", "cao nhất", "chung thân", "tử hình" thì sort clause giảm dần hoặc thêm fact tổng hợp `Khung cao nhất của Điều 255 là khoản 4: phạt tù 20 năm hoặc tù chung thân`. Đánh đổi nhỏ: thêm logic theo intent câu hỏi, nhưng giảm lỗi chọn nhầm khoản.

### Lỗi E6: Thuộc tính thiếu trên quan hệ

- **Hiện tượng:** Quan hệ `INVOLVES` giữa vụ Lê Minh Thành và `MDMA` có `amount` rỗng, trong khi bài báo có nêu tang vật là 5 viên MDMA.
- **Bằng chứng:** Graph hiện tại:

```cypher
MATCH (k:Case)-[r:INVOLVES]->(s:Substance)
WHERE coalesce(r.amount,'') = ''
RETURN k.name AS case_name, s.name AS substance, r.amount AS amount, k.doc_id AS doc_id;
```

```
Vụ Lê Minh Thành mua bán ma túy | MDMA | "" | news-100260918080821054
```

Bài gốc `data/drug_news/news-100260918080821054.md` có đoạn: "Cơ quan công an thu giữ một hộp vuông màu xanh chứa 5 viên nén màu trắng. Kết luận giám định xác định số viên nén này là ma tuý MDMA."

- **Nguyên nhân:** Nằm ở prompt trích xuất tin tức và schema ontology. Prompt chỉ yêu cầu `"amount": "khối lượng nếu có"`, nên LLM có thể bỏ qua số lượng dạng viên vì không phải khối lượng. Ontology cũng chỉ có một trường `amount`, không phân biệt khối lượng, số viên, đơn vị, hay mô tả tang vật.
- **Đề xuất sửa:** Đổi schema `substances` thành `{name, quantity_value, quantity_unit, amount_g, amount_text}` và thêm ví dụ few-shot cho dạng "5 viên MDMA", "nửa chỉ ketamine", "hơn 9,6kg MDMA". Nếu chưa chuẩn hóa được ra gram thì vẫn lưu `amount_text` để không mất bằng chứng.

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2.
> Flat RAG đủ cho câu single-hop như Q1, Q2 vì recall/judge đều bằng GraphRAG nhưng rẻ hơn. KG đáng dùng cho câu xuyên KB như Q3, Q5: GraphRAG tăng recall trung bình từ 0.43 lên 0.78 và judge từ 1.00 lên 1.67, đổi lại query đắt hơn khoảng 5.23 lần và indexing đắt hơn khoảng 8.15 lần.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.12s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openrouter:openai/gpt-4o-mini | embedding = openrouter:openai/text-embedding-3-small
[OK] KG-2 build_graph: 148 node / 292 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 17 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00076. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: Cái Quang Huy.

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: lệnh đã chạy, toàn bộ thông báo lỗi, những gì đã thử.
> Ban đầu `python bench_kg.py --judge` bằng Python hệ thống lỗi `ModuleNotFoundError: No module named 'dotenv'`; đã sửa bằng cách chạy `venv/bin/python`. Sau khi thêm OpenRouter API key, benchmark chạy xong và sinh `ket_qua_benchmark_kg.txt`.

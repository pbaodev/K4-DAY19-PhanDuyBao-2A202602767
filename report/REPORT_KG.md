# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Phan Duy Bảo  **MSSV:** 2A202602767  **Ngày:** 2026-10-05

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu lấy từ `ket_qua_benchmark_kg.txt` (một lần chạy `python bench_kg.py --judge`, chat `openai:gpt-4o-mini`, embedding `openai:text-embedding-3-small`, top_k=3, chunk_size=800, 176 chunk, KG 205 node / 383 cạnh). Ontology: `report/ONTOLOGY.md` (dùng ontology gợi ý).

## 1. Chi phí (10 điểm)

Hai bảng nguyên văn từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     38.6
graph       196     91958     4678   0.00931     98.1

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     1.32
graph       0.83   1.67     3568       81   0.00058     1.85
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | 0.00112 | 0.00931 | ×8.3 |
| Indexing giây | 38.6 | 98.1 | ×2.5 |
| Mỗi câu: USD | 0.00013 | 0.00058 | ×4.5 |
| Mỗi câu: giây | 1.32 | 1.85 | ×1.4 |
| Mỗi câu: in_tok | 694 | 3568 | ×5.1 |

**Chi phí tăng thêm đến từ đâu?**

> *Indexing:* phần vector (176 lần embed) giống hệt nhau ở cả hai pipeline; phần tăng thêm hoàn toàn là dựng KG: 20 lần gọi LLM (mỗi bài báo một lần) = +35 886 token vào, +4 678 token ra, +$0.00819, +59.5 giây (graph trừ flat). Phần luật dựng bằng regex nên không tốn token. *Mỗi câu:* output chỉ tăng 47 → 81 token, nên tiền tăng gần như toàn bộ do **prompt dài hơn ×5.1**: cùng 3 chunk nhưng thêm ~2 900 token dữ kiện graph (tóm tắt vụ án, văn bản các khoản luật, cạnh 1 bước) và thêm một vòng truy vấn Neo4j (+0.53 giây).
>
> Một phần prompt đó là **nhiễu**: ở Q5 graph đưa 32 dữ kiện, trong đó 5/6 tóm tắt vụ án không phải vụ được hỏi, cùng các khoản Điều 249/251/255 của chúng. Ba vụ (36kg, tông CSGT, chuyên án A3-626P) vào qua node chung chung `Substance` "ma túy": tên này nằm trong chuỗi câu hỏi nên thành seed và kéo theo mọi vụ nối vào nó; hai vụ còn lại (Hà Nội, Sầm Sơn) vào qua node `MDMA` cũng có trong câu hỏi (khâu seed của KG-3, suy ra từ các cạnh `INVOLVES` trong dữ kiện). Q2 là ví dụ rõ nhất: cả hai pipeline đều đúng (recall 1.00, judge 2) nhưng graph tốn $0.00058 so với $0.00014 (×4.1).
>
> *Điểm hòa vốn:* theo USD thì graph **không bao giờ hòa vốn** vì đắt hơn cả ở indexing lẫn mỗi câu hỏi (thêm $0.00819 một lần + $0.00045 mỗi câu). Phép so sánh có ý nghĩa là chi phí cho một câu trả lời đúng hoàn toàn (judge = 2): với 6 câu của benchmark, Flat tốn $0.00112 + 6×$0.00013 ≈ $0.0019 cho 2 câu đúng (≈ $0.00095/câu); Graph tốn $0.00931 + 6×$0.00058 ≈ $0.0128 cho 4 câu đúng (≈ $0.0032/câu). Graph đắt hơn ~3.4 lần trên mỗi câu đúng, nhưng với loại câu cross-KB thì Flat không trả lời được ở bất kỳ mức giá nào (mục 2).

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Định nghĩa "tiền chất" nằm gọn trong một chunk luật; graph chỉ thêm token ($0.00019 vs $0.00012). |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Tên bị cáo và án tử hình nằm trong một bài; Flat đủ và rẻ hơn ×4.1. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | **Graph** | Flat trả "Không đủ thông tin" vì chunk tin không có Điều luật; graph đi Person→Case→Crime→Article→khoản 1 và trả đúng 36 tháng, Điều 251, 02–07 năm. |
| Q4 | cross-kb | 0.00 / 0 | 0.67 / 1 | **Graph** (một phần) | Graph nối được Hoàng Nato → Điều 255 nhưng trả sai mức tối đa "7 năm" (đúng: 20 năm/chung thân) vì thiếu khoản 4 (lỗi E2, mục 3). |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 1.00 / 2 | **Graph** | Graph đưa đủ các khoản nêu MDMA của Điều 250 nên chọn đúng "khoản 4"; Flat đoán sai "khoản b)". |
| Q6 | aggregation | 0.00 / 1 | 0.33 / 1 | Hòa | Cả hai liệt kê cùng 3 vụ đúng; recall thấp vì phép đo (lỗi E4, mục 3); graph bị trùng node nên thiếu minh bạch về số vụ. |

**Quy luật:** (a) câu mà đáp án nằm trong **một** nguồn (Q1, Q2) → hòa, Flat rẻ hơn; (b) câu cần **nối hai KB** qua một khóa chung (Q3–Q5) → Graph thắng 3/3, recall trung bình 0.20 → 0.89 và judge 0.33 → 1.67; (c) câu **tổng hợp** (Q6) → graph không thắng rõ vì chất lượng phụ thuộc việc gộp thực thể (lỗi E3), không phải việc có graph hay không.

## 3. Phân tích lỗi (20 điểm)

Các truy vấn Cypher dưới đây chạy trên graph do đúng lần `--judge` ở trên dựng ra. Phần "dữ kiện graph đưa vào prompt" được tái hiện bằng cách embed lại cùng 176 chunk (top_k=3) và gọi `graph.context(...)`; việc này chỉ đọc graph, không ghi lại và không đổi `ket_qua_benchmark_kg.txt`.

### Lỗi E2: Thiếu ngữ cảnh luật — Q4 trả sai khung phạt tối đa dù graph có Điều 255

- **Hiện tượng:** Q4 hỏi hành vi của "Hoàng Nato" và mức phạt tù **tối đa**. Graph trả đúng hành vi và đúng Điều (255) nhưng sai mức: "tối đa 7 năm". Đáp án chuẩn là tù 20 năm hoặc chung thân (khoản 4). Recall 0.67, judge 1.
- **Bằng chứng:**

Câu trả lời của GraphRAG ở Q4 (file kết quả):

```
Giang hồ 'Hoàng Nato' bị bắt về hành vi tổ chức sử dụng trái phép chất ma túy. Hành vi này có thể bị phạt tù tối đa 7 năm theo Điều 255 Bộ luật Hình sự.
```

Dữ kiện luật graph đưa vào prompt cho Q4 (tái hiện `graph.context`; chỉ có khoản 1 của mỗi Điều):

```
[Điều 249 BLHS - Tội tàng trữ trái phép chất ma túy] khoản 1: Người nào tàng trữ trái phép chất ma túy ...
[Điều 251 BLHS - Tội mua bán trái phép chất ma túy] khoản 1: Người nào mua bán trái phép chất ma túy, thì bị phạt tù từ 02 năm đến 07 năm.
[Điều 255 BLHS - Tội tổ chức sử dụng trái phép chất ma túy] khoản 1: Người nào tổ chức sử dụng trái phép chất ma túy dưới bất kỳ hình thức nào, thì bị phạt tù từ 02 năm đến 07 năm.
```

Điều 255 có đủ 4 khung phạt trong graph nhưng không khoản nào `MENTIONS` chất nào:

```cypher
MATCH (:Article {id:'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
OPTIONAL MATCH (cl)-[:MENTIONS]->(s:Substance)
RETURN cl.number AS khoan, cl.penalty AS penalty, count(s) AS so_chat ORDER BY khoan
```

```
khoan=1  penalty='phạt tù từ 02 năm đến 07 năm'              so_chat=0
khoan=2  penalty='phạt tù từ 07 năm đến 15 năm'              so_chat=0
khoan=3  penalty='phạt tù từ 15 năm đến 20 năm'              so_chat=0
khoan=4  penalty='phạt tù 20 năm hoặc tù chung thân'         so_chat=0
```

- **Nguyên nhân:** Ở bước **Cypher của KG-3** (cùng với thiết kế ontology). Quy tắc lọc theo guide là "khoản 1 + các khoản `MENTIONS` một chất mà vụ án `INVOLVES`". Quy tắc này đúng với các tội định khung theo khối lượng chất (Điều 249–251, nên Q5 đúng) nhưng ở Điều 255 các khoản sau định khung theo tình tiết tăng nặng và hậu quả (gây thương tích, làm chết người), không nêu chất nào, nên chỉ còn khoản 1. Prompt cũng không báo cho LLM rằng các khoản khác đã bị lược, nên nó coi khoản 1 là mức cao nhất. Đây là lỗi của **bước lấy ngữ cảnh**, không phải của cầu nối: đường Person→Case→Crime→Article đã đi đúng.
- **Đề xuất sửa:** Trong `Neo4jGraph.context` ([src/graph.py](../src/graph.py)), với mỗi `Article` đã chạm tới, luôn thêm **một dòng thang hình phạt** dựng từ thuộc tính `penalty` có sẵn, ví dụ `Điều 255: kh.1 02–07 năm; kh.2 07–15 năm; kh.3 15–20 năm; kh.4 20 năm hoặc chung thân` (~60 token mỗi Điều, rẻ hơn nhiều so với ~250 token cho văn bản đầy đủ của một khoản), và/hoặc khi câu hỏi có "tối đa/cao nhất" thì lấy thêm khoản có số lớn nhất. Đánh đổi: thêm token cho mỗi Điều (nhưng bù lại có thể bỏ bớt khoản không cần), cần thêm một nhánh Cypher. Chưa áp dụng để giữ nguyên số liệu của lần chạy này.

### Lỗi E4: Phép đo sai — Q6 chấm recall 0.00 / 0.33 cho các câu trả lời về cơ bản đúng

- **Hiện tượng:** Q6 hỏi "những vụ nào liên quan MDMA". Cả Flat (recall 0.00) và Graph (recall 0.33) đều liệt kê đúng ba sự việc của đáp án chuẩn, nhưng bị chấm gần 0 vì `must_include = ["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]` đòi đúng chuỗi tên. Judge cho cả hai 1 (đúng một phần); theo mình hai câu trả lời đều đủ ý chính nên đáng ra có thể được 2.
- **Bằng chứng:**

Flat, Q6 (recall 0.00, judge 1): liệt kê "Vụ việc của Đức", "Vụ việc của Thành", "Vụ việc của Đông (0,686g MDMA ... buồng chữa bệnh)".
Graph, Q6 (recall 0.33, judge 1): liệt kê "Vụ góp tiền mua ma túy tại Hà Nội (Lê Minh Thành)", "Vụ vận chuyển ma túy từ Đức về Việt Nam (9,6kg MDMA)", "Vụ tổ chức sử dụng ma túy tại Sầm Sơn (Lê Văn Đông, 0,686g MDMA)".

Đối chiếu với đáp án chuẩn: "Cái Quang Huy" = vụ vận chuyển từ Đức qua Nội Bài (9,6kg MDMA); "Lê Minh Thành" = vụ góp tiền mua ma túy; "Pháp y tâm thần" = vụ việc ở Viện Pháp y tâm thần có thu giữ MDMA. Hai bài (tựa đề đều nhắc Viện Pháp y tâm thần: `news-100260924105118645` và `news-100260930085028036`) kể về cùng một sự việc và cùng một người:

```cypher
MATCH (:Person {name:'Lê Văn Đông'})-[:INVOLVED_IN]->(k:Case) RETURN k.name AS vu, k.doc_id AS doc ORDER BY doc
```

```
vu='Vụ án tại Viện Pháp y tâm thần Trung ương'  doc='news-100260924105118645'
vu='Vụ tổ chức sử dụng ma túy tại Sầm Sơn'      doc='news-100260930085028036'
```

Hai bài đều có "MDMA" và "Lê Văn Đông" (grep trong `data/drug_news/`).

- **Nguyên nhân:** Nằm ở **phép đo**, không ở pipeline. (1) `recall` đếm chuỗi con theo cách đặt tên của đáp án chuẩn (tên người, tên bài), trong khi tên vụ trong graph do LLM tự đặt ("Vụ vận chuyển ma túy từ Đức về Việt Nam") nên cùng một sự việc không khớp từ khóa. (2) Judge so với cùng đáp án chuẩn nên cũng bị lệch tên và chỉ cho 1. Kết quả: thang `recall` không phân biệt được hai câu trả lời tương đương, và không phản ánh việc graph thật sự gộp được sự việc.
- **Đề xuất sửa:** Với câu aggregation, chấm theo **tập sự việc** thay vì chuỗi từ khóa: gắn mỗi sự việc kỳ vọng với một danh sách bí danh được chấp nhận (ví dụ `["Cái Quang Huy", "từ Đức", "Nội Bài"]`) hoặc với `doc_id` của bài nguồn, và tính precision/recall trên tập đó. Đánh đổi: phải viết tay danh sách bí danh cho mỗi câu (công gán nhãn) nhưng không tốn thêm token; cũng có thể bắt agent trả kèm `doc_id` nguồn để chấm không phụ thuộc cách đặt tên. Không sửa `data/benchmark_kg.json` trong bài này vì số liệu phải khớp file benchmark đã nộp.

### Lỗi E3: Trùng thực thể — cùng một thứ ngoài đời thành nhiều node (ảnh hưởng tới Q6)

- **Hiện tượng:** Cùng một chất, một người và một sự việc bị tách thành nhiều node nên câu hỏi tổng hợp phải đếm lại tay.
- **Bằng chứng:**

```cypher
MATCH (a:Substance),(b:Substance) WHERE id(a) < id(b) AND toLower(a.name)=toLower(b.name) RETURN a.name AS a, b.name AS b
```

```
a='Ketamine'         b='ketamine'
a='methamphetamine'  b='Methamphetamine'
```

```cypher
MATCH (:Person {name:'Dương Minh Tuấn'})-[:INVOLVED_IN]->(k:Case) RETURN k.name AS vu, k.doc_id AS doc ORDER BY doc
```

```
Vụ bắt giang hồ 'Hoàng Nato' và 126 người liên quan 8 đường dây ma túy      news-100260920221957595
Vụ bắt giữ TikToker Phannhibeauty và giang hồ 'Hoàng Nato'                  news-100260922111804786
Vụ sử dụng ma túy etomidate của Hoàng Nato và Phan Kim Nhi                  news-100260924095400982
Vụ triệt phá 8 đường dây ma túy tại TP.HCM                                  news-100260925144412498
```

Tên chất đồng nghĩa/chung chung cũng thành node riêng: `MDMA` và `thuốc lắc` (vụ Hoàng Nato dùng "thuốc lắc"), cùng `ma túy`, `chất ma túy`, `ma túy tổng hợp` (xem `MATCH (s:Substance) RETURN s.name`: 17 node cho ~9 chất thật). Hệ quả cho Q6: Cypher trả thẳng lời giải cho "vụ nào liên quan MDMA/thuốc lắc" cho **5 dòng** (`Vụ góp tiền mua ma túy tại Hà Nội`, `Vụ vận chuyển ma túy từ Đức về Việt Nam`, `Vụ tổ chức sử dụng ma túy tại Sầm Sơn`, `Vụ án tại Viện Pháp y tâm thần Trung ương`, `Vụ bắt giang hồ 'Hoàng Nato' ...`) trong khi ngoài đời chỉ có ~4 sự việc (Sầm Sơn và Pháp y là một); và câu trả lời của GraphRAG chỉ nêu 3 vụ: sự việc Viện Pháp y chỉ xuất hiện dưới tên "vụ Sầm Sơn" (tóm tắt của `Vụ án tại Viện Pháp y` không có trong dữ kiện, xem nguyên nhân bên dưới), còn vụ Hoàng Nato bị bỏ sót (khóa "thuốc lắc" khác "MDMA").
- **Nguyên nhân:** **Thiết kế ontology / khóa định danh.** `MERGE` theo `name` chỉ gộp khi chuỗi **giống hệt**: `Case.name` do LLM đặt riêng cho từng bài nên không bao giờ trùng giữa hai bài về cùng một sự việc; `Substance.name` phân biệt hoa/thường và LLM không chuẩn hóa về `SUBSTANCES`. Ngoài ra `Person.name` làm khóa toàn cục gộp được Hoàng Nato qua 4 bài (đúng) nhưng cũng sẽ gộp nhầm hai người cùng tên. Thêm nữa, tóm tắt `Vụ án tại Viện Pháp y` còn bị cắt vì `MAX_CASES = 6` trong `context()` sắp xếp theo tên (alphabet) chứ không theo độ liên quan, nên vụ này chỉ xuất hiện dưới dạng một cạnh `INVOLVES` mà không có tóm tắt.
- **Đề xuất sửa:** (a) Chuẩn hóa `Substance` trong `add_news_case`: `link_entity(name, SUBSTANCES, normalize=str.lower)` cộng một bảng bí danh nhỏ (`thuốc lắc → MDMA`, `ma túy đá → Methamphetamine`), chất không khớp thì bỏ hoặc gắn nhãn `unspecified`. (b) Khóa `Case` theo `doc_id` (xác định) và thêm một bước gộp sự việc theo tập người chung, hoặc thêm node `Event` do LLM gán nhóm. (c) Đổi `ORDER BY retrieved DESC, name` thành xếp theo số seed liên quan và bỏ qua các node chung chung ("ma túy") khi chọn seed. Đánh đổi: (a) rẻ (không thêm gọi LLM); (b) tốn thêm một lượt gọi LLM hoặc logic so khớp người, dễ gộp nhầm nếu hai vụ có chung bị cáo.

## 4. Kết luận (5 điểm)

Dựa trên số liệu mục 1–2 (một lần chạy, 6 câu, `gpt-4o-mini`; số node graph và đôi chỗ trả lời có thể dao động giữa các lần vì LLM trích xuất):

> **Nên dùng KG khi câu hỏi phải nối hai nguồn qua một khóa chung đáng tin.** Ở bộ này đó là tên tội danh: trên 3 câu cross-KB (Q3–Q5) Graph thắng 3/3, recall trung bình **0.20 → 0.89** và judge **0.33 → 1.67**; Flat trả "Không đủ thông tin" ở Q3 và Q4 vì không chunk nào chứa cả tên bị cáo lẫn Điều luật. **Flat RAG là đủ** khi đáp án nằm trong một nguồn: Q1 và Q2 cùng đạt recall 1.00 / judge 2, trong khi Graph tốn thêm ×1.6 đến ×4.1 tiền mỗi câu. Với aggregation (Q6) graph chưa tự động tốt hơn: kết quả phụ thuộc chất lượng gộp thực thể (lỗi E3), không chỉ vào việc có graph.
>
> **Chi phí:** Graph đắt hơn ở mọi N (indexing ×8.3, mỗi câu ×4.5, prompt ×5.1), nhưng con số tuyệt đối nhỏ ($0.0093 dựng một lần, $0.00058 mỗi câu); trên mỗi câu đúng hoàn toàn thì Graph ≈ $0.0032 so với ≈ $0.00095 của Flat (≈ 3.4 lần), đổi lại trả lời được loại câu mà Flat không thể. **Điều kiện cụ thể để đáng tiền:** (1) có khóa nối hai nguồn chuẩn hóa được (ở đây 13 tên tội, `link_entity` nối được 13/14 vụ; vụ còn lại là vụ tông cảnh sát giao thông, không thuộc 13 tội nên không nối là hợp lý); (2) phần đáng kể trong số câu hỏi là câu nối nguồn (ở benchmark này là 3/6); (3) chấp nhận bỏ công kiểm soát chất lượng trích xuất (trùng node, lọc khoản). Nếu đa số câu hỏi chỉ cần một đoạn văn, Flat RAG đủ và rẻ hơn.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.03s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openai:gpt-4o-mini | embedding = openai:text-embedding-3-small
[OK] KG-2 build_graph: 146 node / 289 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 13 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00065. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: Cái Quang Huy

## Vấn đề gặp phải (không tính điểm)

Không có lỗi nào chặn bài. Ghi chú: kết quả trích xuất bằng LLM dao động nhẹ giữa các lần chạy (cùng code: lần `--check` đầu 148 node / 293 cạnh, lần sau 146 / 289; bản dựng đầy đủ của `--judge` là 205 node / 383 cạnh), nên số node trong báo cáo là của đúng lần `--judge` nộp kèm.

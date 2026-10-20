## Vấn đề: vì sao RAG ngây thơ không đủ

**RAG ngây thơ**, tức là đổ tài liệu thô vào kho không xử lý. Điều này đúng dù dữ liệu nằm trong vector database rồi được truy xuất vào ngữ cảnh, hay được nhét thẳng vào long context. Không có trích xuất tri thức và tiền xử lý có cấu trúc, mô hình không dùng thông tin đó hiệu quả và đáng tin cậy được.

Ví du 1: đếm mèo đen và mèo trắng
Giả sử kho có 100 tài liệu độc lập: 90 về mèo đen, 10 về mèo trắng, mỗi tài liệu là một chunk. Người dùng hỏi “Tỉ lệ là bao nhiêu?”, và hệ thống lấy top-k = 20:

- Chỉ 20 trên 100 chunk được đưa vào ngữ cảnh. **80% dữ liệu không bao giờ đến được mô hình.**
- Mô hình thấy, chẳng hạn, 15 mèo đen và 3 mèo trắng (chỉ 18 chunk thực sự liên quan) và kết luận sai.

Có một điểm tinh tế: 20 chunk này **không phải mẫu ngẫu nhiên**. Chúng được chọn theo độ tương đồng với câu hỏi, nên tỉ lệ trong mẫu có thể lệch bất kỳ hướng nào. Ngay cả về mặt thống kê, mẫu này cũng không đại diện.

Bản chất vấn đề là câu hỏi dạng **tổng hợp** (đếm, tỉ lệ, trung bình, “tất cả”, “có bao nhiêu”) đòi hỏi nhìn **toàn bộ** dữ liệu. Truy xuất top-k thì theo định nghĩa chỉ trả về **một phần**. Hai thứ này mâu thuẫn từ gốc.

## Giải pháp: Lập chỉ mục

Ý tưởng: **dùng LLM tổ chức tri thức trước khi lập chỉ mục**, bằng cách tóm tắt, trừu tượng hóa và thiết lập quan hệ. Đánh đổi là tốn nhiều compute hơn lúc đầu để đổi lấy chất lượng truy xuất tốt hơn. Hiện có hai hướng chính: **cây phân cấp (RAPTOR)** và **đồ thị thực thể–quan hệ (GraphRAG)**.

### RAPTOR: cây tóm tắt đệ quy

**RAPTOR** là viết tắt của _Recursive Abstractive Processing for Tree-Organized Retrieval_, tức “xử lý trừu tượng đệ quy cho truy xuất tổ chức theo cây”. Cách làm là **từ dưới lên (bottom-up)**:

1. **Tạo lá.** Cắt tài liệu dài thành các chunk nhỏ, gọi là _leaf nodes_ (nút lá).
2. **Phân cụm.** Dùng thuật toán clustering gom các lá có ngữ nghĩa tương tự. Tác giả ví như tự động xếp sách thư viện theo chủ đề: tính độ giống nhau giữa các cuốn sách (các chunk), gom những cuốn giống nhau nhất lại, mỗi nhóm là một chủ đề.
3. **Tóm tắt.** LLM viết một bản tóm tắt cấp cao cho mỗi cụm. Bản tóm tắt này trở thành **nút cha**.
4. **Đệ quy.** Lặp lại bước 2–3 trên các nút cha: phân cụm các tóm tắt, rồi tóm tắt tiếp, cho đến khi còn một gốc (hoặc vài nút gốc).

Kết quả là một **cây tri thức**: lá là chi tiết cụ thể, gốc là khái quát rộng.

```
                 [Kiến trúc CPU x86]                ← gốc (khái quát)
                /                    \
   [Tiến hóa tập lệnh SIMD x86]   [Quản lý bộ nhớ]   ← nút tóm tắt
      /              \
"SSE2 hỗ trợ      "SSE4.1 thêm lệnh                  ← lá (chi tiết)
 số nguyên 128-bit" so sánh chuỗi"
```

**Ví dụ trong sách:** các lá nói về SSE (“SSE2 hỗ trợ phép toán số nguyên 128-bit”, “SSE4.1 bổ sung lệnh so sánh chuỗi”) rơi vào cùng một cụm. Hệ thống sinh nút cha “Sự tiến hóa của tập lệnh SIMD x86”. Nhờ vậy, tài liệu có thể được truy xuất ở **nhiều mức độ chi tiết**:

- Câu hỏi chi tiết (“SSE4.1 có lệnh gì mới?”) khớp với lá.
- Câu hỏi khái quát (“Tập lệnh SIMD của x86 phát triển ra sao?”) khớp với nút tóm tắt. Một lá đơn lẻ không thể trả lời tốt câu này, vì đáp án nằm rải rác ở nhiều lá.

Đây chính là lời giải cho ví dụ mèo đen/trắng ở trên: **nút tóm tắt chứa thông tin tổng hợp mà không lá nào có**.

### GraphRAG: đồ thị thực thể–quan hệ

GraphRAG mô hình hóa tri thức thành **đồ thị tri thức** (knowledge graph) gồm các **bộ ba** (triple) dạng _chủ ngữ – vị ngữ – tân ngữ_:

- (Bắc Kinh, là thủ đô của, Trung Quốc)
- (Trương Tam, làm việc tại, Tencent)

Gom đủ nhiều bộ ba sẽ thành một mạng lưới tri thức. Ưu điểm cốt lõi thể hiện ở hai điểm.

**Ưu điểm 1: Suy luận quan hệ đa bước (multi-hop reasoning).**  
Tác giả gọi đây là năng lực **không thể thay thế** của đồ thị tri thức. Ví dụ câu hỏi: “Địa chỉ bệnh viện của bác sĩ của tôi là gì?” Hệ thống phải lần theo chuỗi:

```
người dùng → (có bác sĩ) → BS. Trương → (làm tại) → BV Hữu Nghị → (địa chỉ) → số 1 Trần Khánh Dư
```

Trong một kho bộ nhớ phẳng, có hai khả năng:

- Phải truy xuất nhiều lần độc lập rồi nhờ LLM ghép lại. Cách này chậm, và chỉ cần một bước truy xuất hụt là chuỗi bị đứt.
- Hoặc đơn giản là không biểu diễn được. Câu “địa chỉ bệnh viện của bác sĩ của tôi” không giống ngữ nghĩa với chunk “BV Hữu Nghị ở số 1 Trần Khánh Dư”, vì chunk đó không nhắc gì đến “tôi” hay “bác sĩ”.

**Ưu điểm 2: Phân biệt thực thể (entity disambiguation).**

|           | Phân biệt nghĩa của từ (Word Sense Disambiguation) | Phân biệt thực thể (Entity Disambiguation)  |
| --------- | -------------------------------------------------- | ------------------------------------------- |
| Ví dụ     | “bank” là bờ sông hay ngân hàng?                   | Hai người thật cùng tên “BS. Trương”        |
| Bản chất  | Một từ, nhiều nghĩa                                | Một tên, nhiều đối tượng thực tế            |
| Cách giải | Embedding có ngữ cảnh là đủ                        | Phải lưu **tri thức về chính các thực thể** |
**Quy trình xây dựng GraphRAG:**

1. LLM **trích xuất thực thể** từ văn bản (người, địa điểm, khái niệm, thuật ngữ).
2. LLM **trích xuất quan hệ** giữa các thực thể.
3. Chạy **thuật toán phát hiện cộng đồng** (community detection) trên đồ thị để tìm các cụm thực thể liên kết chặt.
4. **Sinh tóm tắt** cho từng cộng đồng. Kết quả là các nhóm chủ đề tự nhiên, giống một sơ đồ tư duy (mind map).

### Ví dụ minh hoạ GraphRag của Microsoft

GraphRag trên truyện A Christmas Carol (hồn ma đêm giáng sinh) của Charles Dickens

#### Câu chuyện, tóm tắt một dòng

Ông chủ keo kiệt **Scrooge** được hồn ma người cộng sự cũ **Jacob Marley** và ba hồn ma Giáng sinh (Quá khứ, Hiện tại, Tương lai) đưa đi xem lại đời mình. Một trong những cảnh chính là gia đình nghèo của người thư ký **Bob Cratchit**, có cậu con trai ốm yếu **Tiny Tim**. Cuối truyện, Scrooge thay đổi và trở thành “người cha thứ hai” của Tiny Tim.
#### Bước 0: chia nhỏ văn bản

Cả cuốn truyện được cắt thành **42 đoạn**, mỗi đoạn khoảng 1.200 token. Ta theo dõi **đoạn số 21**, cảnh gia đình Cratchit chuẩn bị bữa tối Giáng sinh.

#### Bước 1: LLM trích xuất thực thể

Từ riêng đoạn 21, LLM tìm ra **14 thực thể**:

| Loại     | Thực thể                                                                       |
| -------- | ------------------------------------------------------------------------------ |
| Người    | Bob Cratchit, Mrs. Cratchit, Peter, Martha, Belinda, Tiny Tim, Young Cratchits |
| Địa điểm | London, Parks, Baker’s Shop                                                    |
| Sự kiện  | Christmas, Christmas Day, Christmas Dinner, **Goose**                          |

Mỗi thực thể kèm một mô tả. Ví dụ Tiny Tim: _“con út của Bob và Mrs. Cratchit, ốm yếu, phải dùng nạng, nhưng hiền lành và lạc quan.”_ Gộp cả 42 đoạn, toàn truyện có **529 thực thể**.
#### Bước 2: LLM trích xuất quan hệ

Từ đoạn 21 có **26 quan hệ**. Mỗi quan hệ có trọng số; quan hệ được nhắc lại ở nhiều đoạn sẽ có trọng số cao hơn.

|Quan hệ|Mô tả (dịch)|Trọng số|
|---|---|---|
|Bob Cratchit — Tiny Tim|Người cha tận tụy, hay bế con từ nhà thờ về|78|
|Bob Cratchit — Mrs. Cratchit|Vợ chồng, cùng chăm lo gia đình|57|
|Bob Cratchit — Peter Cratchit|Cha và con trai cả|50|
|Cratchit Family — Goose|Con ngỗng là món chính của bữa tối Giáng sinh|9|
|Baker’s Shop — Goose|Ngỗng được nướng ở lò bánh mì|8|

Toàn truyện có **978 quan hệ**. Riêng Tiny Tim nối với **Scrooge** bằng một cạnh trực tiếp: _“Scrooge xúc động trước bệnh tình của Tiny Tim, thay đổi và trở thành người cha thứ hai của cậu.”_ Đây chính là mạch chính của truyện.

### Bước 3: phát hiện cộng đồng

Thuật toán Leiden chia đồ thị thành **122 cộng đồng ở 4 cấp**: cấp 0 là các nhóm lớn, mỗi nhóm lại chia nhỏ ở cấp sau. Các cộng đồng lớn ở cấp 0 trùng gần như hoàn hảo với các “thế giới” trong truyện, dù thuật toán không đọc truyện mà chỉ nhìn các cạnh:

|Cộng đồng|Số nút|Thành viên chính|Trong truyện là|
|---|---|---|---|
|C3|50|Bob, Mrs. Cratchit, Tiny Tim, Peter, Martha, nhà thờ, bữa tối|Gia đình Cratchit|
|C4|70|Ebenezer Scrooge, Jacob Marley, hồn ma Marley|Scrooge và các hồn ma|
|C14|28|Fezziwig, Mrs. Fezziwig, Dick Wilkins|Thời trẻ của Scrooge, làm học việc cho ông Fezziwig|
|C13|22|Fred, cháu dâu của Scrooge, Topper|Bữa tiệc nhà người cháu Fred|
|C9|7|Old Joe, Mrs. Dilber, bà giúp việc|Những kẻ bán đồ của người chết (viễn cảnh tương lai)|
|C5|4|Belle và gia đình|Người yêu cũ Scrooge đánh mất|

Gia đình Cratchit (C3) còn được chia nhỏ ở cấp 1. Ví dụ C43 “Bữa tối Giáng sinh” gồm ngỗng, nước sốt táo, bánh pudding, lò bánh mì; C45 xoay quanh “Tiny Tim và sự thay đổi của Scrooge”.

File **carol_graph.png** vẽ 30 nhân vật chính với 65 cạnh có thật giữa họ, mỗi màu là một cộng đồng. Nhóm Cratchit (xanh dương) và nhóm Old Joe (vàng) tách rời rõ ràng; Scrooge nằm ở giữa, nối với mọi nhóm.

![](iamges/carol_graph.png)

### Bước 4: LLM viết báo cáo cho từng cộng đồng

Báo cáo thật của C3 (dịch phần tóm tắt):

> **“Gia đình Cratchit và cộng đồng Giáng sinh của họ”**: Cộng đồng xoay quanh gia đình Cratchit, ngôi nhà khiêm tốn của họ và mạng lưới quan hệ, sự kiện trong dịp Giáng sinh. Thực thể chính gồm Bob Cratchit, vợ và các con (nổi bật là Tiny Tim), nhà thờ, Scrooge và bữa tối Giáng sinh. Sự kiên cường, ấm áp và gắn bó của gia đình trước cái nghèo được nhấn mạnh, cũng như tác động của sự thay đổi trong lòng Scrooge.

### Dùng đồ thị để trả lời câu hỏi

**Hỏi chi tiết** (local search): _“Tiny Tim có quan hệ gì với Scrooge?”_ Hệ thống tìm nút Tiny Tim, đi theo các cạnh và thấy ngay cạnh Scrooge — Tiny Tim. Trả lời: Scrooge thay đổi vì cậu bé và trở thành người cha thứ hai của cậu.

**Hỏi tổng quát** (global search): _“Truyện có những nhóm nhân vật nào?”_ Không đoạn văn nào chứa câu trả lời này. Hệ thống đọc các báo cáo cộng đồng cấp 0 rồi tổng hợp lại, cho ra đúng các nhóm trong bảng ở Bước 3.


### Giới hạn của đồ thị tri thức khi làm bộ nhớ người dùng

**Giới hạn 1: Suy giảm ngữ nghĩa (semantic degradation).**  
Ví dụ câu: “Nếu tuần sau trời mưa, tôi sẽ hủy chuyến đi biển và đi bảo tàng thay thế.” Câu này chứa:

- **Logic điều kiện**: _nếu_ mưa _thì_…
- **Phụ thuộc thời gian**: _tuần sau_.
- **Quan hệ thay thế**: bảo tàng _thay cho_ biển, chỉ trong điều kiện đó.

Phân rã thành bộ ba chỉ còn:

- (người dùng, lên kế hoạch, đi biển)
- (người dùng, có kế hoạch dự phòng, đi bảo tàng)

Điều kiện “nếu mưa” và mốc “tuần sau” **mất hoàn toàn**. Một tháng sau, Agent có thể nhắc người dùng “bạn có kế hoạch đi biển” dù chuyến đó đã bị hủy hoặc đã qua. Bộ ba đơn giản quá mức cho nhiều kiểu ngữ nghĩa phức tạp của ngôn ngữ tự nhiên.

**Giới hạn 2: Phụ thuộc chất lượng trích xuất.**  
Độ chính xác của bộ ba phụ thuộc nặng vào khả năng hiểu của LLM. Trích xuất sai sẽ dẫn đến **ô nhiễm tri thức** (knowledge contamination). Cạnh sai trong đồ thị còn nguy hiểm hơn một chunk sai, vì suy luận đa bước sẽ lan truyền lỗi qua cả chuỗi.

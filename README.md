# Capabilities

Khi nhận câu hỏi người dùng thì luồng hoạt động của Agent sẽ như sau:

```mermaid
flowchart TD
    start["Start"]

    subgraph sub1 [" "]
        direction LR
        Kw["Keyword"]
        a["CCCD, CMND, SĐT..."]
        b["Biển số xe"]
        c["Họ tên"]
        Kw --> a
        Kw --> b
        Kw --> c
    end

    embed["Embedding"]

    subgraph sub2 [" "]
        valid["Validator"]
        cap1["CAP1"]
        cap2["CAP2"]
        cap3["CAP3"]
        valid --> cap1
        valid --> cap2
        valid --> cap3
    end

    plan["PLAN (Thực thi plan)"]
    valid2["Validator"]
    finish["End"]

    start --> sub1
    sub1 --> embed
    embed -->|"search CAP"| sub2
    sub2 -->|"validate CAP and chọn CAP"| plan
    plan --> valid2
    valid2 --> finish
```

Chi tiết các **Plan** của từng cap sẽ được viết ở dưới.

## 1. Tra cứu thông tin công khai người theo tên

### Description

Người dùng hỏi một người là ai / làm gì (thường là người có thông tin công khai trên mạng).
Ưu tiên dữ liệu công khai (Person -> Person_Official).

group: identity
anchor_types: [person_name, description]
required_slots: [person_query]

### Example utterances

- Nguyễn Văn A là ai?
- Người có tên là Nguyễn Văn A là ai?
- Cho tôi biết thông tin của Nguyễn Văn A?
- Đỗ Văn C làm gì?
- Người tên Đỗ Văn C có công việc gì?
- Đỗ Văn C có nghề nghiệp như nào?
- Thông tin công khai của Nguyễn Văn A
- Đỗ Văn C có những hoạt động gì?

### Plan

- Step (s*): là một bước trong một plan thực hiện capability, trong mỗi s* có tool, đầu vào đầu ra, mô tả.
- Decision (d*): bản chất không chưa tool, chỉ rẽ nhánh để biết tại một step thì đi tiếp đến đâu.
- Terminal: ANSWER | CLARIFY | NOT_FOUND (phân loại đầu ra của plan)

```mermaid
flowchart TD
    s1["s1 · Tra person-idx theo tên (tên là keyword do LLM phát hiện)"]
    d1{{"d1 · persons có rỗng không?"}}
    s2["s2 · Gom Person → clusters (Official + Person mồ côi)"]
    d2{{"d2 · có bao nhiêu cụm định danh?"}}
    s3["s3 · Fallback: tra netizen-idx theo tên"]
    d3{{"d3 · có bao nhiêu hồ sơ định danh?"}}
 
    ANSWER([ANSWER])
    CLARIFY([CLARIFY])
    NOT_FOUND([NOT_FOUND])
 
    s1 --> d1
    d1 -->|"len(persons) == 0"| s3
    d1 -->|else| s2
 
    s2 --> d2
    d2 -->|"len(clusters) == 1"| ANSWER
    d2 -->|else| CLARIFY
 
    s3 --> d3
    d3 -->|"len(netizens) == 0"| NOT_FOUND
    d3 -->|"len(netizens) == 1"| ANSWER
    d3 -->|else| CLARIFY
```

### Answer policy

Phần này giúp LLM trả lời đúng hướng hơn từ quyết định Terminal của  Plan:

- ANSWER: Khi d2 có đúng một cụm định danh có nghĩa là chỉ có 1 Person_Official hoặc 1 Person mà không có relation đến Person_Official thì sẽ lấy dữ liệu trả lời người dùng. Ngoài ra khi fallback sang Netizen mà xảy ra trường hợp chỉ có 1 netizen trả về (Hiếm) thì cũng sẽ trả lời câu hỏi.
- CLARIFY: Khi có nhiều cụm định danh có nghĩa là có nhiều Person_Official hoặc nhiều Person không tìm thấy relation với Person_Official thì cần liệt kê một lượng nhỏ bản ghi kết hợp với hỏi lại user
- NOT_FOUND: persons và netizen đều rỗng thì nên trả lại không tìm thấy thông tin không suy đoán.

## 2. Tra cứu hồ sơ định danh của người.

### Description

Người dùng hỏi thông tin cá nhân của một người theo **tên** hoặc **số CCCD, CMND, SĐT, ...**, có thể phải trả về ngày sinh, địa chỉ, quê quán, dân tộc, tình trạng hôn nhân, số điện thoại, ...

- anchor_types: [person_name, cccd]
- required_slots: [person_name | cccd, cmnd]
- optional_slots: [attribute]

### Example utterances

- Nguyễn Văn A sinh năm bao nhiêu?
- Thông tin định danh của Nguyễn THị A
- quê quán và dân tộc của bà Phạm Thị Thanh
- số CCCD của ông Nguyễn Văn Ninh
- Nguyễn Văn A đã kết hôn chưa
- ....

### Plan

```mermaid
flowchart TD
    d1{{"d1 · đầu vào là mã định danh hay tên?"}}
    s1["s1 · Tra hồ sơ định danh theo mã (CCCD/CMND/SĐT/MST)"]
    s2["s2 · Tra person-idx theo tên"]
    d2{{"d2 · kết quả tra theo mã có rỗng không?"}}

    d6{{"d6 · kết quả của s2 person có rỗng không?"}}

    s3["s3 · Gom Person cụm định danh (Official + mồ côi)"]
    d3{{"d3 · có bao nhiêu cụm?"}}
    s4["s4 · Bridge từ Person hoặc Person_Official qua Netizen"]
    d4{{"d4 · Có đường đi hay không?"}}
    s5["s5 · Fallback: tra netizen/citizen theo tên"]
    d5{{"d5 · còn lại bao nhiêu người?"}}

    s6["s6 · Lấy hồ sơ đầy đủ theo CCCD (gộp citizen + netizen)"]

    ANSWER([ANSWER])
    CLARIFY([CLARIFY])
    NOT_FOUND([NOT_FOUND])

    d1 -->|"là mã định danh"| s1
    d1 -->|"là tên người"| s2
    d1 -->|else| CLARIFY

    s2 -->d6
    d6 -->|"else"| s3
    d6 -->|"len(persons) == 0"| s5

    s1 --> d2
    d2 -->|"rỗng, fallback theo tên"| CLARIFY
    d2 -->|"không rỗng"| s6

    s2 --> s3
    s3 --> d3
    d3 -->|"len == 0"| s5
    d3 -->|"len == 1"| s4
    d3 -->|else| CLARIFY

    s4 --> d4
    d4 -->|"nối được"| s6
    d4 -->|"không nối được"| s5

    s5 --> d5
    d5 -->|"len == 0"| NOT_FOUND
    d5 -->|"len == 1"| s6
    d5 -->|else| CLARIFY

    s6 --> ANSWER
```


### Answer policy

**ANSWER** — đã xác định đúng một người và có hồ sơ. Trả **thẳng** thông tin được hỏi (ngày
sinh, quê quán, dân tộc, hôn nhân, SĐT...). Nếu người dùng không nói rõ hỏi mục nào → tóm tắt
hồ sơ cơ bản. Thông tin được hỏi mà hồ sơ **không có** → nói rõ "hệ thống không có thông tin
này", KHÔNG suy đoán.
 
**CLARIFY** — còn nhiều người trùng tên. Liệt kê tối đa 3 người kèm thông tin phân biệt (năm
sinh, vài số cuối CCCD, quê quán) rồi hỏi người dùng muốn ai. KHÔNG gộp thành một người, KHÔNG
tự chọn.
 
**NOT_FOUND** — không tìm thấy hồ sơ nào. Trả: "Không tìm thấy hồ sơ định danh cho người này."
KHÔNG suy đoán.


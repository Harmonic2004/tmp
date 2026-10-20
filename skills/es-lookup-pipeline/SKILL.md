---
name: es-lookup-pipeline
description: Tra cứu dữ liệu liên quan đến các thực thể được lưu trữ trên Elasticsearch từ thông tin người dùng cung cấp. Dùng khi muốn biết thông tin về các thực thể.
---

# Elasticsearch Lookup Pipeline

Tài liệu hướng dẫn thực hiện các bước cụ thể trong một pipeline tra cứu để đưa ra câu trả lời cho các câu hỏi tra cứu dữ liệu về thực thể. Cách dữ liệu được lưu (loại thực thể, index, nhóm thông tin, field, quan hệ) nằm ở skill `es-data-explorer`.

## Thuật ngữ

| Thuật ngữ           | Ý nghĩa                                                                               |
| ------------------- | ------------------------------------------------------------------------------------- |
| `subject`           | Thực thể mà câu hỏi muốn biết thông tin hoặc dữ liệu.                                 |
| `anchor properties` | Bộ thông tin đủ mạnh để bắt đầu tra cứu. Xem điều kiện ở Bước 2.                      |
| `root entity`       | Thực thể trong câu hỏi có `anchor properties`, dùng để bắt đầu tra cứu.               |
| `cluster`           | Một nhóm kết quả do tool `search` trả về cho một thực thể.                            |
| `cluster tree`      | Cây ghi lại các `cluster` tìm được từ một lần tìm `root entity`. Mỗi ứng viên `root entity` ở Bước 2 có một cây riêng (`t1`, `t2`, ...). |
| `path`              | Chuỗi các `cluster` đã đi từ `root entity` đến vị trí hiện tại.                       |
| `dead end`          | Lần gọi `search` không trả về `cluster` nào.                                          |
| `branch`            | Một `cluster` khác chưa được đi, nằm cùng cấp với `cluster` đã chọn; kể cả cây của các ứng viên `root entity` không được chọn. |
| `backtrack`         | Quay lại đi một `branch` chưa đi.                                                     |
## Quy trình Thực hiện các bước theo thứ tự. 

Khi một bước yêu cầu hỏi người dùng, gọi tool `ask_user_question`, sau đó dừng lại và chờ câu trả lời.
### Bước 1. Chuẩn hoá

1. Nạp skill `es-data-explorer` và đọc file chi tiết của mọi loại thực thể có trong câu hỏi. Xong khi đã biết nhóm thông tin (Bảng 1 của `es-data-explorer`) và field của mọi thông tin trong câu hỏi.
2. Đọc `references/normalize-input.md`. Chuẩn hoá mọi giá trị `identifier` và `time` mà người dùng cung cấp trong câu hỏi.
3. Rà soát các quan hệ giữa các thực thể trong câu hỏi. Nếu có quan hệ chưa rõ ràng, hỏi lại người dùng, rồi dừng và chờ trả lời.

### Bước 2. Chọn root entity theo dữ liệu

Một thực thể có `anchor properties` khi thoả một trong hai điều kiện (nhóm theo Bảng 1 của `es-data-explorer`):

- Có ít nhất một thông tin nhóm 1.
- Không có thông tin nhóm 1, nhưng có ít nhất một thông tin nhóm 2 và ít nhất một thông tin khác không thuộc nhóm `category`.

`root entity` được chọn theo kết quả tìm thật, không chọn trước:

1. **Ứng viên.** Mọi thực thể có `anchor properties` là ứng viên `root entity`. Nếu không có ứng viên nào, hỏi lại người dùng, rồi dừng và chờ trả lời.
2. **Tìm thử các ứng viên.** Gọi `search` cho mọi ứng viên cùng lúc, mỗi ứng viên là một lần tìm `root entity`: `entity` là ứng viên, cùng `subject`, `anchor` và `hint` theo `references/finder.md`. Mỗi lần tìm mở một `cluster tree` riêng. Sau đó gọi `ranking_cluster` cho mọi cây cùng lúc, với `of` là id cây (`t1`, `t2`, ...).
3. **Chọn `root entity`.** So `cluster` có `rank` 1 của từng ứng viên, chuyển sang tiêu chí kế tiếp khi chưa phân thắng bại:
    1. Khớp nhiều thông tin hơn.
    2. Ít người hơn.
    3. Ứng viên gần `subject` hơn (ít quan hệ hơn).

    Nêu lựa chọn kèm `cluster` rank 1 của từng ứng viên (signature, số docs, số người).

### Bước 3 <a id="buoc-3"></a>. Tìm kiếm 

Mỗi lần gọi `search`, tool ghi các `cluster` tìm được vào `cluster tree`. Cây của các ứng viên không được chọn ở Bước 2 được giữ lại làm `branch` để `backtrack`.

1. **Bắt đầu từ `root entity`.** `path` bắt đầu từ `cluster` có `rank` 1 trong cây của `root entity` đã chọn ở Bước 2; không gọi `search` lại. Sang Bước 3.3.
    
2. **Chọn `cluster`.** Gọi `ranking_cluster` với `of` là `fromCluster` đã dùng ở lần `search` vừa rồi, rồi đi theo `cluster` có `rank` bằng 1. Nếu lần `search` không ra `cluster` nào thì đó là `dead end`, không cần gọi `ranking_cluster`.
    
3. **Chọn thực thể kế tiếp.** Trong các thực thể chưa xét và có quan hệ trực tiếp với một thực thể trên `path`, chọn theo thứ tự ưu tiên:
    
    1. Thực thể gần `subject` hơn (ít quan hệ hơn) đi trước, để đường tới `subject` được xác minh sớm nhất.
    2. Nếu bằng nhau, thực thể có thông tin ở nhóm nhỏ hơn đi trước, vì thông tin mạnh loại được `cluster` sai sớm nhất.
4. **Tìm thực thể kế tiếp.** Gọi `search` với:
    
    - `fromCluster` là `cluster` vừa chọn ở bước 3.2.
    - `relation` là quan hệ giữa thực thể kế tiếp và thực thể liên kết.
    - `hint` theo `references/finder.md`.
    - Thêm `linkFrom` là `cluster` của thực thể liên kết, chỉ khi thực thể liên kết không phải thực thể của `fromCluster` mà là một thực thể khác trên `path`.
    
    Tool tự lấy `anchor` từ `cluster` liên kết.
    
5. **Lặp lại** các bước 3.2 đến 3.4 cho đến khi một trong hai điều kiện xảy ra: không còn thực thể chưa xét, hoặc lần `search` vừa rồi là `dead end`.
### Bước 4. Trả lời

1. Gọi `get_tree` với `cluster` cuối cùng của `path`.
2. Trả lời gồm ba phần:
    - **Kết luận về `subject`**: số người tìm được, thông tin khớp, thông tin không khớp hoặc thiếu dữ liệu, các thực thể không tìm thấy.
    - **Dẫn chứng**: kết quả về `subject` do tool trả về.
    - **`cluster tree`**: khối mermaid và cây dạng chữ do tool trả về, giữ nguyên, không sửa.
3. Nếu `path` đã đi hết các thực thể: trả lời xong thì dừng, không hỏi tìm tiếp.
4. Nếu `path` gặp `dead end` và còn `branch` chưa đi: nêu số `branch` chưa đi, nêu id của `branch` kế tiếp, và hỏi người dùng có muốn `backtrack` sang `branch` đó không. Dừng và chờ trả lời.

### Bước 5. `backtrack`

Chỉ thực hiện khi người dùng đồng ý ở Bước 4.4, hoặc tự yêu cầu đi một `branch` khác.

1. Gọi `next_branch`. Nếu người dùng chỉ định một `cluster` cụ thể thì truyền `cluster` đó vào `next_branch`.
2. Tool trả về `cluster` được chọn và các thực thể đã xét trên `path` tới `cluster` đó.
3. Quay lại Bước 3.3 với `cluster` này.



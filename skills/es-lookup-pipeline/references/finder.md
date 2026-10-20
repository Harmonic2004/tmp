# Dựng anchor và hint cho `search`

Mỗi lần gọi `search` tìm một thực thể thuộc một loại (`entityType`): `person`, `vehicle` hoặc `organization`. Thông tin dùng để tìm chia làm hai phần: `anchor` (bắt buộc khớp) và `hint` (khớp càng nhiều càng tốt, dùng để gom `cluster`).

## Anchor khi tìm root entity

- Root có thông tin nhóm 1 (`identifier`): mọi định danh là `anchor`.
- Root không có thông tin nhóm 1: tên (nhóm 2, `name`) là `anchor`. Xe không có nhóm `name`, nên xe chỉ làm root khi có định danh.

Khi tìm thực thể kế tiếp, không truyền `anchor`: tool tự lấy từ `cluster` liên kết.

## Hint

| Lần tìm | `hint` gồm |
| --- | --- |
| Root (mỗi ứng viên root ở Bước 2) | Mọi thông tin còn lại của root, và mọi thông tin của các thực thể có quan hệ trực tiếp với root mà câu hỏi nói rõ. |
| Thực thể kế tiếp | Mọi thông tin của thực thể đó, và mọi thông tin của các thực thể **chưa xét** có quan hệ trực tiếp với nó. |

- Quan hệ suy ra qua thực thể khác không phải quan hệ trực tiếp: chồng của mẹ chưa chắc là bố, nên khi tìm con thì không dùng thông tin của chồng người mẹ.
- Thực thể đã xét không dùng làm `hint`: `cluster` liên kết đã khớp thông tin của chúng.

Mỗi `hint` là một thông tin:

- `entity`: thực thể sở hữu thông tin, cùng chuỗi `entity` đã dùng khi chuẩn hoá.
- `relation`: quan hệ của `entity` với thực thể đang tìm, đọc theo khuôn "`entity` là `relation` của thực thể đang tìm". Bỏ trống khi là thông tin của chính thực thể đang tìm.
- `field`: field của **loại thực thể sở hữu thông tin**, theo skill `es-data-explorer`. Thông tin của mẹ dùng field của `person`; thông tin của trường học dùng field của `organization`.
- Giá trị: `value`, hoặc `values` (nhiều giá trị có thể, khớp một trong số đó), hoặc `from`/`to` cho thời gian, lấy từ kết quả chuẩn hoá.

Ví dụ, tìm Hưng học Khoa học Tự nhiên, có mẹ tên Hương:

```json
[
  { "entity": "Hưng", "field": "dateOfBirth", "from": "2004-01-01", "to": "2004-12-31" },
  { "entity": "Khoa học Tự nhiên", "relation": "school", "field": "companyName", "value": "Khoa học Tự nhiên" },
  { "entity": "Hương", "relation": "mother", "field": "fullName", "value": "Hương" }
]
```
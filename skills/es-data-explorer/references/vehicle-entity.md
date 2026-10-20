# Vehicle

Thực thể vehicle được lưu trong một index: `vehicle`, mỗi xe một doc. Tên field là field logic; tool tự ánh xạ sang field thật trong index `vehicle`.

**Bảng V1.** Thông tin của xe, theo nhóm ở Bảng 1

| Nhóm | `vehicle` |
| --- | --- |
| 1. `identifier` | `licencePlateNumber`, `vINNumber`, `engineNumber` |
| 2. `name` | |
| 3. `time` | `licensePlateIssueDate` (ngày cấp biển gần nhất), `firstLicensePlateIssueDate` (ngày cấp biển lần đầu) |
| 4. `location` | `address`: địa chỉ đăng ký xe |
| 5. `semantic` | |
| 6. `category` | `vehicleType`, `carBrand`, `color` |

**Bảng V2.** Thông tin về thực thể liên quan

| Quan hệ | `vehicle` | `netizen` (doc của người) |
| --- | --- | --- |
| `owner` | họ tên chủ xe (lưu ý rằng các bản ghi có thể có thêm năm sinh hoặc thông tin khác được ghép trong field này.) | |
| `identityId` | thông tin về CCCD của chủ xe (sử dụng để truy ngược thông tin chi tiết của chủ xe) | |
| `vehicle` | | biển số xe của người |

- Doc xe không có định danh của chủ xe. Liên kết chắc chắn giữa xe và người là biển số ghi trong doc `netizen` của người; liên kết qua họ tên và năm sinh của chủ xe có thể trùng người.
- Chủ xe là người đứng tên đăng ký, có thể không phải người đang dùng xe.

# Organization

Thực thể organization được lưu trong một index: `organization`. Tên field là field logic; tool tự ánh xạ sang field thật trong index `organization`.

**Bảng O1.** Thông tin của tổ chức, theo nhóm ở Bảng 1

| Nhóm | `organization` |
| --- | --- |
| 1. `identifier` | `companyCode` (mã doanh nghiệp), `phone` |
| 2. `name` | `companyName`, `companyForeignName` (tên nước ngoài) |
| 3. `time` | `registrationDate` (ngày đăng ký), `createdDate` (ngày tạo) |
| 4. `location` | `address` (địa chỉ đầy đủ), `provinceName`, `districtName`, `communeName` (từng cấp hành chính) |
| 5. `semantic` | |
| 6. `category` | `status` (trạng thái) |

**Bảng O2.** Thông tin về người liên quan

| Quan hệ | Field |
| --- | --- |
| Người đại diện theo pháp luật | `legalRepresentativeIdentity`, `legalRepresentativeFullName`, `legalRepresentativeDateOfBirth`, `legalRepresentativePlaceOfBirth`, `legalRepresentativeAddress`, `legalRepresentativePosition` (chức vụ) |
| Chủ sở hữu | `ownerId`, `ownerName` |
| Giám đốc | `directorName`, `directorPhone` |
| Kế toán | `accountantName`, `accountantPhone` |

- Người đại diện theo pháp luật và chủ sở hữu có định danh (`legalRepresentativeIdentity`, `ownerId`), dùng để nối tới doc của người. Giám đốc và kế toán chỉ có họ tên và SĐT.
- Tên tổ chức trong doc `insurance` của người không liên kết tới doc nào của index `organization` (xem mục Organization trong `SKILL.md`).

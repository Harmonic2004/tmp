---
name: es-data-explorer
description: Cách dữ liệu được lưu trên Elasticsearch, gồm mỗi loại thực thể (person, vehicle, organization) nằm ở index nào và mỗi index chứa thông tin, quan hệ nào. Dùng khi cần biết dữ liệu đang được tổ chức và lưu trữ như thế nào trên Elasticsearch.
---

# Elasticsearch Data Explorer

Dữ liệu cần tra cứu và phân tích gồm nhiều loại thực thể (`entity`), ví dụ person, organization, vehicle. Mỗi loại thực thể được lưu trữ trong một hoặc nhiều index trên Elasticsearch.

Thông tin của mỗi thực thể được chia thành 6 nhóm, trình bày ở [Bảng 1](#bang-1).

<a id="bang-1"></a>**Bảng 1.** Các nhóm thông tin

| Nhóm | Tên nhóm | Nội dung |
| --- | --- | --- |
| 1 | `identifier` | Định danh: CCCD (Căn cước công dân), CMND (Chứng minh nhân dân), BHYT (Bảo hiểm y tế), BHXH (Bảo hiểm xã hội), MST (Mã số thuế), SĐT (Số điện thoại), biển số xe, ... |
| 2 | `name` | Tên của thực thể |
| 3 | `time` | Thông tin thời gian: ngày sinh, ngày đăng ký biển số xe, ... |
| 4 | `location` | Thông tin địa điểm: quê quán, nơi ở, nơi thường trú, nơi sinh, ... |
| 5 | `semantic` | Thông tin mô tả dạng văn bản: nghề nghiệp, đặc điểm, ghi chú tự do, ... |
| 6 | `category` | Thông tin phân loại: giới tính, tình trạng hôn nhân, dân tộc, tôn giáo, loại xe, hãng xe, màu xe, ... |

### Mức quan trọng của thông tin

Với các nhóm `name`, `time` và `location`, thông tin càng đầy đủ và chi tiết thì càng giúp xác định đúng thực thể. Cụ thể:

- **`name`**: họ và tên đầy đủ có giá trị xác định cao hơn chỉ có tên.
- **`time`**: ngày, tháng, năm đầy đủ có giá trị cao hơn chỉ có một hoặc hai thành phần.
- **`location`**: địa chỉ đủ các cấp hành chính có giá trị cao hơn địa chỉ chỉ có một hoặc hai cấp.

## Thực thể và index

<a id="bang-2"></a>**Bảng 2.** Index của từng loại thực thể

| Thực thể | Index | Số doc của một thực thể | Vai trò | Chi tiết |
| --- | --- | --- | --- | --- |
| person | `netizen` | 1 | Trung tâm định danh: đủ mọi định danh của người, nối các index khác | `references/person-entity.md` |
| person | `citizen` | 1 | Hồ sơ dân cư: địa chỉ, thông tin phân loại, họ tên người thân | `references/person-entity.md` |
| person | `insurance` | nhiều | Bản ghi bảo hiểm: nơi đăng kí bảo hiểm (trường học, công ty) | `references/person-entity.md` |
| vehicle | `vehicle` | 1 | Đăng ký xe | `references/vehicle-entity.md` |
| organization | `organization` | | Thông tin của tổ chức: mã, tên, địa chỉ, người đại diện, chủ sở hữu | `references/organization-entity.md` |

Đọc file chi tiết của loại thực thể đang xét để biết từng index chứa thông tin nào và ghi theo cách nào.

## Liên kết giữa các index

- **Cùng một người**: các doc nối với nhau bằng định danh, CCCD là khoá chính. Doc `netizen` có đủ định danh nên là nơi đổi CMND, BHXH, BHYT, SĐT về CCCD. `citizen` có một doc cho mỗi người; `insurance` có nhiều doc cho mỗi người.
- **Người với người thân**: định danh của người thân ghi trong doc `netizen` và `citizen` của người; họ tên người thân chỉ có ở `citizen`.
- **Người với xe**: biển số xe ghi trong doc `netizen` của người. Doc xe ghi thông tin về họ tên và CCCD của chủ xe.
- **Tổ chức với người**: doc `organization` ghi định danh của người đại diện theo pháp luật và chủ sở hữu; giám đốc và kế toán chỉ có họ tên và SĐT.

## Organization

Tổ chức (trường học, công ty, cơ quan) được lưu ở index `organization` (chi tiết ở `references/organization-entity.md`).

Tên tổ chức còn xuất hiện trong doc `insurance` của người, ở nơi đăng kí bảo hiểm: `companyName`, `healthInsurancePayer`, `addressRegistrationSocialInsurance`. Nơi đăng kí có thể là trường học hoặc công ty hoặc một địa chỉ cụ thể. Hai nơi lưu này khác nhau:

| | index `organization` | nơi đăng kí bảo hiểm trong `insurance` |
| --- | --- | --- |
| Là gì | Doc của chính tổ chức | Thông tin của người |
| Nội dung | Tên và các thông tin khác của tổ chức | Chỉ có tên |
| Liên kết | | Không liên kết tới doc nào của index `organization` |

- Tên trong `insurance` do đơn vị khai, thường là tên đầy đủ (ví dụ "Công ty trách nhiệm hữu hạn công nghiệp thực phẩm Liwayway Hà Nội"), có thể khác cách người dùng gọi.
- Mỗi doc `insurance` là một bản ghi tại một thời điểm: nơi đăng kí trong doc là nơi người đó đã làm việc hoặc học, chưa chắc là hiện tại.

# Person

Thực thể person được lưu trong ba index: `netizen`, `citizen` và `insurance`. Tên field là field logic; tool tự ánh xạ sang field thật trong từng index.

## So sánh ba index

| | `netizen` | `citizen` | `insurance` |
| --- | --- | --- | --- |
| Số doc của một người | 1 | 1 | nhiều |
| Vai trò | Trung tâm định danh | Hồ sơ dân cư | Bản ghi bảo hiểm |

**Bảng P1.** Thông tin của chính người, theo nhóm ở Bảng 1

| Nhóm | `netizen` | `citizen` | `insurance` |
| --- | --- | --- | --- |
| 1. `identifier` | `identityId`, `nationalId`, `socialInsuranceNumber`, `healthInsuranceNumber`, `taxCode`, `phone`, `licencePlateNumber` | `identityId`, `nationalId` | `identityId`, `socialInsuranceNumber`, `healthInsuranceNumber`, `phone` |
| 2. `name` | `fullName` | `fullName` | `fullName` |
| 3. `time` | `dateOfBirth` | `dateOfBirth` | `dateOfBirth` |
| 4. `location` | | `address`: nơi ở hiện tại<br>`hometown`: quê quán<br>`residential`: địa chỉ thường trú<br>`newAddress`: nơi ở hiện tại theo địa giới mới<br>`newHometown`: quê quán theo địa giới mới<br>`newResidential`: địa chỉ thường trú theo địa gới mới | `companyName`, `healthInsurancePayer`, `addressRegistrationSocialInsurance`: nơi đăng kí bảo hiểm<br>`address`: nơi ở tại thời điểm của bản ghi bảo hiểm |
| 5. `semantic` | | | `socialInsuranceNote` |
| 6. `category` | | `gender`, `maritalStatus`, `ethnicity`, `religion`, `houseHolderRelation` | `houseHolderRelation` |

**Bảng P2.** Thông tin về thực thể liên quan có trong doc của mỗi người

| Quan hệ | `netizen` | `citizen` | `insurance` |
| --- | --- | --- | --- |
| `father`, `mother`, `couple`, `child`, `houseHolder` | `identityId`, `nationalId` | `fullName`, `identityId`, `nationalId` | `socialInsuranceNote`: có thể ghi thông tin bất kỳ về người thân (nếu có) |
| `vehicle` | `licencePlateNumber` | | |
| `organization` | | | `companyName`, `healthInsurancePayer`, `addressRegistrationSocialInsurance`: tên nơi đăng kí bảo hiểm, không liên kết tới index `organization` |

## `netizen`

- Mỗi người một doc, có đủ định danh. Một doc có thể ghi nhiều SĐT.
- Không có địa chỉ và thông tin phân loại.
- Người thân chỉ có định danh, không có họ tên.

## `citizen`

- Mỗi người một doc, nối với `netizen` qua CCCD hoặc CMND. Định danh chỉ có CCCD và CMND.
- Địa chỉ tách theo loại (quê quán, thường trú, nơi ở hiện tại, nơi sinh), mỗi loại ghi theo cấp hành chính (xã, huyện, tỉnh) và một dòng chi tiết.
- Địa chỉ ghi theo địa giới cũ, và có thêm bản theo địa giới mới sau sắp xếp năm 2025 (không còn cấp huyện, nhiều xã đổi tên) cho nơi ở hiện tại (`newAddress`) và quê quán (`newHometown`). Địa danh mới chỉ khớp được ở `citizen`.
- Người thân có họ tên; CCCD, CMND của người thân có thể thiếu. Một người có thể có nhiều `child`.

## `insurance`

- Một người có nhiều doc, mỗi doc là một bản ghi bảo hiểm tại một thời điểm và chỉ ghi một phần field; các doc của cùng một người ghi field khác nhau.
- Trong index `insurance` có thể chứa những thông tin `semantic` liên quan đến người hoặc người thân của người như nghề nghiệp, mô tả, ...
- Nơi đăng kí bảo hiểm (trường học, công ty) chỉ được ghi dưới dạng tên (xem mục Organization trong `SKILL.md`).

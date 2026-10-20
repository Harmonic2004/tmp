# Chuẩn hoá thông tin đầu vào

Mọi giá trị định danh, giới tính, thời gian của mọi thực thể trong câu hỏi đều được chuẩn hoá bằng tool `normalize_input`, gửi chung trong một lần gọi. Mỗi giá trị là một item:

- `entity`: cách câu hỏi gọi thực thể sở hữu giá trị, giữ nguyên văn (`Thắng`, `mẹ Thắng`, `công ty ABC`, `xe 38H12345`); hai thực thể trùng cách gọi thì thêm vai trò (`Hương (mẹ)`, `Hương (vợ)`). Mỗi thực thể dùng một `entity` duy nhất; tool kiểm tra chéo các item cùng `entity`.
- `kind` theo dạng của giá trị, không theo thực thể sở hữu: mọi SĐT là `phone`, mọi MST là `taxCode`, mọi mốc thời gian là `time`.
- Định danh, giới tính: `value` nguyên văn, `kind` theo [Định danh](#định-danh).
- Thời gian: `kind: "time"`, `role` và `expr` theo [Thời gian](#thời-gian).

## Định danh

| Người dùng gọi                                                                    | `kind`                  |
| --------------------------------------------------------------------------------- | ----------------------- |
| CCCD, căn cước, căn cước công dân, số định danh cá nhân, mã định danh, CMND 12 số | `identityId`            |
| CMND, chứng minh thư, CMT                                                         | `nationalId`            |
| SĐT, số điện thoại, di động, máy bàn, số Zalo                                     | `phone`                 |
| mã BHXH, số sổ BHXH                                                               | `socialInsuranceNumber` |
| mã thẻ BHYT, số thẻ bảo hiểm y tế                                                 | `healthInsuranceNumber` |
| MST, mã số thuế                                                                   | `taxCode`               |
| biển số, BKS, biển kiểm soát                                                      | `licencePlateNumber`    |
| số khung, VIN                                                                     | `vINNumber`             |
| số máy, số động cơ (của xe)                                                       | `engineNumber`          |
| nam, nữ, giới tính                                                                | `gender`                |

Gửi `kind` theo nhãn người dùng dùng, kể cả khi nhãn sai loại (CMND 12 số, mã BHYT 10 số); tool tự nhận và trả `kind` đúng.

Khi nhãn không rõ:

- "Số máy" đi với xe là `engineNumber`; đi với người là `phone`.
- Dãy số không có nhãn: 12 chữ số → `identityId`; 9 chữ số → `nationalId`; 10 chữ số bắt đầu bằng `0` → `phone`; 10 chữ số còn lại → hỏi người dùng (BHXH hay MST).
- 2 chữ cái + 13 chữ số → `healthInsuranceNumber`; 2 chữ số + chữ cái + 4–6 ký tự → `licencePlateNumber`; 17 ký tự chữ và số → `vINNumber`.

## Thời gian

`expr` giữ đúng con số người dùng nói; tool tự tính theo ngày hôm nay, tự áp dụng tuổi mụ và quy tắc năm 2 chữ số. Năm 2 chữ số truyền dạng chuỗi (`"94"`).

| Người dùng nói                        | `expr`                                                                                    |
| ------------------------------------- | ----------------------------------------------------------------------------------------- |
| `19/11/1994`, `19-11-94`              | `{ "type": "date", "day": 19, "month": 11, "year": 1994 }` (năm `"94"` nếu viết 2 chữ số) |
| `11/1994`, `tháng 11 năm 1994`        | `{ "type": "month", "month": 11, "year": 1994 }`                                          |
| `sinh năm 2002`, `năm 94`             | `{ "type": "year", "year": 2002 }`                                                        |
| `từ 1990 đến 1995`                    | `{ "type": "yearRange", "fromYear": 1990, "toYear": 1995 }`                               |
| `24 tuổi`                             | `{ "type": "ageRange", "min": 24, "max": 24 }`                                            |
| `20–25 tuổi`                          | `{ "type": "ageRange", "min": 20, "max": 25 }`                                            |
| `khoảng / tầm / chừng T tuổi`         | `ageRange`, `min = T − 2`, `max = T + 2`                                                  |
| `ngoài / hơn / trên T` (T tròn chục)  | `ageRange`, `min = T + 1`, `max = T + 9`                                                  |
| `gần T`, `U T` (T tròn chục)          | `ageRange`, `min = T − 5`, `max = T − 1`                                                  |
| `8x`, `thế hệ 9x`                     | `{ "type": "decade", "decade": 8 }`                                                       |
| `đầu / giữa / cuối 9x`                | `{ "type": "decade", "decade": 9, "part": "early" \| "mid" \| "late" }`                   |
| `năm nay`, `năm ngoái`, `3 năm trước` | `{ "type": "yearsAgo", "n": 0 \| 1 \| 3 }`                                                |

| Ngữ cảnh                          | `role`                                  |
| --------------------------------- | --------------------------------------- |
| sinh, tuổi, thế hệ                | `dateOfBirth` (mặc định)                |
| biển số cấp, đăng ký xe           | `licensePlateIssueDate`                 |
| đăng ký xe lần đầu                | `firstLicensePlateIssueDate`            |
| thẻ bảo hiểm có hiệu lực từ / đến | `insuranceFromDate` / `insuranceToDate` |

## Dùng kết quả

- Kết quả cần người dùng xác nhận (giá trị sai, nhiều cách hiểu, mâu thuẫn trong cùng thực thể): gom mọi giá trị vào một câu hỏi, mỗi giá trị nêu giá trị gốc, lý do và cách đọc đề xuất của tool. Khi có trả lời, chuẩn hoá lại các giá trị đã sửa.
- Ghi bảng `gốc → chuẩn hoá` cho mọi giá trị bị đổi; bảng này dùng lại ở câu trả lời cuối.

**Xong khi**: mọi giá trị đã chuẩn hoá, không còn giá trị chờ người dùng xác nhận, và bảng `gốc → chuẩn hoá` đã ghi.

## Thông tin dạng chữ

Tên người, địa chỉ, tổ chức, ghi chú không qua tool: giữ nguyên văn, có dấu, viết đầy đủ từ thay cho chữ viết tắt ("cty" → "công ty", "ĐH" → "đại học", "TP" → "thành phố") vì tìm kiếm so khớp theo từ.

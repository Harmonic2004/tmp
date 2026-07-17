# Capabilities 

## CAP-01. Tìm thông tin về người

### Mô tả

Mục tiêu của capability này sẽ lấy ra các thông tin liên quan đến một người. Các thông tin liên quan đến một người bao gồm:
- Thông tin công khai, xuất hiện trong tài liệu/bài báo công khai
- Thông tin định danh và nhân khẩu học (Netizen và Citizen)

Ví dụ với câu hỏi: "Tìm Nguyễn Văn A quê ở Hưng Yên và đi xe Honda màu đỏ."

Với câu hỏi trên thì kỳ vọng capability trả về:
- Thông tin công khai của Nguyễn Văn A
- Thông tin định danh (CCCD, SĐT, ...) của Nguyễn Văn A (netizen)
- Thông tin nhân khẩu học (Giới tính, Tình trạng hôn nhân, quê quán, nơi ở, ...) của Nguyễn Văn A

### Plan 1

```mermaid
flowchart TD
    s1["input_process()<br/>in: raw_query: str<br/>out: Extracted"] --> d1{Router}

    d1 -->|Only 1 Input| d2{Input Type}
    d2 -->|name| s2["search_es_person()"]
    s2 --> s3["resolve_person()"]
    s3 --> d3{"len(output)"}
    d3 -->|"len(output) > output_limit"| CLARIFY["Trả về một lượng output và làm rõ với người dùng"]
    CLARIFY --> s4
    d3 -->|"else"| s4["Bridge Person/Person_Official --> Netizen and Citizen"]
    s4 --> validator
    validator --> END

    d2 -->|keyword| d_2{"keyword_classifier()"}
    d_2 -->|"Type keyword is Person"| s_3["search_es_person() <br/> search theo field text"]
    s_3 --> s_4["resolve_person()"]
    s_4 --> d_3{"Có các loại keyword khác không? [Vehicle, Company, Facebook]"} 
    d_3 -->|Không| s4
    d_3 -->|Có| s_5["Bridge Person/Person_Official --> Netizen và expand Netizen sang các loại keyword để filter theo các keyword"]
    s_5 --> validator

    d2 -->|identifier| s__2["search_es_netizen()"]
    s__2 --> d__2{"len(output)"}
    d__2 -->|"len(output) > output_limit"| s__3["Hỏi lại người dùng"]
    d__2 -->|"else"| s__4["Bridge Netizen --> Person/Person_Official"]
    s__3 --> s__4
    s__4 --> validator

    d1 -->|> 1 Input| D2{"check_input()"}
    D2 -->|"HAS_Identify"| S2["search_es_netizen()"]
    S2 --> D3{"Có các keyword in [Citizen, Vehicle, Company, Facebook]"}
    D3 -->|"Có"| S3["expand_neo4j() <br/> so sánh các keyword xem đúng chưa?"]
    S3 --> D4{"Có keyword thuộc Person?"}
    D4 -->|"Có"| S4["Bridge_netizen_person() và so sánh các keyword từ person"]
    S4 --> validator

    D2 -->|"NO_Identify"| D_2{"check_input()"}
    D_2 -->|"HAS_NAME"| S_2["search_es_person()"]
    S_2 --> S_3["resolve_person()"]
    S_3 --> D_3{"Có các loại keyword khác không? [Vehicle, Company, Facebook]"} 
    D_3 -->|Không| S_4["Bridge Person/Person_Official --> Netizen and Citizen"]
    D_3 -->|Có| S_5["Bridge Person/Person_Official --> Netizen và expand Netizen sang các loại keyword để filter theo các keyword"]
    S_5 --> validator
    S_4 --> validator
```

## CAP-02. Tra cứu thông tin về phương tiện (Vehicle)

### Mô tả

Mục tiêu của capability này sẽ trả lời các dạng câu hỏi sau:
- Tra cứu thông tin của xe dựa vào biển số xe hoặc các keyword liên quan đến xe

### Plan 2

Tra cứu thông tin của xe dựa vào biển số xe hoặc các keyword liên quan đến xe

```mermaid
flowchart TD
    s1["input_process()<br/>in: raw_query: str<br/>out: Extracted"] --> d1{Router}

    d1 -->|Only 1 Input| d2{"input_type()"}
    d2 -->|"Biển số xe"| s2["search_es_vehicle()"]
    s2 --> validator
    validator --> END

    d2 -->|keyword| s_3["search_es_vehicle()"]
    s_3 --> validator

    d1 -->|> 1 Input| combine_search("Combine Search")
    combine_search --> S_combine["search_es_vehicle() <br/> search các input theo field text"]
    S_combine --> d_search{"Output (Combine Search)"}
    d_search -->|"Có kết quả"| validator
    d_search -->|"Rỗng"| separate_search("Separate Search")
    separate_search --> D2{"check_input()"}
    D2 -->|"Có biển số xe"| S2["search_es_vehicle()"]
    S2 --> D3{"Có các keyword?"}
    D3 -->|"Có"| S3["So sánh các keyword"]
    S3 --> validator
```

## CAP-03. Tra cứu thông tin về Company

### Mô tả
Mục tiêu của capability này sẽ trả lời các dạng câu hỏi sau:
- Tra cứu thông tin của Company dựa vào mã số thuế, sđt và các keyword liên quan đến Company
- Tra cứu thông tin người đại diện công ty dựa vào mã số thuế, sđt và các keyword liên quan đến company

### Plan 3
Tra cứu thông tin của Company dựa vào mã số thuế và các keyword liên quan đến Company

```mermaid
flowchart TD
    s1["input_process()<br/>in: raw_query: str<br/>out: Extracted"] --> d1{Router}

    d1 -->|Only 1 Input| d2{"input_type()"}
    d2 -->|"Mã số thuế"| s2["search_es_company()"]
    s2 --> validator
    validator --> END

    d2 -->|keyword| s_3["search_es_company()"]
    s_3 --> validator

    d2 -->|sđt| s_4["expand_phone_to_company()"]
    s_4 --> s_5["es_search_company()"]
    s_5 --> validator

    d1 -->|> 1 Input| combine_search("Combine Search")
    combine_search --> S_combine["search_es_company() <br/> search các input theo field text"]
    S_combine --> d_search{"Output (Combine Search)"}
    d_search -->|"Có kết quả"| d1_{"check_input()"}
    d1_ -->|"Có Sđt"| s_check["expand_company_to_phone() <br/> expand kết hợp lọc/đối chiếu sđt từ input"]
    s_check --> validator
    d_search -->|"Rỗng"| separate_search("Separate Search")
    separate_search --> D2{"check_input()"}
    D2 -->|"Có MST"| S2["search_es_company()"]
    S2 --> D3{"Có các keyword"}
    D3 -->|"Có"| S3["So sánh các keyword"]
    S3 --> S4["expand_company_to_phone() <br/> expand kết hợp lọc/đối chiếu sđt từ input"]
    S4 --> validator

```

## CAP-04. Tra cứu thông tin về facebook

Mục tiêu của capability này sẽ trả lời các dạng câu hỏi sau:
- Tra cứu thông tin về facebook sử dụng tên fb, sđt hoặc id (hiếm gặp)
- Tra cứu thông tin của chủ fb dựa vào thông tin fb.

### Plan 4

Tra cứu thông tin về facebook sử dụng tên fb, sđt hoặc id (hiếm gặp)

```mermaid
flowchart TD
    s1["input_process()<br/>in: raw_query: str<br/>out: Extracted"] --> d1{Router}

    d1 -->|Only 1 Input| d2{"input_type()"}
    d2 -->|"fb_Id | Sđt"| s2["search_es_facebook()"]
    s2 --> validator
    validator --> END

    d2 -->|keyword| s_3["search_es_facebook()"]
    s_3 --> validator

    d1 -->|> 1 Input| combine_search("Combine Search")
    combine_search --> S_combine["search_es_facebook() <br/> search các input theo field text"]
    S_combine --> d_search{"Output (Combine Search)"}
    d_search -->|"Có kết quả"| validator
    d_search -->|"Rỗng"| separate_search("Separate Search")
    separate_search --> D2{"check_input()"}
    D2 -->|"Có fb_Id | Sđt"| S2["search_es_facekook()"]
    S2 --> D3{"Có các keyword"}
    D3 -->|"Có"| S3["So sánh các keyword"]
    S3 --> validator
```

## CAP-05. Tra cứu thông tin về tổ chức/sự kiện công khai

### Mô tả
Mục tiêu của capability này là sẽ tra cứu các thông tin về tổ chức hoặc sự kiện sử dụng dữ liệu cung cấp từ câu hỏi người dùng.

### Plan 5

```mermaid
flowchart TD
    s1["input_process()<br/>in: raw_query: str<br/>out: Extracted"] --> s2["es_search_organization_or_event()"]
    s2 --> s3["resolve_entity() <br/> Gom event | organization về *_Officail"]
    s3 --> validator
    validator --> END
```

## CAP-06. Tra cứu thông tin về Bài báo/Tài liệu dựa vào chủ đề, mô tả

### Mô tả 

Mục tiêu của Capability này là tra cứu thông tin về bài báo hoặc tài liệu liên quan đến mô tả hoặc chủ đề được người dùng cung cấp

### Plan 6

```mermaid
flowchart TD
    s1["input_process()<br/>in: raw_query: str<br/>out: Extracted"] --> d1{Router}

    d1 -->|"Các dạng đầu vào từ bên ngoài (link bài báo)"| s2["LLM_helper() <br/> Sử dụng LLM để lấy ra mô tả hoặc chủ đề"]
    s2 --> s3["search_es_document()"]
    s3 --> d2{"len(output)"}
    d2 -->|else| s4["re_rank() <br/> Chọn ra bài báo khớp với đầu vào và thời điểm gần đây hoặc khoảng thời gian trong input"]
    s4 --> validator
    d2 -->|<= output_limit| validator
    validator --> END

    d1 -->|"Mô tả | Chủ đề"| s_2["search_es_document()"]
    s_2 --> d2
```

### CAP-07. Tài liệu | Bài báo đề cập đến những ai, tổ chức, sự kiện nào?

### Mô tả

Mục tiêu của capability này là đi tìm thông tin các thực thể như person, organization và evnet được đề cập trong một bài báo cụ thể.

### Plan 7

```mermaid
flowchart TD
    s1["input_process()<br/>in: raw_query: str<br/>out: Extracted"] --> plan[Plan 6]
    plan --> d1{"Mention entity type"}
    d1 -->|"person"| s2["Bridge document --> Person"]
    d1 -->|"event"| s3["Bridge document --> Event"]
    d1 -->|"organization"| s4["Bridge document --> organization"]
    s2 --> plan_1["Plan 1"]
    s3 --> plan_5["Plan 5"]
    s4 --> plan_5["Plan 5"]
```

## CAP-08. Tài sản liên kết với một người

### Mô tả

Người dùng cung cấp tên, identifier, keyword về người và hỏi về phương tiện, số điện thoại, company hoặc fb của người đó

### Plan 8

```mermaid
flowchart TD
    s1["input_process()<br/>in: raw_query: str<br/>out: Extracted"] --> plan_1[Plan 1]
    plan_1 --> d1{"asset type"}
    d1 -->|"Vehicle"| s2["Bridge Netizen --> Vehicle"]
    s2 --> plan_2.1["Plan 2.1"]
    plan_2.1 --> validator

    d1 -->|"Company"| s3["Bridge Netizen --> Company"]
    d1 -->|"Facebook"| s4["Bridge Netizen --> Facebook"]
    s3 --> plan_3.1["Plan 3.1"]
    s4 --> plan_4.1["Plan 4.1"]
    plan_3.1 --> validator
    plan_4.1 --> validator
    validator --> END
```

## CAP-09. Quan hệ gia đình

### Mô tả 

Người dùng cung cấp name, keyword hoặc identify về người và hỏi về quan hệ gia đình của người đó

### Plan 9

```mermaid
flowchart TD
    s1["input_process()<br/>in: raw_query: str<br/>out: Extracted"] --> plan_1[Plan 1]
    plan_1 --> s2["Expand Netizen --> Netizen <br/> expand đúng quan hệ gia đình, cha mẹ, vợ chồng, con, chủ hộ, ..."]
    s2 -->  validator
    validator --> END
```

## CAP-10. Tra cứu người hoặc số điện thoại hay liên lạc với ai, số điện thoại nào?

### Mô tả

Mục tiêu của capability này là tìm một người hay liên lạc với các người khác và số điện thoại hay liên lạc với số điện thoại khác.

### Plan 10

```mermaid
flowchart TD
    s1["input_process()<br/>in: raw_query: str<br/>out: Extracted"] --> d1{"Router"}
    d1 -->|"person"| plan_1["Plan 1"]
    d1 -->|"Sđt"| s2["expand_phone_to_phone()"]
    plan_1 --> s3["expand_phone_to_phone()"]
    s3 --> plan__1["Plan 1"]
    plan__1 --> validator
    s2 --> validator
    validator --> END
   
```


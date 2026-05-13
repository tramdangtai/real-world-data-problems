---
date: 2026-05-13
---
# 📊 Dynamic Latest File Retrieval from SharePoint Folders (Power Query)

## 📌 Overview

Trong quá trình làm việc, mình gặp một bài toán liên quan đến dữ liệu inventory được lưu trên SharePoint:

> Cần tự động đi vào đúng Company Folder + Current Year Folder, lấy latest file theo date trong filename, rồi combine dữ liệu lại.

Ý tưởng cốt lõi:

- Dữ liệu luôn là dữ liệu mới nhất
- Không cần chỉnh tay mỗi tháng
- Query có thể tự chạy theo thời gian thực

---

## 🧩 Problem

Cấu trúc folder:

```text
item_inventory_by_store/{Company}/{CurrentYear}
```

Ví dụ:

```text
item_inventory_by_store/Company_A/2026
```

---

### Format file

```text
Items by Location Matrix_04_13_2026_AHCL_HO_LIVE.xlsx
```

---

### Yêu cầu

- Dynamic Current Year
- Dynamic Company Folder
- Lấy latest file theo date trong filename
- Import Excel
- Combine dữ liệu giữa nhiều company

---

## ⚠️ Constraints

- Dữ liệu nằm trên SharePoint
- Dùng:
```m
SharePoint.Contents
```

- Không được load toàn bộ file
- Cần tối ưu performance
- Code phải dễ đọc và dễ maintain
- Chỉ lấy đúng file Excel

---

## 🧠 Thinking Process

Trước đó mình đã làm 2 case:

- Dynamic File Filtering by Year-Month [Link](https://github.com/tramdangtai/real-world-data-problems/tree/main/02-power-query/dynamic-file-filtering-by-year-month)
- Optimizing Year-Based File Filtering in Power Query [Link](https://github.com/tramdangtai/real-world-data-problems/tree/main/02-power-query/optimizing-year-based-file-filtering-in-power-query)

Nên mình biết chắc:

> sẽ có cách tối ưu hơn việc transform quá nhiều trên Table.

---

## ❌ Old Approach

Flow cũ của mình:

### Với từng company:

- Đi vào folder
- Lấy current year
- Extract date từ filename:
  - `Text.AfterDelimiter`
  - `Text.BeforeDelimiter`
- Convert text → date
- Lấy latest
- Import Excel

---

Sau đó:

- Làm lại toàn bộ cho company khác
- `Table.Combine`

---

### ❗ Vấn đề

- Logic bị lặp
- Nhiều step
- Query dài
- Khó scale nếu thêm company mới

---

## 💡 Solution

Mình chia bài toán thành 2 phần:

---

# Part 1 — Function lấy latest file

Tạo function riêng:

```text
fn_get_latest_excel_file
```

---

## Function

```m
(source as table) as table =>
let
    AddSortKey =
        Table.AddColumn(
            source,
            "SortKey",
            each
                let
                    p = Text.Split([Name], "_")
                in
                    p{3} & p{1} & p{2},
            type text
        ),

    MaxKey = List.Max(AddSortKey[SortKey]),

    LatestFile =
        Table.SelectRows(
            AddSortKey,
            each [SortKey] = MaxKey
        ){0}[Content],

    ImportedExcelWorkbook =
        Excel.Workbook(LatestFile)

in
    ImportedExcelWorkbook
```

---

## 🔍 Logic Breakdown

### Tách filename

```m
p = Text.Split([Name], "_")
```

Ví dụ:

```text
Items by Location Matrix_04_13_2026_AHCL_HO_LIVE.xlsx
```

→ sẽ thành list

---

### Tạo SortKey

```m
p{3} & p{1} & p{2}
```

→ convert thành:

```text
20260413
```

---

### Vì sao dùng text thay vì date?

Mình nhận ra:

> Mục tiêu không phải tạo date column thật sự.

Mục tiêu chỉ là:

```text
lấy latest row
```

Và format:

```text
YYYYMMDD
```

đã đủ để:

```m
List.Max
```

hoạt động chính xác.

---

### Lấy latest file

```m
List.Max(AddSortKey[SortKey])
```

→ lấy giá trị lớn nhất

Sau đó:

```m
Table.SelectRows(...)
```

→ filter đúng latest file

---

# Part 2 — Dynamic Company Loop

Sau khi xử lý latest file xong:

Mình bắt đầu nghĩ:

> Giá như M Code có loop giống Python.

Và đây là lúc mình bắt đầu thật sự hiểu rõ hơn về:

```m
List.Transform
```

---

## Companies List

```json
{
    "Company_AHCL",
    "Company_VHS"
}
```

---

## Function xử lý từng company

```m
fn_GetCompanyInventory =
    (company as text) =>
    let
        CompanyFolder  =
            item_inventory_by_store{[Name=company]}[Content],

        YearFolder =
            CompanyFolder{[Name=CurrentYear]}[Content],

        FilteredFiles =
            Table.SelectRows(
                YearFolder,
                each
                    Text.StartsWith(
                        [Name],
                        "Items by Location Matrix_" & CurrentMonth & "_"
                    )
                    and Text.EndsWith([Name], ".xlsx")
            ),

        LatestFile =
            fn_get_latest_excel_file(FilteredFiles)

    in
        LatestFile
```

---

## Loop qua Companies

```m
ImportedFiles =
    List.Transform(
        Companies,
        each fn_GetCompanyInventory(_)
    )
```

---

## 🧠 Ý tưởng tổng thể

```text
Loop qua từng company

→ Đi vào folder năm hiện tại
→ Filter đúng file cần dùng
→ Lấy latest file
→ Import Excel

Cuối cùng:
combine toàn bộ dữ liệu
```

---

## ⚙️ Optimization

---

### 1. Chỉ lấy file Excel

```m
Text.EndsWith([Name], ".xlsx")
```

→ tránh:
- txt
- folder
- file lỗi
- file upload nhầm

---

### 2. Chỉ filter current month

```m
Text.StartsWith(
    [Name],
    "Items by Location Matrix_" & CurrentMonth & "_"
)
```

→ giảm số lượng row cần check

---

### 3. Dynamic CurrentMonth

```m
Text.PadStart(
    Text.From(Date.Month(DateTime.LocalNow())),
    2,
    "0"
)
```

---

### 💡 Insight

`Text.PadStart` cực hữu ích:

```text
4  → 04
```

→ đảm bảo match đúng format filename.

---

### 4. Tách function riêng

```text
fn_get_latest_excel_file
```

→ code:
- dễ đọc hơn
- dễ maintain hơn
- reusable hơn

---

## 🤖 Role of AI

Case này mình dùng AI khá nhiều để:

- brainstorm pattern
- validate logic
- mở rộng góc nhìn

Nhưng phần quan trọng nhất là:

- tự test
- tự benchmark
- tự verify output

Đặc biệt:

> Case này là lần đầu mình thật sự “cảm” được `List.Transform` như một vòng lặp trong M Code.

---

## 🧠 Key Takeaways

### 1. List.Transform thật sự là “loop”

Trước đây mình biết syntax.

Nhưng đến case này mình mới:
- hiểu rõ mindset của nó
- bắt đầu nghĩ theo hướng functional hơn

---

### 2. Không phải lúc nào cũng cần convert sang date

Nếu mục tiêu chỉ là sorting:

```text
YYYYMMDD (text)
```

đã đủ dùng.

---

### 3. Tách function giúp giảm complexity rất nhiều

Thay vì:
- nested logic
- repeated steps

→ chia đúng responsibility:
- function xử lý latest file
- function xử lý company
- query chính combine dữ liệu

---

### 4. Performance = giảm số row cần xử lý

Case này tiếp tục củng cố cho mình:

> “Performance in Power Query is not about fewer lines of code — it's about fewer rows touched.”

---

## 📁 SharePoint Folder Structure  
  
```text  
SharePoint Site

Projects/  
└── data/  
	├── Company_A
		├── 2026
			├── list inv file.xlsx
	├── Company_B
		├── 2026
			├── list inv file.xlsx
```

---

## 📁 Files

- [data](https://github.com/tramdangtai/real-world-data-problems/tree/main/02-power-query/dynamic-latest-file-sharepoint-power-query/data)
- [solution](https://github.com/tramdangtai/real-world-data-problems/tree/main/02-power-query/dynamic-latest-file-sharepoint-power-query/solution)

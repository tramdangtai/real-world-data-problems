---
date: 2026-05-09
---
# 📊 Optimizing Year-Based File Filtering in Power Query

## 📌 Overview

Trong quá trình build report, mình gặp một bài toán nhỏ nhưng xuất hiện khá thường xuyên:

> Chỉ load các file sales data của x năm gần nhất thay vì load toàn bộ dữ liệu.

Dataset đang được lưu dạng nhiều file `.csv`, mỗi file tương ứng với dữ liệu của một năm.

Ví dụ:

```text
fact_sales_yearly_level_sku_store_[year].csv
```

Mục tiêu:

- Không load toàn bộ historical data
- Chỉ load những năm cần thiết
- Tối ưu performance cho report
- Và giữ code dễ maintain

---

## 🧩 Problem

Cách làm ban đầu của mình khá nhiều step.

Flow cũ:

1. Tạo biến lấy current year
2. Dùng `Table.TransformColumns`
3. Extract year từ file name:
   - `Text.AfterDelimiter`
   - `Text.BeforeDelimiter`
4. Dùng `Table.TransformColumnTypes`
   - đổi type sang number
5. Dùng `Table.SelectRows`
   - filter latest years

Ví dụ:

```m
each [Name] >= param_value_currentYear - 3
```

---

### ❗ Vấn đề

Case này không khó.

Nhưng mình cảm thấy:

- Quá nhiều step
- Query bị dài
- Logic extract year hơi “nặng”
- Và đang xử lý trên `Table` nhiều hơn mức cần thiết

---

## 🧠 Thinking Process

Trước đó mình từng làm project:

```text
dynamic-file-filtering-by-year-month
```

[Link](https://github.com/tramdangtai/real-world-data-problems/tree/main/02-power-query/dynamic-file-filtering-by-year-month)

Trong đó mình có dùng pattern:

```m
List.AnyTrue
List.Transform
Table.SelectRows
Text.Contains
```

để filter file theo format:

```text
YYYY_MM
```

---

Lúc nhìn lại case này, mình nhận ra:

> Thực ra mình không cần extract year ra nữa.

Mình chỉ cần:

```text
check xem tên file có chứa year cần lấy hay không
```

---

## ⚠️ Constraints

- Không thay đổi data source
- Đảm bảo output giống hoàn toàn cách cũ
- Không ảnh hưởng các step phía sau
- Code phải dễ maintain

---

## 💡 Solution

### 1. Tạo parameter chứa list các năm cần dùng

```m
let
    CurrentYear = Date.Year(DateTime.LocalNow()),
    LastThreeYears = {CurrentYear - 3 .. CurrentYear},
    YearTextList = List.Transform(LastThreeYears, each Text.From(_))
in
    YearTextList
```

---

### Ý tưởng

Tạo sẵn:

```text
{"2023","2024","2025","2026"}
```

---

### Vì sao phải convert sang text?

Vì:

```text
Tên file = text
```

và:

```m
Text.Contains
```

chỉ hoạt động với text.

---

## 2. Filter trực tiếp bằng List Logic

```m
Table.SelectRows(  
    Source,   
    each List.AnyTrue(  
        List.Transform(
            param_list_lastFourYears_w_textFormat, 
            (m) => Text.Contains([Name], m)
        )  
    )  
)
```

---

## 🔍 Logic Breakdown

### `List.Transform`

Loop từng item trong list year:

```text
2023
2024
2025
2026
```

---

### `Text.Contains`

Check:

```text
Tên file có chứa year đó không?
```

Ví dụ:

```text
fact_sales_yearly_level_sku_store_2026.csv
```

---

### `List.AnyTrue`

Nếu chỉ cần:

```text
1 year match
```

→ return TRUE

---

### `Table.SelectRows`

Giữ lại các row có:

```text
TRUE
```

---

## 🧠 Ý tưởng tổng thể

```text
Loop qua danh sách các năm cần dùng

Nếu tên file chứa ít nhất 1 năm:
    giữ lại
Ngược lại:
    loại bỏ
```

---

## ⚙️ Optimization

### 1. Chuyển từ Table logic → List logic

Trước đó:

- Extract year từ text
- Convert type
- Filter number

Hiện tại:

- Chỉ check text trực tiếp

---

### 2. Dùng List thay vì Table.TransformColumns

Ở case này:

```text
List nhẹ hơn Table
```

vì:

- Không cần transform cả column
- Không cần tạo thêm intermediate step
- Chỉ cần evaluate điều kiện

---

### 3. Tách parameter riêng

```text
param_list_lastFourYears_w_textFormat
```

→ dễ:
- maintain
- reuse
- thay đổi logic sau này

Ví dụ:

```text
latest 2 years
latest 5 years
```

chỉ cần đổi parameter.

---

## ⚡ Result

Từ:

- nhiều step
- nhiều transform
- extract + type conversion

→ giảm xuống chỉ còn:

- 1 parameter
- 1 filter step

---

## 🧠 Key Takeaways

### 1. Không phải lúc nào cũng cần extract dữ liệu

Ban đầu mình nghĩ:

```text
Muốn filter year → phải extract year
```

Nhưng thực tế:

> Chỉ cần check điều kiện đúng là đủ.

---

### 2. List rất mạnh trong Power Query

Case này tiếp tục làm mình thấy:

- `List.Transform`
- `List.AnyTrue`

rất phù hợp cho dynamic filtering.

---

### 3. Tối ưu không chỉ là performance

Mà còn là:

- ít step hơn
- code dễ đọc hơn
- maintain dễ hơn

---

### 4. Reusable pattern

Pattern này có thể reuse cho:

- latest months
- latest weeks
- version filtering
- dynamic folder filtering

---

## 📁 Files

- [Data](https://github.com/tramdangtai/real-world-data-problems/tree/main/02-power-query/optimizing-year-based-file-filtering-in-power-query/data)
- [Solution](https://github.com/tramdangtai/real-world-data-problems/tree/main/02-power-query/optimizing-year-based-file-filtering-in-power-query/solution)

# 📊 Dynamic File Filtering by Year-Month (Power Query M)

## 1. Problem Context

Trong quá trình build report, mình gặp một bài toán:

> Cần filter các file `.csv` theo format `YYYY_MM` trong tên file

### Mục tiêu:

- Không load toàn bộ dữ liệu
- Chỉ load dữ liệu của **tháng hiện tại + tháng trước đó**
- → **Tối ưu performance cho report**

### Cách tiếp cận:

Thay vì 1 file lớn → chia nhỏ thành nhiều file theo tháng  
→ Khi load data, chỉ đọc những file cần thiết

---

## 2. Initial Approach (My First Solution)

### Ý tưởng:

- Tạo 2 biến:
    - `currentMonth`
    - `lastMonth`
- Dùng `Text.Contains` để filter

### Code:

```
paramCurrentMonthFilter =   
	if param_value_currentMonth < 10 then Text.From(param_value_currentYear) & "_0"& Text.From(param_value_currentMonth)   
	else Text.From(param_value_currentYear) & "_"& Text.From(param_value_currentMonth),   
  
paramLastMonthFilter =   
	if param_value_currentMonth -1 < 10 then Text.From(param_value_currentYear) & "_0"& Text.From(param_value_currentMonth - 1)   
	else Text.From(param_value_currentYear) & "_"& Text.From(param_value_currentMonth - 1),   
  
FilteredRows_TwoLatestMonths =   
	Table.SelectRows(  
	#"2026",   
	each   
		Text.Contains([Name], paramCurrentMonthFilter) or   
		Text.Contains([Name], paramLastMonthFilter)),
```

---

## 3. Issues Identified

Sau khi review lại, có một số vấn đề:

- Lặp lại logic format tháng (`YYYY_MM`)
- Xử lý sai case tháng 1  
    → `1 - 1 = 0` → không tồn tại tháng 0
- Hard-code string → khó maintain
- `Text.Contains` + `or`  
    → không scale nếu cần filter nhiều tháng hơn

---

## 4. Optimized Approach

### Key Idea:

- Dùng **date handling** thay vì xử lý bằng number/string
- Tách logic thành function
- Dùng **list + functional approach** để scale

### Code:

```
// Create date  
currentDate = Date.From(DateTime.LocalNow()),  
lastMonthDate = Date.AddMonths(currentDate, -1),  
  
// Format function  
formatMonth = (d as date) => Date.ToText(d, "yyyy_MM"),  
  
// Create list of filters  
monthFilters = {  
    formatMonth(currentDate),  
    formatMonth(lastMonthDate)  
},  
  
// Apply filter  
FilteredRows_TwoLatestMonths =   
    Table.SelectRows(  
        Source,   
        each List.AnyTrue(  
            List.Transform(monthFilters, (m) => Text.Contains([Name], m))  
        )  
    ),
```

---

## 5. Key Learnings

### 5.1 Use Date Instead of Manual Calculation

```
Date.AddMonths(currentDate, -1)
```

- Tránh bug tháng 0
- Code rõ ràng hơn
- Dễ maintain hơn

---

### 5.2 Use Function to Avoid Repetition

```
formatMonth = (d as date) => Date.ToText(d, "yyyy_MM")
```

- Tránh duplicate logic
- Dễ reuse
- Clean hơn

---

### 5.3 Use List for Scalability

```
monthFilters = {  
    formatMonth(currentDate),  
    formatMonth(lastMonthDate)  
}
```

- Dễ mở rộng:

```
monthFilters = {  
    formatMonth(currentDate),  
    formatMonth(lastMonthDate),  
    formatMonth(Date.AddMonths(currentDate, -2))  
}
```

---

### 5.4 Functional Pattern: Dynamic OR

```
List.AnyTrue(  
    List.Transform(monthFilters, (m) => Text.Contains([Name], m))  
)
```

**Ý nghĩa:**

> Giữ lại dòng nếu `[Name]` chứa **ít nhất 1 giá trị trong list**

---

## 6. Logic Breakdown

Equivalent logic:

```
each   
    let  
        checks = List.Transform(  
            monthFilters,  
            (m) => Text.Contains([Name], m)  
        )  
    in  
        List.AnyTrue(checks)
```

---

## 7. Example

```
monthFilters = {"2026_03", "2026_02"}  
Name = "fact_sales_daily_level_sku_store_2026_01.csv"
```

### Step:

```
List.Transform → { true, false }  
List.AnyTrue → true
```

→ Row được giữ lại

---

## 8. Pattern Summary

### ❌ Old approach

```
Text.Contains([Name], A) or Text.Contains([Name], B)
```

### ✅ New approach

```
List.AnyTrue(  
    List.Transform(list, condition)  
)
```

---

## 9. Concept Mapping

|Function|Meaning|
|---|---|
|`List.AnyTrue`|OR|
|`List.AllTrue`|AND|
|`List.Transform`|Loop + map|
|`Text.Contains`|Condition|
|`Table.SelectRows`|Filter|

---

## 10. Key Takeaway

- Ưu tiên:
    - Date handling > String handling
    - Function > Copy-paste logic
    - List + functional pattern > Hard-coded conditions
- Đây là pattern có thể reuse:

```
FILTER rows WHERE ANY(condition over a list)
```

## 📁 Files
- [Folder]()
- [Solution]()

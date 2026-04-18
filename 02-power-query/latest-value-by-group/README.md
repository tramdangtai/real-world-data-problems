# 📊 Latest Value Extraction in Power Query (Correctness vs Performance)

## 📌 Overview

Đây là một case mình gặp khi xử lý dữ liệu trong Power Query:

> Lấy **Quantity tại ngày gần nhất (latest Posting Date)** theo từng group (SKU, Location)

Nghe thì đơn giản, nhưng thực tế:

- Có nhiều cách làm “nhanh”
- Nhưng **không đảm bảo đúng 100%**
- Và có những cách đúng, nhưng lại **không tối ưu hiệu suất**

Case này giúp mình nhận ra:

> Trong xử lý dữ liệu:  
> **Correctness luôn là ưu tiên số 1 → sau đó mới đến Performance**

---

## 🧩 Problem

Dataset dạng fact table:

- SKU  
- Location Code  
- Quantity  
- Posting Date  

Yêu cầu:

- Group theo:
  - SKU  
  - Location Code  
- Lấy:
  - Latest Posting Date  
  - **Quantity tại ngày đó**

---

### ❗ Vấn đề gặp phải

Một số cách phổ biến nhưng **không đáng tin cậy**:

---

### ❌ Cách 1: Sort Desc + Remove Duplicate

- Sort giảm dần theo Posting Date  
- Remove duplicate theo SKU  

→ **Sai trong một số trường hợp**

---

### ❌ Cách 2: Sort Desc + lấy dòng đầu

```m
Record.Field(_{0}, "Quantity")
```

→ **Không ổn định**

---

### Nguyên nhân

> Power Query **không đảm bảo thứ tự sau khi sort** trong mọi ngữ cảnh (đặc biệt trong `Table.Group`)

→ Không thể dựa vào sort để lấy latest value

---

## ⚠️ Constraints

- Logic nằm trong `Table.Group`
- Không được phá cấu trúc query
- Chỉ sửa phần tạo column `Latest Quantity`
- Không ảnh hưởng các step khác
- Phải test kỹ trước khi apply

---

## 🧠 Thinking Process

---

### 0. Setup để test

- Tạo bản copy của query để test
- Filter 1 vài SKU đại diện
- Tạo thêm column:

```m
each _
```

→ giữ lại toàn bộ **Nested Table** để debug

---

### 1. Thử dùng `Table.Max`

```m
each Table.Max(_, "Posting Date")[Quantity]
```

---

#### ❗ Vấn đề:

- Có nhiều dòng cùng ngày
- `Table.Max` trả về **random 1 dòng**

→ Sai logic

---

### 2. Giải đúng bằng Nested Table

```m
let   
	_sub = _,  
	_latest = List.Max(_sub[Posting Date]),  
	_filtered = Table.SelectRows(_sub, each [Posting Date] = _latest)  
in   
	List.Sum(_filtered[Quantity])
```

---

#### ✅ Ưu điểm:

- Đúng logic
- Xử lý được nhiều dòng cùng ngày

---

#### ❗ Nhược điểm:

- Dùng `Table.SelectRows`  
    → Power Query scan lại table  
    → **Không tối ưu performance**

---

### 3. Chuyển sang Record

```m
let   
	_sub = _,  
	_latest = List.Max(_sub[Posting Date]),  
	_qty = List.Sum(  
		List.Transform(  
			Table.ToRecords(_sub),  
			(r) => if r[Posting Date] = _latest then r[Quantity] else 0  
		)  
	)  
in   
	_qty
```

---

#### Nhận xét:

- Tốt hơn Table
- Nhưng vẫn chưa tối ưu nhất

---

## 💡 Final Solution — Làm việc hoàn toàn với List

```m
let   
	dates = [Posting Date],  
	qtys = [Quantity],  
	latest = List.Max(dates),  
	result = List.Sum(  
		List.Transform(  
			List.Positions(dates),  
			(i) => if dates{i} = latest then qtys{i} else 0  
		)  
	)  
in   
	result
```

---

## 🔍 Logic Breakdown

---

### Step 1 — Tạo 2 list song song

```m
dates = [Posting Date]  
qtys = [Quantity]
```

---

### Step 2 — Lấy ngày lớn nhất

```m
latest = List.Max(dates)
```

---

### Step 3 — Loop theo index

```m
List.Positions(dates)
```

---

### Step 4 — Filter + map

```m
(i) => if dates{i} = latest then qtys{i} else 0
```

---

### Step 5 — Sum

```m
List.Sum(...)
```

---

### 🧠 Ý tưởng tổng thể

```
Duyệt từng dòng:  
    Nếu Posting Date = latest → lấy Quantity  
    Ngược lại → lấy 0  
→ rồi cộng lại
```

---

## ⚙️ Optimization

---

### 1. Clean Code

```m
positions = List.Positions(dates)
```

→ Tách ra biến riêng để code dễ đọc hơn

---

### 2. Performance Insight

Hiện tại:

- `List.Transform` chạy toàn bộ list

---

### Ý tưởng tối ưu:

Chỉ xử lý những dòng có latest date

```m
filteredPositions =  
    List.Select(  
        positions,  
        (i) => dates{i} = latest  
    )
```

→ Sau đó chỉ sum các index cần thiết

---

### 💬 Insight quan trọng

> “Performance in Power Query is not about fewer lines of code — it's about fewer rows touched.”

---

## 🔁 Reusable Function

function name: `fn_sum_latest_by_date`

```m
(fnTable as table, dateCol as text, valueCol as text) as number =>
// Dùng trong Table.Group with purpose: calculate sum latest value of column (number type)
// Example use: fn_sum_latest_by_date(_, "latest_date_receving", "latest_qty_receving")
let
    // Extract list
    dates = Table.Column(fnTable, dateCol),
    values = Table.Column(fnTable, valueCol),

    // Latest date
    latest = List.Max(dates),

    // Positions (index)
    positions = List.Positions(dates),

    // Filter index theo latest date
    filteredPositions =
        List.Select(
            positions,
            (i) => dates{i} = latest
        ),

    // Sum value theo index đã filter
    result =
        List.Sum(
            List.Transform(
                filteredPositions,
                (i) => values{i}
            )
        )
in
    result
```

### Cách dùng

```m
fn_sum_latest_by_date(_, "latest_date_receving", "latest_qty_receving")
```

---

## 🤖 Role of AI

- Hỏi → hiểu → test → phản biện → hỏi tiếp
- Không copy solution
- Mà dùng AI như:
    
    > một “sparring partner” để đào sâu vấn đề
    

---

## 🧠 Key Takeaways

1. **Độ chính xác > code đơn giản**
2. Không tin vào sort khi xử lý dữ liệu trong Power Query
3. Performance = số lượng row được xử lý, không phải số dòng code
4. List là công cụ rất mạnh trong M Code
5. Luôn nghĩ đến khả năng **tái sử dụng (reusability)**

---

## 📁 File
- [Data](https://github.com/tramdangtai/real-world-data-problems/tree/main/02-power-query/latest-value-by-group/data)
- [Solution](https://github.com/tramdangtai/real-world-data-problems/tree/main/02-power-query/latest-value-by-group/solution)


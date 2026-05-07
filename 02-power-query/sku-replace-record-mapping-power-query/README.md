---
date: 2026-05-07
---
# 📊 SKU Replace Mapping with Record Dictionary (Power Query)

## 📌 Overview

Trong hệ thống thực tế, có những SKU cũ sẽ ngừng sử dụng và được thay thế bằng SKU mới (SKU Replace).

Vấn đề là:

> Khi SKU replace được tạo trên hệ thống, toàn bộ lịch sử giao dịch trước đó của SKU mới sẽ trống hoàn toàn.

Trong khi thực tế:
- Đây vẫn là cùng một sản phẩm
- Chỉ thay đổi mã SKU

Nếu không xử lý:
- Purchase Order sẽ thiếu lịch sử bán hàng
- Tracking sản phẩm bị đứt đoạn
- Các báo cáo historical bị sai lệch

Mục tiêu của case này là:

> Mapping toàn bộ lịch sử giao dịch từ SKU cũ sang SKU replace để đảm bảo việc tracking được liên tục và chính xác.

---

## 🧩 Problem

Dataset gồm:

- `fact_sales`
- `mapping_item_replace`

Logic cần xử lý:

- Nếu SKU có replace:
  - dùng SKU replace
- Nếu không:
  - giữ SKU cũ

Sau đó:
- Group lại dữ liệu
- Sum sales / quantity

---

### ❌ Cách làm ban đầu

Ban đầu mình xử lý bằng flow thông thường:

1. Merge mapping table vào fact table
2. Expand cột SKU replace
3. Add column:
   - nếu SKU replace = null → dùng SKU cũ
   - ngược lại → dùng SKU replace
4. Xóa SKU cũ
5. Rename temporary column
6. Table.Group lại

---

### ❗ Vấn đề

Cách này:

- Quá nhiều step
- Khó maintain
- Query dài
- Khó reuse
- Và không “đẹp”

---

## ⚠️ Constraints

- Tên cột SKU phải giữ nguyên
- SKU replace phải đúng
- SKU sau replace phải unique
- Toàn bộ lịch sử giao dịch phải được chuyển sang SKU mới

---

## 🧠 Thinking Process

Mình bắt đầu tìm cách tối ưu hơn.

Sau khi hỏi ChatGPT, mình được gợi ý:

> Dùng `Record` như một dạng Dictionary để mapping.

Vì trước đó mình từng học Python, nên khi nghe đến:

```text
Key : Value
```

thì mình liên tưởng ngay đến Dictionary.

---

### Ý tưởng

Thay vì:

- Merge table
- If else
- Temporary column
- Rename column

→ Chỉ cần:

```text
Old SKU → New SKU
```

---

## 💡 Building the Mapping Record

Để dùng được:

```m
Record.FromTable
```

table mapping cần có:

| Name | Value |
|---|---|
| old_sku | new_sku |

---

### Một điều mình phát hiện khá thú vị

Table mapping của mình thực tế có nhiều cột hơn 2 field.

Nhưng khi test:

```m
Record.FromTable
```

thì mình nhận ra:

> M Code này chỉ đọc 2 cột:
- `Name`
- `Value`

Các cột khác hoàn toàn bị bỏ qua.

---

## 💡 Final Solution

### Create Mapping Record

```m
MappingRecord = Record.FromTable(mapping_item_replace)
```

---

### Replace SKU trực tiếp

```m
Table.TransformColumns(  
 fact_sales,  
	{{  
		"sku",  
		each Record.FieldOrDefault(MappingRecord, _, _),  
		type text  
	}}  
)
```

---

## 🔍 Logic Breakdown

### `Table.TransformColumns`

Loop qua từng dòng của cột `sku`

---

### `Record.FieldOrDefault`

```m
Record.FieldOrDefault(MappingRecord, _, _)
```

Ý nghĩa:

- Check giá trị SKU hiện tại
- Nếu tồn tại trong MappingRecord:
  - return SKU replace
- Nếu không:
  - return chính SKU cũ

---

### Ý tưởng tổng thể

```text
Nếu SKU tồn tại trong mapping:
    replace bằng SKU mới
Ngược lại:
    giữ nguyên SKU cũ
```

---

## ⚡ Impact

Chỉ với một đoạn M Code ngắn:

```m
Record.FieldOrDefault(...)
```

→ thay thế được:
- Merge
- If Else
- Temporary column
- Remove column
- Rename column

---

## ⚙️ Optimization

### 1. Reuse Mapping Record

Vì logic replace SKU được dùng ở nhiều query:

→ mình tạo riêng:

```text
map_record_itemReplace
```

để dễ reference và reuse.

---

### 2. Tối ưu query structure

Ban đầu:

- 1 query table
- 1 query record

Sau đó mình nhận ra:

> Không cần giữ query table trung gian nữa.

→ Gộp trực tiếp thành query Record luôn:
- Ít step hơn
- Ít dependency hơn
- Query gọn hơn

---

## 🧠 Key Takeaways

### 1. Luôn có cách làm tốt hơn

Ban đầu mình nghĩ:
- Merge + If Else là “bình thường”

Nhưng sau khi hiểu Record:
- Có thể replace toàn bộ flow bằng một pattern tối ưu hơn rất nhiều.

---

### 2. Record cực mạnh cho Mapping

`Record` trong Power Query hoạt động rất giống:
- Dictionary
- Hash map

→ Rất phù hợp cho:
- Mapping
- Lookup
- Replace logic

---

### 3. Hiểu bản chất quan trọng hơn nhớ syntax

Case này giúp mình hiểu rõ hơn:
- Khi nào nên dùng:
  - Table
  - Record
  - List

Mỗi loại structure sẽ phù hợp với một mục tiêu khác nhau.

---

## 🤖 Role of AI

Mình không dùng AI để “copy solution”.

Quy trình thực tế là:

- Hỏi
- Test
- Đọc lại code
- Tự giải thích
- So sánh performance
- Refactor tiếp

AI giúp mình:
- mở rộng góc nhìn
- tiếp cận pattern mới nhanh hơn

Nhưng việc hiểu và quyết định áp dụng như thế nào vẫn là phần quan trọng nhất.

---

## 📁 File

- [Data](https://github.com/tramdangtai/real-world-data-problems/tree/main/02-power-query/sku-replace-record-mapping-power-query/data)
- [Solution](https://github.com/tramdangtai/real-world-data-problems/tree/main/02-power-query/sku-replace-record-mapping-power-query/solution)

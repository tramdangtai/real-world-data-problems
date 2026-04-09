# 📊 Dynamic Cumulative Sales trong Excel (Không thay đổi cấu trúc dữ liệu)

## 📌 Tổng quan

Đây là một case thực tế mình gặp khi làm việc với Excel trong môi trường công ty.

Bài toán nghe khá đơn giản:

> Tính **sales lũy kế theo tháng**

Nhưng khi bắt tay vào làm, mình nhận ra:  
→ Vấn đề không nằm ở việc “tính tổng”  
→ Mà nằm ở cách **xử lý dữ liệu trong điều kiện bị giới hạn**

---

## 🧩 Bối cảnh thực tế

Ở công ty mình:

- Phần lớn data vẫn xử lý bằng **Excel**
- Power BI có dùng, nhưng không nhiều
- Dữ liệu thường được giữ ở dạng bảng để **lấy số trực tiếp**, không phải để visualize

→ Đây là một bối cảnh rất phổ biến trong doanh nghiệp

---

## 📁 Cấu trúc dữ liệu

Data có dạng:

- **Rows (dòng):** Description / Product
- **Columns (cột):** Tháng (Jan → Dec)
- **Values:** Sales

👉 Quan trọng: dữ liệu nằm **ngang (horizontal)**, không phải dạng chuẩn “long format”

---

## 🎯 Yêu cầu bài toán

- Tính **sales lũy kế** cho từng Description
- Cho phép user **chọn tháng (1–12)** tại một ô input
- Kết quả phải **dynamic theo tháng được chọn**

---

## ⚠️ Các ràng buộc (constraints)

Đây là phần làm bài toán trở nên “thực tế”:

- ❌ Không được thay đổi cấu trúc dữ liệu
- ❌ Không dùng Power Query / Power Pivot
- ❌ Không tách ra nhiều cột phụ
- ✅ Công thức phải nằm trong **1 ô duy nhất**
- ✅ Phải **giải thích được logic** khi người khác hỏi

---

## 🧠 Quá trình suy nghĩ

### 1. Loại bỏ các hướng không phù hợp

- `VLOOKUP` → chỉ dùng để lookup, không xử lý được cumulative
- `SUMIF` → có thể dùng, nhưng range bị **fix cứng**, không dynamic

→ Mình nhận ra:

> Thứ mình thiếu không phải là hàm SUM  
> Mà là cách tạo **range động**

---

### 2. Bước ngoặt: Hiểu lại về hàm INDEX

Khi hỏi ChatGPT, mình nhận được gợi ý:
```
=SUM(F7:INDEX(F7:Q7, param_lookup_value))
```
Ban đầu mình khá bất ngờ:

→ `INDEX` có thể trả về **một ô (reference)**  
→ Và có thể dùng để tạo **range**

---

### 📌 Insight quan trọng:

`INDEX` không chỉ trả về value  
→ mà có thể trả về **vị trí ô trong Excel**

Điều này cho phép mình làm:
```
F7 : INDEX(...)
```
→ tạo ra một range từ đầu → đến tháng được chọn

---

### 3. Vấn đề tiếp theo: Không thể fix dòng

Công thức trên chỉ đúng cho dòng 7.

Nhưng trong thực tế:

- Có nhiều Description
- Không biết trước dòng nào sẽ được tính

→ Không thể hardcode `F7`

---

### 4. Giải pháp: MATCH để tìm đúng dòng
```
MATCH(param_product, A7:A100, 0)
```
→ trả về đúng dòng chứa Description cần tính

---

### 5. Ghép lại toàn bộ logic
```
=SUM(
  INDEX(F7:Q100, MATCH(product, A7:A100, 0), 1)
  :
  INDEX(F7:Q100, MATCH(product, A7:A100, 0), param_lookup_value)
)
```
---

### Ý tưởng chính:

- `MATCH` → tìm đúng dòng
- `INDEX(..., row, 1)` → điểm bắt đầu
- `INDEX(..., row, month)` → điểm kết thúc
- `:` → tạo range
- `SUM` → tính lũy kế

---

## 🚀 Tối ưu & làm sạch công thức với LET

Khi công thức bắt đầu dài và lặp lại nhiều logic, mình dùng `LET`:

```
=LET(  
  desValueToFind, $I7,  
  desRangeToMatch, 'PL S01 LY'!$B$7:$B$100,  
  valueRangeToCalculate, 'PL S01 LY'!$F$7:$Q$100,  
  
  rowNum, MATCH(desValueToFind, desRangeToMatch, 0),  
  startRange, INDEX(valueRangeToCalculate, rowNum, 1),  
  endRange, INDEX(valueRangeToCalculate, rowNum, param_lookup_value),  
  
  SUM(startRange:endRange)  
)
```

---

### ✅ Lợi ích của LET

- Không phải lặp lại `MATCH` nhiều lần
- Dễ đọc hơn (giống viết code)
- Dễ debug
- Dễ maintain khi đổi sheet / range

---

## ⚙️ Một số tối ưu nhỏ nhưng quan trọng
### 1. Giới hạn range

❌ Không nên:

```
F:Q
```

✅ Nên:

```
F7:Q100
```

→ Giảm đáng kể chi phí tính toán

---

### 2. Đặt tên biến rõ ràng

Ví dụ:

- `desValueToFind`
- `valueRangeToCalculate`

→ Giúp người khác đọc hiểu nhanh hơn

---

### 3. Tránh volatile functions

- `OFFSET`, `INDIRECT` → có thể gây chậm  
    → `INDEX` là lựa chọn tốt hơn

---

## 🤖 Vai trò của ChatGPT

Trong case này, mình không dùng ChatGPT để “xin công thức hoàn chỉnh”.

Mà dùng theo kiểu:

1. Gợi ý hướng đi (INDEX)
2. Tự kiểm chứng lại (Microsoft Docs)
3. Tiếp tục refine (MATCH → LET → performance)

---

## 🧠 Điều mình học được

Không phải là một hàm Excel mới.

Mà là:

> Hiểu sâu hơn về cách Excel xử lý **reference vs value**

Và quan trọng hơn:

> Cách kết nối những thứ mình đã biết

- INDEX
- MATCH
- SUM
- tư duy dynamic

---

## 🔄 Ứng dụng thực tế

Pattern này có thể áp dụng cho:

- YTD (Year-to-date)
- Rolling 3 tháng / 6 tháng
- Báo cáo tài chính
- Bất kỳ data nào dạng horizontal

---

## 🚀 Vì sao case này quan trọng

Trong thực tế:

- Data không “đẹp”
- Không phải lúc nào cũng được chọn tool
- Constraint luôn tồn tại

→ Khả năng làm việc trong constraint  
quan trọng không kém kỹ năng tool

---

## 📌 Một suy nghĩ cá nhân

Những thứ như:

- data cleaning
- data structure
- viết công thức

…thường rất chán.

Nhưng:

> Nó là nền móng của mọi thứ phía sau.

---

## 📬 Kết

Có một câu mình từng đọc:

> “Một ngày nào đó, những mảnh ghép rời rạc sẽ tự kết nối lại.”

Case này là một lần mình cảm nhận rõ điều đó.

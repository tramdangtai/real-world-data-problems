# 📊 KPI Chart in Excel (Actual vs Target with % Achieved)

> A real-world Excel workaround to simulate KPI-style charts without Power BI.

## 📌 Overview

Đây là một case thực tế từ đồng nghiệp trong team:  
nhờ mình build một chart trong Excel để theo dõi:

- Doanh số thực tế (Actual)
- Doanh số mục tiêu (Target)
- % đạt được theo từng tháng

Yêu cầu quan trọng là:

> Nhìn vào chart phải hiểu ngay:
> - Đã đạt target chưa
> - Còn thiếu bao nhiêu
> - % đạt được là bao nhiêu

---

## 🧩 Problem

Excel không có sẵn dạng chart KPI như Power BI.

Một số hướng thử ban đầu:

- **Clustered Column**
  - Hiển thị được số (Actual)
  - ❌ Không thể hiện % đạt được

- **Stacked Column / 100% Stacked**
  - ❌ Không thể hiện rõ Actual vs Target
  - ❌ Không thấy “khoảng cách” để đạt target

→ Không có cách nào trực tiếp để vừa:
- Hiển thị số
- Hiển thị %
- Và thể hiện KPI một cách trực quan

---

## ⚠️ Constraints

- Không dùng:
  - Power Query  
  - Pivot Chart  
- Phải vẽ trực tiếp trên Excel
- Chart cần:
  - Vừa hiển thị số (Actual)
  - Vừa hiển thị % đạt
  - Thể hiện rõ đã đạt target hay chưa
- Có thể in ra (print-friendly)

---

## 🧠 Thinking Process

- Mình thử hỏi ChatGPT:
  - Gợi ý dùng **Column + Line Chart**
    - Column = Actual  
    - Line = Target  

→ Nhưng không phù hợp:

- Line chart tạo cảm giác “liên tục theo thời gian”
- Trong khi mình chỉ cần:
  - Target = một “mốc” (threshold), không phải trend

---

→ Tiếp tục tìm hiểu → gặp được video này:
[How to Make a Horizontal Bullet Graph in Excel](https://www.youtube.com/watch?v=qcL33P_Y1Io)

→ Đây chính là hướng design gần giống với nhu cầu:
- Có “thanh” thể hiện actual
- Có “mốc ngang” thể hiện target

---

## 💡 Solution

### Core idea:

Vẫn dùng **Clustered Column**, nhưng “biến tấu” lại:

---

### 1. Actual (Column)

- Dùng column bình thường để thể hiện doanh số thực tế

---

### 2. Target (Hidden Column)

- Tạo thêm 1 series Target:
  - Không fill màu (invisible)
  - Set **Series Overlap = 100%** để chồng lên Actual

---

### 3. Dùng Error Bar để tạo “target line” cho Series Target

Cấu hình:

- **Vertical Error Bar**
  - Direction = Minus  
  - End Style = No Cap  

- **Error Amount**
  - Percentage = 100%

- **Line**
  - Solid line  
  - Width = 20 pt  

→ Đây chính là “thanh ngang” thể hiện Target

---

## ⚙️ Optimization

### 1. Tránh Data Label bị đè

- Nếu show cả Actual & Target → bị chồng chữ
- Giải pháp:
  - Chỉ hiển thị Data Label của Actual
  - Target → dùng axis + gridlines để đọc

---

### 2. Hiển thị đồng thời số + %

Tạo thêm 1 cột:

```
Actual w % Achieved
```

→ Dùng công thức Excel để combine:

- Giá trị thực tế
- % đạt được

Ví dụ:

```
100,000 (80%)
```

---

### 3. Dùng "Value From Cells"

- Không dùng label mặc định
- Dùng:
  - **Value From Cells**
  - Chọn range chứa:
    - số + %

---

### 4. Tối ưu hiển thị

Vấn đề

- Label dài (số + %) → bị kéo ngang
- Nhỏ chữ → khó đọc

Giải pháp:

- Dùng:

```
CHAR(10)
```

→ Xuống dòng trong label

Ví dụ:

```
100,000  
(80%)
```

→ Giữ font size lớn, dễ đọc hơn

---

### 5. Tối ưu khi bị đè lên Target line

Vấn đề:

- Label có thể nằm đè lên target line

Giải pháp:

- Thêm **Glow effect** cho text

→ Khi in:
- Vẫn thấy số
- Vẫn thấy line
- Không mất thông tin

---

### 6. Format số lớn (K / M) để dễ đọc

### Vấn đề 1:

- Khi doanh số lớn:
    - Data Label bị dài (ví dụ: 1,245,678)
    - Khó đọc, đặc biệt khi in
- User không cần độ chính xác tuyệt đối  
    → chỉ cần **magnitude (K / M)**

Giải pháp:

- Dùng `LET` để:
    - Tách logic rõ ràng
    - Dễ maintain
- Format theo điều kiện:
    - ≥ 1,000 → K
    - ≥ 1,000,000 → M
- Kết hợp với `% achieved`
- Dùng `IFERROR` để tránh lỗi (ví dụ chia cho 0)

Công thức:

```
=LET(   
  salesActual, C465,  
  salesTarget, C466,  
  salesActualWithFormat,  
    SWITCH(  
      TRUE(),  
      salesActual >=1000000, TEXT(salesActual/1000000,"0.0")&"M",  
      salesActual >=1000, TEXT(salesActual/1000,"0.0")&"K",  
      TEXT(salesActual,"0.0")  
    ),  
  percentageAchieved, TEXT(salesActual/salesTarget,"0%"),  
  result, salesActualWithFormat & CHAR(10) & "(" & percentageAchieved & ")",  
  IFERROR(result,"")  
)
```

### Vấn đề 2:

- Tương tự như vấn đề 1, số cần được chuyển sang định dạng K, M ở Axis

Giải pháp:

- Sử dụng custom trong Number tại Format Axis

Công thức:

```
[>=1000000]0,,"M";[>=1000]0,"K";0
```

Insight:

> Với KPI chart: readability > precision

---

### 7. Trick hiển thị Legend cho Target

Vấn đề:

- Target đang:
    - Không fill màu
    - Overlap 100%  
        → Legend icon bị “mất”

→ User nhìn chart sẽ:

- Không biết thanh ngang là gì

Giải pháp (simple nhưng hiệu quả):

- Sửa **Series Name của Target**
- Thêm ký tự `-` phía trước

Ví dụ:

```
- Target
```

Kết quả:

- Legend sẽ hiển thị:
    - Một dấu gạch ngang
- Người dùng dễ hiểu:
    - Đây là “target line”

---

Insight:

- Không cần chỉnh chart phức tạp
- Chỉ cần **tận dụng cách Excel render text**

---

## 🧠 Key Takeaways

- Excel không có sẵn KPI chart  
→ Nhưng hoàn toàn có thể **tự build bằng cách combine nhiều feature**

- Quan trọng không phải là chart gì  
→ Mà là:
> “Người xem có hiểu được vấn đề ngay không?”

- Một chart tốt cần:
  - Thể hiện đúng context
  - Trực quan
  - Dễ đọc (đặc biệt khi in)

---

## 📁 Files

- data
- solution

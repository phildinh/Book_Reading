# Forecasting Notes — Công cụ chẩn đoán (Trước khi chọn phương pháp)

*Notebook đồng hành với notes Moving Average và Exponential Smoothing. Hai notebook đó nói về chính các phương pháp forecasting. Notebook này nói về bước nên làm **trước** khi chọn giữa các phương pháp đó: làm sao biết chắc dữ liệu của mình có trend thật hay seasonality thật, thay vì đoán từ biểu đồ? Hai công cụ ở đây, cả hai đều xây hoàn toàn từ những gì bạn đã biết (hypothesis testing ở Section 3, correlation/regression ở Section 4) — không có thống kê mới, chỉ là một cách ứng dụng mới.*

---

## Block 0 — Tại sao bước này tồn tại

Mọi phương pháp forecasting từ trước tới giờ (SMA cho tới Holt-Winters) đều yêu cầu bạn **quyết định trước** xem trend và/hoặc seasonality có tồn tại hay không, vì quyết định đó xác định phương pháp nào phù hợp:

- Không trend, không seasonality → SMA/WMA/SES là đủ.
- Có trend thật, không có seasonality → cần Holt's method.
- Có trend thật + seasonality thật → cần Holt-Winters.

**Cái bẫy:** nhìn biểu đồ bằng mắt không đáng tin — noise có thể che giấu một mẫu hình thật, và mắt bạn cũng có thể "thấy" một mẫu hình trong sự ngẫu nhiên thuần túy dù nó không thực sự tồn tại. **Hai công cụ này biến "cái này trông có vẻ theo mùa không?" thành một con số thực sự với một quy tắc quyết định** — đúng cách mà mọi hypothesis test ở Section 3 đã thay thế "cảm thấy có ý nghĩa không?" bằng một p-value.

---

## Block 1 — Công cụ 1: Autocorrelation (ACF)

**Định nghĩa:** correlation — chính xác là `CORREL()` từ Section 4 — áp dụng cho một chuỗi **so với chính bản sao bị dịch (lag) của nó**, thay vì hai biến khác nhau.

### Ý tưởng cốt lõi: xây dựng một "lag"

Lấy dữ liệu của bạn, tạo một bản sao thứ hai, và trượt nó đi k kỳ. Ghép mỗi giá trị với giá trị từ k kỳ trước đó. Chạy `CORREL()` trên hai cột kết quả.

**Lag-1** = ghép mỗi kỳ với kỳ ngay trước nó → kiểm tra momentum ngắn hạn/hành vi giống trend.
**Lag-p** (p = độ dài mùa vụ nghi ngờ, vd 12 theo tháng / 4 theo quý) = ghép mỗi kỳ với cùng điểm ở chu kỳ trước → kiểm tra seasonality.

### Mẹo ghi nhớ ACF

- 🧠 **"Lag = trượt dải giấy sang."** Nếu khái niệm nào đó cảm thấy trừu tượng, quay lại đây: viết chuỗi dữ liệu hai lần, trượt một bản sao đi k ô, tính correlation giữa hai cột bạn vừa có.
- 🧠 **Một tiếng vọng hoàn hảo = correlation hoàn hảo, và thường bạn có thể thấy trước khi tính toán.** Nếu hai cột bị lag di chuyển rõ ràng đồng bộ (mỗi cặp chênh nhau một lượng gần như nhau), bạn đã biết correlation sẽ cao — phép tính chỉ xác nhận điều mắt bạn đã bắt được.
- 🧠 **Kiểm tra cả lag-1 VÀ lag-p — đừng dừng ở lag-1.** Một chuỗi có thể có correlation lag-1 yếu (không có momentum ngày-qua-ngày đơn giản) trong khi vẫn có một đỉnh lag-p mạnh (seasonality thật) — đây là hai câu hỏi riêng biệt, và dữ liệu theo mùa thường trông "lộn xộn" ở lag-1 trong khi rất rõ ràng ở lag-p.
- 🧠 **Ngưỡng ý nghĩa: ±1.96/√N** (N = tổng số quan sát trong chuỗi). Bất kỳ giá trị ACF nào nằm **trong** ngưỡng này không thể phân biệt được với 0 về mặt thống kê — coi đó là noise, không phải signal.
- 🧠 **⚠️ Mẫu nhỏ có thể tạo ra đỉnh trông ấn tượng nhưng không đáng tin.** Một correlation lag-p "hoàn hảo" tính từ chỉ 2–3 cặp (tức chỉ 2–3 năm lịch sử) nên khiến bạn *thận trọng hơn*, không phải tự tin hơn — đây là cùng vấn đề sức mạnh mẫu nhỏ (small-sample power) từ Section 3. Ưu tiên có 3+ chu kỳ đầy đủ trước khi tin vào kết quả ACF theo mùa.

### Ví dụ tính tay — xây trực giác đơn giản, không cần tính toán

Dữ liệu tăng đều: 100, 105, 110, 115, 120.

| "Hôm nay" | "Hôm qua" (lag-1) |
|---|---|
| 105 | 100 |
| 110 | 105 |
| 115 | 110 |
| 120 | 115 |

Mỗi cặp đều tăng +5 — một tiếng vọng hoàn hảo rõ ràng. **Correlation = 1.0**, không cần tính toán để thấy trước điều đó.

Mẫu hình lặp lại (độ dài mùa vụ 3): 10, 20, 30, 10, 20, 30.

| "Kỳ này" | "3 kỳ trước" (lag-3) |
|---|---|
| 10 | 10 |
| 20 | 20 |
| 30 | 30 |

Các cặp giống hệt nhau. **Correlation = 1.0** — dấu hiệu của seasonality.

### Ví dụ tính tay — dữ liệu quý thực tế của Supagas (200, 250, 300, 250, 220, 275, 330, 275)

**Cột lag-1:**

| Quý này | Quý trước |
|---|---|
| 250 | 200 |
| 300 | 250 |
| 250 | 300 |
| 220 | 250 |
| 275 | 220 |
| 330 | 275 |
| 275 | 330 |

Lộn xộn — đôi khi tăng theo sau tăng, đôi khi một giá trị lớn theo sau là giảm. **CORREL ≈ 0.17.**

**Cột lag-4 (cùng quý, một năm trước):**

| Quý này | Cùng quý năm ngoái |
|---|---|
| 220 | 200 |
| 275 | 250 |
| 330 | 300 |
| 275 | 250 |

Mỗi dòng đều di chuyển theo cùng một cách nhất quán. **CORREL = 1.00.**

**Kiểm tra ý nghĩa:** ngưỡng = ±1.96/√8 ≈ ±0.69. 0.17 của lag-1 nằm **trong** ngưỡng (không có ý nghĩa — không có tín hiệu momentum ngắn hạn đáng tin cậy). 1.00 của lag-4 nằm **xa ngoài** ngưỡng (rõ ràng có ý nghĩa — seasonality thật), dù đã lưu ý ở trên là dựa trên rất ít cặp, nên coi đây là manh mối mạnh, chưa phải bằng chứng cuối cùng, cho tới khi có thêm nhiều năm dữ liệu xác nhận.

---

## Block 2 — Công cụ 2a: Kiểm định Trend (Regression Slope + Hypothesis Test)

**Thiết lập:** X = chỉ số thời gian (1, 2, 3, ...), Y = demand. Fit simple linear regression (Section 4, Block 5–6). Slope b₁ là ước lượng trend của bạn.

**Kiểm định giả thuyết (quy trình 5 bước của Section 3, áp dụng cho hệ số hồi quy):**
- H₀: β₁ = 0 (slope có thể chỉ là noise)
- H₁: β₁ ≠ 0 (có trend thật)
- **Công cụ Regression của Excel / `LINEST` cho bạn p-value của slope trực tiếp** — không cần tự suy ra test statistic bằng tay.
- Quy tắc quyết định: giống như mọi khi. p-value < 0.05 → bác bỏ H₀ → trend là thật. p-value ≥ 0.05 → coi slope là noise.

### Mẹo ghi nhớ

- 🧠 **Đây không phải một kiểm định mới — nó là cùng logic 5-bước hypothesis testing từ Section 3, chỉ nhắm vào một hệ số hồi quy thay vì một mean hay proportion.**
- 🧠 **Một slope dốc không tự động là "thật."** Với rất ít dữ liệu, ngay cả một slope trông có vẻ dốc thật sự vẫn có thể không đạt ý nghĩa thống kê — mẫu nhỏ có power thấp (Section 3, Block 3), và việc phát hiện trend cũng không ngoại lệ.

### Ví dụ tính tay — sự tương phản làm rõ quy tắc quyết định

| Dữ liệu phẳng/nhiễu (100, 98, 102, 99, 101) | Dữ liệu tăng rõ ràng (100, 110, 120, 130, 140) |
|---|---|
| Slope ≈ 0.3 | Slope = 10 |
| p-value ≈ 0.85 → không bác bỏ H₀ | p-value ≈ 0.0001 → bác bỏ H₀ |
| Coi như không có trend thật | Trend thật, đã xác nhận thống kê |

---

## Block 3 — Công cụ 2b: Kiểm định Seasonality (ANOVA giữa các nhóm)

**Thiết lập:** nhóm dữ liệu của bạn theo "vị trí trong chu kỳ" (vd: theo quý, hoặc theo tháng) và hỏi đúng câu hỏi mà ANOVA đã trả lời từ Section 3b: **các mean của nhóm có khác nhau nhiều hơn mức ngẫu nhiên có thể giải thích không?**

- H₀: tất cả các nhóm (vd: cả 4 quý) có cùng mean demand thật — không có hiệu ứng mùa vụ.
- H₁: ít nhất một nhóm thực sự khác biệt.

### Mẹo ghi nhớ

- 🧠 **Trực giác, trước khi đến công thức:** so sánh độ phân tán **giữa** các mean nhóm với độ phân tán **trong** mỗi nhóm (qua các năm khác nhau, cùng quý). Nếu các quý không thực sự khác nhau, hai loại phân tán này nên trông tương tự về độ lớn. Độ phân tán giữa-nhóm lớn hơn hẳn độ phân tán trong-nhóm chính là tín hiệu seasonality.
- 🧠 **Điều này tái sử dụng chính xác công thức ANOVA của Section 3b** — F = MS_between / MS_within, so với F-critical cho bậc tự do của bạn.

### Ví dụ tính tay — dữ liệu Supagas thực tế, nhóm theo quý

| Quý | Năm 1 | Năm 2 | Trung bình quý |
|---|---|---|---|
| Q1 | 200 | 220 | 210.0 |
| Q2 | 250 | 275 | 262.5 |
| Q3 | 300 | 330 | 315.0 |
| Q4 | 250 | 275 | 262.5 |

Trung bình chung = 262.5. Trung bình các quý dao động xa tới ±52.5 so với trung bình chung; trong cùng một quý, chênh lệch năm-qua-năm chỉ 20–30. Sự mất cân bằng đó thể hiện qua:

| Nguồn | SS | df | MS | F |
|---|---|---|---|---|
| Giữa các quý | 11,025 | 3 | 3,675 | **11.53** |
| Trong các quý | 1,275 | 4 | 318.75 | |

F-critical (α=0.05, df=3,4) ≈ 6.59. Vì 11.53 > 6.59: **bác bỏ H₀ — seasonality được xác nhận thống kê.**

---

## Block 4 — Bảng tổng hợp nhanh

| Công cụ | Tái sử dụng | Kiểm tra cái gì | Kết quả trên ví dụ Supagas |
|---|---|---|---|
| ACF lag-1 | `CORREL()` (Section 4) | Gợi ý momentum ngắn hạn/trend | 0.17 — không có ý nghĩa |
| ACF lag-p | `CORREL()` (Section 4) | Chu kỳ mùa vụ lặp lại | 1.00 — có ý nghĩa (nhưng có lưu ý mẫu nhỏ) |
| Kiểm định slope hồi quy | Simple linear regression + hypothesis test (Section 3+4) | Có trend thật hay không | Cần thêm dữ liệu để kiểm tra đáng tin cậy |
| ANOVA trên các nhóm | ANOVA (Section 3b) | Các nhóm mùa vụ có thực sự khác nhau không | F=11.53 > 6.59 — có ý nghĩa |

**Khi hai công cụ độc lập (vd: ACF và ANOVA) đồng thuận cùng một kết luận, sự đồng thuận đó mới là điều khiến phát hiện đáng tin cậy — không phải một kiểm định đơn lẻ.**

---

## Block 5 — Tham khảo Excel

| Việc cần làm | Cách làm |
|---|---|
| Xây cột lag | Đơn giản là dịch tham chiếu cột đi k dòng (vd: `=A2` copy bắt đầu từ một dòng dưới cho lag-1) |
| Autocorrelation | `=CORREL(vùng_gốc, vùng_bị_lag)` |
| Ngưỡng ý nghĩa cho ACF | `=1.96/SQRT(N)` — so sánh giá trị ACF của bạn với ± ngưỡng này |
| Kiểm định trend | Data Analysis add-in → **Regression**, với X = một cột chỉ số thời gian; đọc p-value cạnh hệ số X |
| Kiểm định seasonality | Data Analysis add-in → **ANOVA: Single Factor**, với mỗi cột là một "vị trí trong chu kỳ" (vd: một cột cho mỗi quý) |

---

## Block 6 — Các bẫy phổ biến

1. **"Correlation lag-1 cao luôn có nghĩa là seasonality."** ❌ Lag-1 kiểm tra momentum ngắn hạn/hành vi giống trend. Seasonality được kiểm tra cụ thể ở **lag = độ dài mùa vụ**, không phải lag-1.
2. **"Một đỉnh ACF hoàn hảo từ 2 năm dữ liệu là bằng chứng vững chắc."** ❌ Rất ít cặp dữ liệu có thể tạo ra kết quả trông ấn tượng nhưng mong manh về mặt thống kê — coi đó là manh mối, xác nhận bằng thêm lịch sử.
3. **"Nếu biểu đồ trông phẳng, chắc chắn không có trend."** ❌ Và ngược lại — một biểu đồ trông như đang trending vẫn có thể không đạt kiểm định ý nghĩa chính thức nếu quá ít dữ liệu. Để p-value quyết định, không phải con mắt.
4. **"ANOVA và kiểm định trend hồi quy là thống kê hoàn toàn mới."** ❌ Không phải — cả hai đều là ứng dụng lại trực tiếp của Section 3 (hypothesis testing, ANOVA) và Section 4 (regression), chỉ nhắm vào câu hỏi chẩn đoán forecasting thay vì câu hỏi kinh doanh chung.

---

**Tiếp theo:** playbook chọn phương pháp & kết hợp (blending) — tổng hợp mọi phương pháp forecasting và mọi công cụ chẩn đoán đã học thành một hướng dẫn quyết định thực tế.

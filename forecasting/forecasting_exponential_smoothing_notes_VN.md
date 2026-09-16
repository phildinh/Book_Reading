# Forecasting Notes — Họ Exponential Smoothing (Simple, Double/Holt's, Triple/Holt-Winters)

*Notebook đồng hành với notes họ Moving Average. Cùng ý tưởng, nâng lên một bậc: thay vì chọn tay độ dài window (SMA) hay bộ trọng số cố định (WMA), exponential smoothing tự động tạo ra trọng số giảm dần chỉ từ một hằng số có thể tinh chỉnh — rồi mở rộng ý tưởng đó để theo dõi cả trend và seasonality. Bao gồm cả ba cấp độ, kèm mẹo ghi nhớ, ví dụ tính tay, cái bẫy về khởi tạo (initialization) mà bạn đã phát hiện ra, và bảng tổng hợp nhanh.*

---

## Block 0 — Phả hệ (genealogy), trong một bảng (đọc cái này trước)

| Phương pháp | Theo dõi | Nó khắc phục gì so với cái trước | Nó vẫn còn sai ở đâu |
|---|---|---|---|
| **SMA** | Chỉ Level (trung bình window phẳng) | Triệt tiêu noise nhờ lấy trung bình | Cân bằng dữ liệu cũ & mới; trễ n kỳ khi có level shift thật |
| **WMA** | Chỉ Level (trọng số tùy chỉnh) | Cân dữ liệu gần đây nhiều hơn | Trọng số chọn tay, tùy tiện, cố định; vẫn bỏ hoàn toàn mọi thứ ngoài window |
| **Simple Exponential Smoothing (SES)** | Chỉ Level (trọng số tự giảm dần) | Một hằng số tinh chỉnh (α) thay thế danh sách trọng số chọn tay; dùng *toàn bộ* lịch sử, không chỉ một window | Không có khái niệm trend — luôn trễ khi có trend thật, sustained |
| **Double Exp. Smoothing (Holt's)** | Level + Trend | Thêm phương trình thứ hai để theo dõi *tốc độ* di chuyển của level | Không có khái niệm về mùa vụ lặp lại |
| **Triple Exp. Smoothing (Holt-Winters)** | Level + Trend + Seasonality | Thêm phương trình thứ ba để theo dõi chỉ số mùa vụ (seasonal index) lặp lại | Vẫn thuần time-series (không có driver bên ngoài như nhiệt độ/khuyến mãi); cần 2+ mùa đầy đủ trước khi thực sự forecast được (xem Block 4) |

**Ý tưởng xuyên suốt mọi cấp độ:** mỗi phương pháp thực chất là một moving average được ngụy trang — chỉ khác nhau ở cách thông minh và tự động hơn để quyết định *bao nhiêu trọng số* nên dành cho quá khứ. Theo dõi nhiều thành phần hơn = linh hoạt hơn = nhiều quyết định phán đoán hơn (nhiều hằng số cần tinh chỉnh hơn) và cần nhiều lịch sử hơn trước khi bắt đầu được.

---

## Block 1 — Simple Exponential Smoothing (SES)

**Công thức:**

**Forecast_t = α × Actual_(t-1) + (1 − α) × Forecast_(t-1)**

- **α (alpha)** = hằng số smoothing, từ 0 đến 1 — bao nhiêu trọng số dành cho điều vừa xảy ra.
- **(1 − α)** = damping factor — bao nhiêu trọng số giữ lại cho forecast trước đó (vốn đã mã hóa mọi thứ trước nó, theo kiểu đệ quy).

### Mẹo ghi nhớ SES

- 🧠 **"Đệ quy, không phải theo window."** Khác với SMA/WMA, SES không cần tham chiếu một danh sách cố định các giá trị quá khứ thô — nó chỉ cần *actual gần nhất* và *forecast gần nhất*. Đó là điều cho phép nó dùng "toàn bộ lịch sử" mà không cần lưu trữ nó.
- 🧠 **"Exponential" = mở đệ quy ra.** Thay công thức đệ quy vào chính nó nhiều lần, mỗi actual quá khứ cuối cùng được cân bởi α×(1−α)^k, với k = số kỳ về trước. Đó là một trọng số giảm dần theo cấp số nhân (hàm mũ) — không có gì bị bỏ hoàn toàn, nó chỉ mờ dần về (không bao giờ chạm) 0.
- 🧠 **α là nút xoay duy nhất, cùng kiểu đánh đổi như độ dài window/trọng số trước đó:**

| α thấp (vd: 0.1) | α cao (vd: 0.7–0.9) |
|---|---|
| Mượt, phản ứng chậm — hoạt động như SMA window dài | Phản ứng gần như ngay lập tức — gần giống naive forecast (α=1 *chính là* naive forecast) |
| Tin tưởng nhiều vào forecast cũ | Tin tưởng nhiều vào actual mới nhất |

- 🧠 **Điểm yếu quan trọng cần nhớ: SES luôn trễ khi có trend thật, và độ trễ *tăng dần* khi trend tiếp tục** — không chỉ "hơi lệch," sai số cộng dồn mỗi kỳ, vì cả hai thành phần trong công thức (actual cũ, forecast cũ) luôn đi sau một mục tiêu đang di chuyển.

### Ví dụ tính tay — chứng minh độ trễ do trend

Demand tăng đều: 100, 110, 120, 130, 140, 150 (α=0.3, forecast ban đầu=100):

| Kỳ | Actual | Forecast | Sai số |
|---|---|---|---|
| 2 | 110 | 100 | +10 |
| 3 | 120 | 103.0 | +17.0 |
| 4 | 130 | 108.1 | +21.9 |
| 5 | 140 | 114.7 | +25.3 |
| 6 | 150 | 122.3 | +27.7 |

**Sai số không phân tán ngẫu nhiên — nó luôn dương và luôn tăng.** Đó là dấu hiệu của một bias mang tính cấu trúc, không phải noise ngẫu nhiên. Đây chính xác là điều Holt's method khắc phục.

---

## Block 2 — Double Exponential Smoothing (Holt's Method)

**Ba công thức — level, trend, và forecast kết hợp cả hai:**

**Level: L_t = α × Actual_t + (1−α) × (L_(t-1) + T_(t-1))**
**Trend: T_t = β × (L_t − L_(t-1)) + (1−β) × T_(t-1)**
**Forecast (m kỳ tới): F_(t+m) = L_t + m × T_t**

- **α** — cùng vai trò như SES: level phản ứng nhanh đến mức nào.
- **β (beta)** — hằng số thứ hai, độc lập: ước lượng *trend* phản ứng nhanh đến mức nào với thay đổi của trend.

### Mẹo ghi nhớ Holt's

- 🧠 **Phiên bản dễ hiểu: Level = độ cao hiện tại của bạn. Trend = tốc độ leo hiện tại. Forecast = độ cao + (tốc độ × số bước tới).** Mọi công thức trên chỉ là ý tưởng này bằng số.
- 🧠 **Giờ có hai nút xoay, và chúng tương tác với nhau** — β cao kết hợp α thấp sẽ hoạt động khác với từng cái riêng lẻ. Nhiều sức mạnh hơn, cần nhiều phán đoán hơn.

| | Điều khiển | Giá trị thấp | Giá trị cao |
|---|---|---|---|
| **α** | Tốc độ phản ứng của Level | Level cập nhật mượt, chậm | Level cập nhật nhảy, nhanh |
| **β** | Tốc độ phản ứng của ước lượng Trend | Ước lượng trend ổn định, chậm nhận ra tăng/giảm tốc | Ước lượng trend dao động nhanh — có thể phản ứng thái quá với một kỳ nhiễu như thể đó là thay đổi tốc độ thật |

- 🧠 **Khởi tạo cần hai giá trị bắt đầu, không phải một:** L₁ = actual đầu tiên, T₁ = actual thứ hai trừ actual đầu tiên (ước lượng đầu tiên về trend mỗi kỳ). SES chỉ cần một forecast bắt đầu — Holt's cần một *level và độ dốc* bắt đầu, đó là lý do nó cần ít nhất hai điểm dữ liệu trước khi có thể bắt đầu.
- 🧠 **Bằng chứng điều này sửa được bias của SES:** chạy Holt's trên cùng dữ liệu trending từ ví dụ SES ở Block 1, sai số là **0 ở mọi kỳ** (xem bên dưới) — vì forecast giờ di chuyển *cùng* với trend thay vì đuổi theo nó từ một bước phía sau.
- 🧠 **Điều nó vẫn không làm được:** chu kỳ mùa vụ lặp lại. Nó không có cơ chế riêng cho "điều này xảy ra mỗi 12 tháng" — một dao động mùa vụ chỉ trông giống "trend" hoặc "noise" đối với Holt's, tùy vào thời điểm.

### Ví dụ tính tay — demand bình gas Supagas thực tế (có nhiễu), T1–T6

Actual: 200, 215, 225, 245, 255, 270 (thay đổi: +15, +10, +20, +10, +15 — có xu hướng tăng, nhưng không phải một đường leo hoàn hảo). α=0.3, β=0.3. L₁=200, T₁=215−200=15.

| Tháng | Actual | Forecast (L_(t-1)+T_(t-1)) | Sai số |
|---|---|---|---|
| T2 | 215 | 215.00 | 0 |
| T3 | 225 | 230.00 | −5.00 |
| T4 | 245 | 243.05 | +1.95 |
| T5 | 255 | 258.36 | −3.36 |
| T6 | 270 | 271.78 | −1.78 |

**So sánh với sai số của SES trên dữ liệu trending (+10, +17, +21.9, +25.3 — luôn tăng, luôn dương) với Holt's ở đây (0, −5, +1.95, −3.36, −1.78 — nhỏ, dao động giữa dương và âm).** Sự tương phản đó chính là toàn bộ giá trị của việc thêm phương trình trend.

---

## Block 3 — Triple Exponential Smoothing (Holt-Winters)

*(Cùng một phương pháp, hai tên gọi — "Triple Exponential Smoothing" mô tả nó làm gì — làm mượt 3 thành phần; "Holt-Winters" ghi công người phát triển: Holt làm level+trend, Winters thêm seasonality vào.)*

**Bốn công thức — level, trend, chỉ số mùa vụ, và forecast kết hợp cả ba (phiên bản multiplicative — hiệu ứng mùa vụ dưới dạng tỷ lệ):**

**Level: L_t = α × (Actual_t / S_(t-p)) + (1−α) × (L_(t-1) + T_(t-1))**
**Trend: T_t = β × (L_t − L_(t-1)) + (1−β) × T_(t-1)**
**Seasonal: S_t = γ × (Actual_t / L_t) + (1−γ) × S_(t-p)**
**Forecast (m kỳ tới): F_(t+m) = (L_t + m×T_t) × S_(t+m-p)**

- **p** = độ dài mùa vụ (12 cho dữ liệu tháng với seasonality theo năm, 4 cho theo quý).
- **γ (gamma)** = hằng số smoothing thứ ba — *chỉ số mùa vụ* được phép cập nhật nhanh đến mức nào.

### Neo trực giác bằng ngôn ngữ đơn giản

- **Level** = quy mô nền của bạn, **đã loại bỏ yếu tố mùa vụ**.
- **Trend** = quy mô nền đó đang tăng trưởng nhanh đến mức nào, bất kể mùa vụ.
- **Seasonal index** = một hệ số nhân nói rằng "thời điểm cụ thể này trong chu kỳ năm luôn chạy cao/thấp hơn X% so với bình thường" — hoàn toàn tách biệt với việc doanh nghiệp đã lớn tới mức nào.
- **Forecast = (tăng trưởng level về phía trước theo trend) × (áp lại hệ số nhân mùa vụ cho bất kỳ kỳ tương lai nào bạn đang dự đoán).**

### Mẹo ghi nhớ Holt-Winters

- 🧠 **γ thường được giữ nhỏ hơn α.** Hình dạng mùa vụ (vd: "mùa đông luôn chạy cao hơn 20% bình thường") thường được xem là khá ổn định qua từng năm, nên bạn không muốn một mùa bất thường làm lệch chỉ số mùa vụ quá nhanh — γ nhỏ hơn nghĩa là ước lượng mùa vụ cập nhật thận trọng hơn so với level.
- 🧠 **Multiplicative vs. additive seasonality — chọn dựa trên hình dạng của dao động:**

| Multiplicative (S là tỷ lệ) | Additive (S là một lượng cố định) |
|---|---|
| Dùng khi dao động mùa vụ tỷ lệ *theo* mức nền tổng thể ("mùa đông chạy cao hơn 20% bình thường") | Dùng khi dao động mùa vụ giữ nguyên *kích thước* cố định bất kể mức nền lớn tới đâu ("mùa đông luôn chạy +50 đơn vị") |

- 🧠 **Forecast cho một kỳ tương lai dùng lại chỉ số mùa vụ của chu kỳ trước (S_(t+m-p)), không phải của kỳ hiện tại.** Đó là cơ chế cho phép Holt-Winters dự đoán đúng đỉnh mùa vụ *trước khi* nó xảy ra — nó không cần thấy đỉnh năm nay để biết nó sắp tới, vì chỉ số của năm ngoái đã cho biết rồi.

### ⚠️ Cái bẫy khởi tạo — sai lầm dễ mắc nhất với phương pháp này

**Bạn không thể tính ước lượng trend từ ít hơn hai chu kỳ mùa vụ đầy đủ**, vì trend là *tốc độ thay đổi giữa hai điểm thời gian* — bạn không thể đo tốc độ thay đổi từ một điểm duy nhất.

- **Ý nghĩa thực tế:** nếu bạn chỉ có dữ liệu Năm 1, bạn có thể tính seasonal index, nhưng **không thể** tính ước lượng trend, vì điều đó cần so sánh trung bình Năm 1 với trung bình Năm 2.
- **Cái bẫy:** rất dễ nghĩ "tôi có Năm 1, nên tôi có thể forecast Năm 2 từng quý một." Bạn không thể — không phải với một ước lượng trend thật — vì ước lượng trend đó chỉ có thể tính *sau khi* Năm 2 đã xảy ra (trung bình Năm 2 − trung bình Năm 1). Dùng nó để "forecast" Năm 2 sẽ là vòng lặp logic (circular): dự đoán Năm 2 bằng một con số vốn đã cần biết Năm 2.
- **Điều thực sự xảy ra với 2 năm dữ liệu đầy đủ:** toàn bộ giai đoạn đầu đó (Năm 1–2) hoàn toàn không phải là forecast — nó được dùng hoàn toàn như một **giai đoạn hiệu chỉnh/làm nóng (calibration/warm-up)** để cố định level, trend, và seasonal index bắt đầu. **Forecast thật sự, mù thông tin đầu tiên** chỉ khả thi từ Năm 3 trở đi.
- 🧠 **Cách khắc phục nếu bạn chỉ có 1 mùa và muốn bắt đầu sớm hơn:** khởi tạo trend bằng một hồi quy tuyến tính đơn giản của demand theo thời gian, chỉ fit trên một năm đó (simple linear regression ở Section 4 — slope trở thành ước lượng trend bắt đầu). Cách này tránh được vòng lặp logic, với cái giá là ước lượng trend nhiễu hơn, vì các dao động mùa vụ trong một năm đó bị lẫn vào slope thay vì được tách ra rõ ràng.

### Ví dụ tính tay (đã đóng khung đúng — giai đoạn hiệu chỉnh vs. forecast thật sự)

**Dữ liệu:** demand bình gas theo quý, p=4.

| | Q1 | Q2 | Q3 (mùa đông) | Q4 |
|---|---|---|---|---|
| Năm 1 | 200 | 250 | 300 | 250 |
| Năm 2 | 220 | 275 | 330 | 275 |

**Giai đoạn hiệu chỉnh (dùng toàn bộ hindsight của Năm 1–2 — không phải forecast thật):**

- Seasonal index từ Năm 1 (tỷ lệ so với trung bình năm đó, 250): **S₁=0.80, S₂=1.00, S₃=1.20, S₄=1.00**.
- Level bắt đầu L₄ = trung bình Năm 1 = **250**.
- Trend bắt đầu T₄ = (trung bình Năm 2 là 275 − trung bình Năm 1 là 250)/4 = **6.25/quý** — *chỉ tính được bây giờ khi Năm 2 đã biết đầy đủ.*
- Chạy ba phương trình đệ quy qua bốn actual của Năm 2 (220, 275, 330, 275) tinh chỉnh chúng thành các giá trị hiệu chỉnh cuối: **L₈=282.9, T₈=6.96**, và seasonal index cập nhật **S=0.812, 1.004, 1.196, 0.992**.

**Forecast thật sự — Năm 3, thực hiện khi chưa thấy actual nào của Năm 3:**

| Năm 3 | Forecast |
|---|---|
| Q1 | (282.9 + 1×6.96) × 0.812 = **235.4** |
| Q2 | (282.9 + 2×6.96) × 1.004 = **298.0** |
| Q3 (mùa đông) | (282.9 + 3×6.96) × 1.196 = **363.3** |
| Q4 | (282.9 + 4×6.96) × 0.992 = **308.1** |

**Kết quả:** Q3 được forecast cao hơn hẳn Q1 — đỉnh mùa vụ đã được dự đoán đúng *trước khi* nó xảy ra, chỉ dùng seasonal index của chu kỳ trước. Đó là điều mà không phương pháp nào trước đó (từ SMA tới Holt's) từng làm được.

---

## Block 4 — Bảng tổng hợp nhanh (toàn bộ họ, kèm cả MA)

| Phương pháp | Thành phần theo dõi | Hằng số cần tinh chỉnh | Lịch sử tối thiểu cần có | Xử lý trend? | Xử lý seasonality? | Forecast thật sự nhiều kỳ tới được không? |
|---|---|---|---|---|---|---|
| SMA | Level | 0 (chỉ độ dài window n) | n kỳ | ❌ | ❌ | ❌ (chỉ one-step-ahead) |
| WMA | Level | 0 (trọng số, chọn tay) | n kỳ | ❌ | ❌ | ❌ |
| SES | Level | α | 1 kỳ | ❌ | ❌ | ❌ (ngoại suy đường phẳng) |
| Holt's | Level + Trend | α, β | 2 kỳ | ✅ | ❌ | ✅ (ngoại suy đường thẳng) |
| Holt-Winters | Level + Trend + Season | α, β, γ | 2 mùa đầy đủ | ✅ | ✅ | ✅ (ngoại suy có đường cong, nhận biết mùa vụ) |

---

## Block 5 — Tham khảo Excel

| Việc cần làm | Hàm / công cụ |
|---|---|
| Simple Exponential Smoothing | Data Analysis add-in → công cụ **Exponential Smoothing** (input = damping factor, tức 1−α); hoặc thủ công: `=α*B2+(1-α)*C2` |
| Holt's method (level+trend) | Không có công cụ Data Analysis cổ điển riêng — tự xây cột L và T thủ công bằng công thức trên, hoặc dùng `FORECAST.ETS()` (xem bên dưới) |
| Holt-Winters (level+trend+season) | **`FORECAST.ETS()`** — hàm forecasting hiện đại của Excel tự động phát hiện và fit trend + seasonality (một model họ ETS/exponential-smoothing) mà không cần bạn tự xây ba phương trình bằng tay. `FORECAST.ETS.SEASONALITY()` ước lượng độ dài mùa vụ p giúp bạn; `FORECAST.ETS.CONFINT()` cho khoảng tin cậy quanh forecast (kết nối trực tiếp với khái niệm CI/PI ở Section 3). Nút **"Forecast Sheet"** trên ribbon Data là một wrapper point-and-click quanh chính hàm này. |
| Công thức thủ công cho cả ba (nếu muốn toàn quyền kiểm soát, như trong notebook này) | Xây L, T, S thành các cột rõ ràng bằng công thức từ Block 1–3, đúng như đã tính tay ở đây |

---

## Block 6 — Các bẫy phổ biến

1. **"α cao luôn tốt hơn vì phản ứng nhanh hơn."** ❌ Phản ứng nhanh hơn cũng có nghĩa nhiều noise lọt qua hơn — α=1 chính là naive forecast, thứ bạn đã biết là baseline tệ.
2. **"Holt's method cuối cùng sẽ bắt kịp trend và giữ được vị trí bắt kịp."** ❌ Chỉ đúng hoàn toàn với một trend hằng số hoàn hảo. Trend thật luôn dao động, nên Holt's luôn đuổi theo một chút — chỉ ít hơn nhiều so với SES.
3. **"Tôi có thể bắt đầu forecast bằng Holt-Winters ngay khi có một mùa dữ liệu."** ❌ Bạn cần một mùa *thứ hai* đầy đủ chỉ để tính một ước lượng trend — dùng trung bình của năm "tương lai" để khởi tạo một giá trị dùng để forecast chính năm đó là vòng lặp logic (đúng sai lầm đã bị phát hiện và sửa ở Block 3).
4. **"Theo dõi nhiều thành phần hơn (Holt-Winters) luôn là lựa chọn an toàn hơn."** ❌ Nếu mẫu hình thật không có trend hay seasonality thật sự, một model phức tạp hơn có nguy cơ fit vào noise như thể đó là mẫu hình thật — cùng rủi ro overfitting đã cảnh báo với R² ở Section 4. Độ phức tạp nên được biện minh bằng bằng chứng (autocorrelation, kiểm định giả thuyết trên hệ số hồi quy, hoặc so sánh sai số forecast giữa các phương pháp ứng viên), không phải mặc định vì "nghe có vẻ mạnh hơn."
5. **"γ nên được tinh chỉnh giống như α."** ❌ Vì hình dạng mùa vụ thường ổn định hơn qua từng năm so với level hiện tại, γ thường được giữ nhỏ hơn α, để một mùa bất thường không làm lệch quá nhiều seasonal index của năm sau.

---

**Tiếp theo trong lộ trình:** Forecast Accuracy & Uncertainty (MAPE, MAD, RMSE) — công cụ thực sự trả lời câu hỏi "phương pháp nào trong số này hoạt động tốt nhất trên dữ liệu *của tôi*," thay vì chỉ đoán từ lý thuyết.

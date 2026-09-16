# Forecasting Notes — Họ Moving Average (SMA, WMA, và các biến thể khác)

*Notebook đồng hành với notes Section 4 (Correlation & Regression) — đây là block đầu tiên trong lộ trình forecasting. Bao gồm mọi biến thể "moving average" đáng biết, không chỉ hai loại đã học tương tác, kèm mẹo ghi nhớ, ví dụ tính tay, và bảng tổng hợp nhanh ở cuối.*

---

## Block 0 — Nền tảng cần nhớ lại (ý tưởng gốc mà mọi biến thể bên dưới đều dựa vào)

Dữ liệu demand lịch sử = **signal** (mức nền thực sự, có thể kèm trend/seasonality) + **noise** (biến động ngẫu nhiên không có giá trị dự đoán). **Lấy trung bình (averaging) giúp triệt tiêu noise** vì các biến động ngẫu nhiên lên/xuống phần nào bù trừ lẫn nhau khi cộng lại. Mọi phương pháp "moving average" bên dưới chỉ là một cách khác nhau để chọn **giá trị nào** để lấy trung bình và **trọng số (weight)** bao nhiêu cho mỗi giá trị — đó là toàn bộ không gian thiết kế.

**Đánh đổi phổ quát cần nhớ, dù là biến thể nào:** càng smoothing nhiều (window lớn hơn / dữ liệu cũ được cân nhiều hơn) → ít noise hơn, nhưng phản ứng chậm hơn với thay đổi thật. Càng ít smoothing (window nhỏ / dữ liệu gần đây được cân nhiều hơn) → phản ứng nhanh hơn, nhưng để lọt nhiều noise hơn. **Không có cách nào có cả hai miễn phí — mọi lựa chọn ở đây là một cái nút xoay (dial), không phải một cách "sửa triệt để."**

---

## Block 1 — Simple Moving Average (SMA)

**Định nghĩa:** trung bình của **n** kỳ (period) gần nhất, mỗi kỳ được tính **bằng nhau (equal weight)**.

**Công thức:**

**SMA_t = (X_(t-1) + X_(t-2) + ... + X_(t-n)) / n**

Forecast cho kỳ tiếp theo = giá trị trung bình này.

### Mẹo ghi nhớ SMA

- 🧠 **"Cũ rớt ra, mới thêm vào" (Oldest drops, newest joins).** Mỗi lần cửa sổ (window) dịch tới một kỳ, bạn bỏ giá trị cũ nhất trong window và thêm giá trị thực tế mới nhất — window luôn có đúng n giá trị.
- 🧠 **"Cần n kỳ trước khi có forecast đầu tiên."** Bạn không thể tạo forecast cho tới khi có đủ n kỳ dữ liệu thực tế. Một MA 12 tháng cần đủ một năm dữ liệu sạch mới cho ra kết quả — đây là cái giá trực tiếp của việc chọn window dài.
- 🧠 **"Một cú nhảy mức (level shift) cần đúng n kỳ để hấp thụ hoàn toàn."** Nếu demand nhảy lên một mức mới và duy trì ở đó, forecast SMA sẽ **không** bắt kịp ngay — nó cần n kỳ để mọi giá trị cũ trong window được thay thế bằng mức mới. **Đây là điều quan trọng nhất cần nhớ về SMA**, vì rất dễ nghĩ nhầm "MA phản ứng ngay trong phạm vi window" trong khi thực ra nó chỉ phản ứng dần dần suốt cả window.
- 🧠 **Độ dài window là một sự đánh đổi, không phải một con số "đúng":**

| n ngắn hơn (vd: 3) | n dài hơn (vd: 12) |
|---|---|
| Phản ứng nhanh với thay đổi thật | Phản ứng chậm — trễ so với thay đổi thật |
| Vẫn còn nhiễu/nhảy | Rất mượt |
| Cần ít dữ liệu lịch sử hơn để bắt đầu | Cần nhiều dữ liệu lịch sử hơn để bắt đầu |

- 🧠 **SMA trong thực tế chỉ là phương pháp one-step-ahead (dự đoán 1 kỳ tới).** Về mặt kỹ thuật bạn có thể copy công thức xuống xa hơn, nhưng qua khỏi kỳ forecast đầu tiên, nó bắt đầu lấy trung bình của chính các forecast trước đó thay vì dữ liệu thực — sách gốc gọi đây là "kéo căng quá mức" ("stretching it too far"), và chất lượng giảm nhanh. Đừng dùng SMA để forecast nhiều kỳ về sau.
- 🧠 **Điều nó không thể làm được về mặt cấu trúc:** trend và seasonality. Vì mỗi kỳ trong window được tính ngang nhau, một trend tăng thật sự luôn khiến SMA **under-forecast**, và một trend giảm thật sự luôn khiến nó **over-forecast** — một cách có thể dự đoán trước, lần nào cũng vậy, không phải do xui rủi.

### Ví dụ tính tay — Demand bình gas Supagas, SMA 3 tháng

| Tháng | Thực tế | SMA 3 tháng (= forecast cho tháng sau) |
|---|---|---|
| T1 | 500 | — |
| T2 | 540 | — |
| T3 | 480 | — |
| T4 | 610 | (500+540+480)/3 = **506.7** |
| T5 | 550 | (540+480+610)/3 = **543.3** |
| T6 | 520 | (480+610+550)/3 = **546.7** |

**Chứng minh "level shift" (tại sao "cần n kỳ để hấp thụ hoàn toàn" quan trọng):** nếu T7 nhảy lên mức mới 700 và duy trì:

| Tháng | Thực tế | Forecast SMA 3 tháng |
|---|---|---|
| T8 | 700 | (550+520+700)/3 = **590** |
| T9 | 700 | (520+700+700)/3 = **640** |
| T10 | 700 | (700+700+700)/3 = **700** ✅ (mất đúng 3 tháng = n) |

---

## Block 2 — Weighted Moving Average (WMA)

**Điều nó khắc phục so với SMA:** SMA tính giá trị 3 tháng trước ngang bằng tháng trước — nhưng dữ liệu gần đây gần như luôn mang nhiều thông tin hơn. WMA gán **trọng số lớn hơn cho các kỳ gần đây**.

**Công thức:**

**WMA_t = (w₁·X_(t-1) + w₂·X_(t-2) + ... + wₙ·X_(t-n)) / (w₁ + w₂ + ... + wₙ)**

### Mẹo ghi nhớ WMA

- 🧠 **"Trọng số lớn nhất luôn dành cho kỳ gần đây nhất."** Nếu không chắc đầu nào của danh sách trọng số là "gần đây," đây là quy tắc — làm ngược lại sẽ khiến forecast *tệ hơn* cả SMA, không chỉ là kém hiệu quả hơn (đã chứng minh ở ví dụ bên dưới).
- 🧠 **Luôn chia cho tổng các trọng số**, không phải chia cho n. Nếu trọng số là 3, 2, 1 (không cần tổng bằng 1), bạn chia cho 6 — đây là điều giữ nó là một *trung bình* có trọng số thực sự thay vì một tổng bị phóng đại. Nếu bạn chọn trọng số đã tổng bằng 1 sẵn (như 0.5, 0.3, 0.2), có thể bỏ qua bước chia — nó đã được tính sẵn.
- 🧠 **Các cách chọn trọng số phổ biến** bạn sẽ gặp trong thực tế:
  - **Giảm dần tuyến tính (linear decreasing)** — vd: 3/2/1 (kỳ gần nhất có trọng số gấp 3 lần kỳ cũ nhất). Đơn giản, dễ giải thích cho stakeholder.
  - **Theo phần trăm** — vd: 50%/30%/20%, chọn theo kinh nghiệm hoặc bằng cách thử nghiệm để tìm tỷ lệ cho sai số forecast lịch sử thấp nhất.
  - Không có công thức nào cho bạn biết trọng số "đúng" — **việc chọn trọng số là một quyết định mang tính phán đoán (judgment call)**, và đó chính xác là điểm yếu của WMA (xem bên dưới).
- 🧠 **Cái giá của việc phản ứng nhanh hơn: nhiều noise lọt vào hơn.** Vì WMA dựa nhiều nhất vào dữ liệu mới nhất, một tháng gần đây bất thường (lỗi dữ liệu, một đơn hàng số lượng lớn một lần) giờ có ảnh hưởng quá mức lên forecast — bạn đã đánh đổi bớt khả năng triệt tiêu noise của SMA để lấy độ phản ứng nhanh. Không cái nào "tốt hơn" tuyệt đối — nó tùy vào ngữ cảnh.

### Ví dụ tính tay — cùng dữ liệu, trọng số 3(gần nhất)/2/1(cũ nhất)

Forecast cho T8, dùng T5=550, T6=520, T7=700:

**WMA_T8 = (3×700 + 2×520 + 1×550) / (3+2+1) = (2100+1040+550) / 6 = 3690/6 = 615**

So với SMA thường là 590 trên cùng ba tháng — WMA đã gần với mức mới hơn ngay từ *kỳ đầu tiên*, vì T7 được tính gấp 3 lần T5.

### ⚠️ Cái bẫy — trọng số bị đảo ngược khiến mọi thứ tệ hơn, không phải trung tính

Nếu bạn vô tình gán trọng số nặng nhất cho tháng **cũ nhất** (1-gần nhất/2/3-cũ nhất) thay vì tháng gần nhất:

**WMA_T8 = (1×700 + 2×520 + 3×550) / 6 = (700+1040+1650)/6 = 3390/6 = 565**

565 *xa hơn* mức mới thực sự (700) so với cả SMA thường (590). **Đặt sai hướng trọng số không chỉ là không giúp ích — nó chủ động làm kém hơn cả phương pháp đơn giản hơn.** Luôn kiểm tra lại đầu nào của danh sách trọng số khớp với "gần đây nhất" trước khi tin vào kết quả.

---

## Block 3 — Các biến thể khác trong họ Moving Average (theo yêu cầu — danh sách đầy đủ)

Các phương pháp này ít được dùng hàng ngày cho demand forecasting hơn SMA/WMA, nhưng mỗi cái giải quyết một vấn đề cụ thể và đáng để biết là chúng tồn tại.

### 3a. Double Moving Average (DMA) — "MA-của-MA," cách sửa độ trễ trend của SMA

**Vấn đề nó nhắm tới:** bạn đã chứng minh SMA under/over-forecast một trend thật, một cách có thể dự đoán trước. Double Moving Average sửa điều này **mà không cần rời khỏi họ moving average** — đây là một phương pháp cổ điển thay thế cho Holt's method, chỉ dùng các phép trung bình.

**Cách hoạt động — lấy trung bình của trung bình:**
1. Tính **M1_t** = một SMA n-kỳ bình thường trên dữ liệu thô (giống hệt Block 1).
2. Tính **M2_t** = một SMA n-kỳ trên chính chuỗi M1 (lấy trung bình của các trung bình).
3. Kết hợp chúng để suy ra một mức (level) đã khử độ trễ và một ước lượng trend rõ ràng:

**Level: a_t = 2×M1_t − M2_t**
**Trend (mỗi kỳ): b_t = (2/(n−1)) × (M1_t − M2_t)**
**Forecast, m kỳ tới: F_(t+m) = a_t + b_t × m**

- 🧠 **Mẹo ghi nhớ:** M2 luôn trễ hơn M1 theo cùng cách M1 trễ hơn dữ liệu thô. Khoảng cách giữa chúng (M1 − M2) chính là một phép đo "độ trễ/trend đang bị che giấu bao nhiêu" — đó là lý do vì sao nhân đôi khoảng cách của M1 tới M2 rồi trừ đi (2×M1 − M2) sẽ khôi phục một level đã sửa trend, thay vì level đang bị trễ.
- **Tại sao điều này quan trọng về mặt khái niệm:** đây chính xác là cùng ý tưởng với Holt's method (theo dõi level và trend riêng biệt) — chỉ khác là dùng phép trung bình đơn giản thay vì trọng số kiểu exponential. Nên biết nó tồn tại như một cầu nối, nhưng Holt's method (đã học) thường được ưu tiên hơn trong thực tế vì cần ít dữ liệu lịch sử hơn và phản ứng mượt hơn.

### 3b. Centered Moving Average (CMA) — dùng để đo lường, không phải để forecast

**Vấn đề nó nhắm tới:** SMA như định nghĩa ở trên chỉ nhìn **về phía sau** (dùng n kỳ trước để ước lượng "hiện tại"), điều này ổn cho việc forecast tương lai nhưng lại tạo ra độ trễ khi mục tiêu của bạn thay vào đó là **đo lường trend-cycle thực sự tại một điểm đã quan sát được** — ví dụ: để sau đó tính một **seasonal index (chỉ số mùa vụ)** (đây chính xác là kỹ thuật đứng sau phương pháp phân rã mùa vụ cổ điển, và là bản xem trước của việc Holt-Winters đang làm bên trong).

**Điểm khác biệt:** thay vì chỉ lấy trung bình n kỳ *trước* thời điểm t, bạn lấy trung bình các kỳ **xung quanh (centered around)** t — dùng cả dữ liệu trước và sau. Với n chẵn (như 12, cho dữ liệu theo tháng), cần thêm một bước nhỏ (lấy trung bình hai giá trị n-kỳ trung bình liên tiếp) để rơi đúng vào một kỳ thời gian thực thay vì nằm giữa hai kỳ.

- 🧠 **Mẹo ghi nhớ:** CMA cần **dữ liệu tương lai so với điểm bạn đang tính** — vì vậy **bạn không thể dùng nó để forecast thời gian thực**. Đây là công cụ **đo lường/phân tích**, dùng sau khi đã có dữ liệu để trích xuất một đường trend-cycle sạch hoặc để tính seasonal index từ dữ liệu lịch sử — không phải phương pháp forecast trực tiếp. Đừng nhầm nó với SMA khi ai đó nói "moving average" trong ngữ cảnh mùa vụ.

### 3c. Cumulative Moving Average — "trung bình chạy" (running average)

**Định nghĩa:** trung bình của **toàn bộ dữ liệu kể từ điểm bắt đầu** cho tới kỳ hiện tại — window không bao giờ ngừng mở rộng.

**Công thức:** CMA_t = (X_1 + X_2 + ... + X_t) / t

- 🧠 **Mẹo ghi nhớ:** khi t càng lớn, mỗi điểm dữ liệu *mới* càng có ảnh hưởng nhỏ hơn lên trung bình (vì chia cho t ngày càng lớn) — vì vậy nó trở nên **cực kỳ mượt nhưng cực kỳ chậm phản ứng**, là đầu cực đoan nhất của phổ đánh đổi "window dài." Hiếm khi hữu ích cho demand forecasting cụ thể (một doanh nghiệp thực tế hầu như luôn quan tâm hành vi gần đây hơn là trung bình toàn bộ lịch sử), nhưng bạn sẽ gặp nó trong các ngữ cảnh như theo dõi tỷ lệ lỗi trung bình chạy hoặc điểm số trung bình lũy kế.

### 3d. Exponentially Weighted Moving Average (EWMA) — cầu nối tới chủ đề tiếp theo của bạn

Đáng để biết cái tên này tồn tại: **EWMA về mặt toán học chính là Simple Exponential Smoothing** (đã học ở phiên trước) — mọi giá trị quá khứ được gán một trọng số giảm dần theo kiểu hàm mũ (exponential) tùy vào nó xa bao nhiêu, được tạo ra bởi một hằng số smoothing duy nhất thay vì một danh sách trọng số chọn sẵn bằng tay. Một số sách xếp nó vào "họ moving average" (vì về cơ bản nó vẫn là một trung bình có trọng số của các giá trị quá khứ) thay vì là một loại riêng — đừng để cái tên khác nhau làm bạn nhầm lẫn nếu gặp nó ở nơi khác được gọi tên này. Chi tiết đầy đủ nằm trong notes Exponential Smoothing, không nhắc lại ở đây.

---

## Block 4 — Bảng tổng hợp nhanh để ghi nhớ

| Phương pháp | Điểm khác biệt | Phản ứng với thay đổi thật | Xử lý trend? | Xử lý seasonality? | Cần dữ liệu tương lai? |
|---|---|---|---|---|---|
| **SMA** | Trọng số bằng nhau, window cố định n | Chậm (trễ n kỳ khi có level shift) | ❌ Không — luôn trễ | ❌ Không | Không |
| **WMA** | Trọng số tùy chỉnh, gần đây được cân nhiều hơn | Nhanh hơn (nếu cân đúng hướng) | ❌ Không — vẫn trễ, chỉ ít hơn | ❌ Không | Không |
| **Double MA** | MA-của-MA, có ước lượng trend rõ ràng | Nhanh hơn, đã sửa trend | ✅ Có | ❌ Không | Không |
| **Centered MA** | Lấy trung bình cả trước *và* sau điểm đó | Không áp dụng — không phải phương pháp forecast | — | Dùng để giúp *đo lường* nó | ✅ Có (đó là lý do nó không thể forecast thời gian thực) |
| **Cumulative MA** | Window mở rộng mãi từ kỳ 1 | Cực kỳ chậm | ❌ Không | ❌ Không | Không |
| **EWMA (= Simple Exp. Smoothing)** | Trọng số giảm theo hàm mũ, một hằng số (α) | Tùy chỉnh được qua α | ❌ Không (xem Holt's cho việc đó) | ❌ Không | Không |

---

## Block 5 — Tham khảo Excel

| Việc cần làm | Hàm / cách làm |
|---|---|
| Simple Moving Average | `=AVERAGE(range)`, copy xuống; hoặc Data Analysis add-in → công cụ **Moving Average** |
| SMA linh hoạt độ dài (đổi window mà không cần viết lại công thức) | `=AVERAGE(OFFSET($A2,0,0,$C$1,1))` — $C$1 chứa độ dài window n |
| Weighted Moving Average | Không có hàm dựng sẵn — dùng `=SUMPRODUCT(vùng_trọng_số, vùng_giá_trị)/SUM(vùng_trọng_số)` |
| Cumulative Moving Average | `=AVERAGE($A$2:A2)`, copy xuống (tham chiếu hỗn hợp `$A$2` khóa điểm bắt đầu, `A2` mở rộng khi copy) |
| Centered Moving Average | Tạo cột SMA trước, sau đó lấy trung bình mỗi cặp giá trị SMA liên tiếp (hoặc dùng công thức trung bình dịch chuyển) — thường làm thủ công trong cột phụ vì Excel không có công cụ trực tiếp cho việc này |

---

## Block 6 — Các bẫy phổ biến (đối chiếu lại với sự hiểu của bạn)

1. **"MA 3 tháng phản ứng với level shift trong vòng 3 tháng, hoàn toàn."** ❌ Sai — nó phản ứng *dần dần*, cần đủ n kỳ để hấp thụ hoàn toàn thay đổi (xem bảng T8→T10 ở Block 1).
2. **"Cân dữ liệu gần đây nhiều hơn luôn khiến WMA tốt hơn SMA."** ❌ Sai — chỉ đúng nếu bạn cân *đúng hướng*. Trọng số bị đảo ngược khiến WMA chủ động tệ hơn SMA (cái bẫy ở Block 2).
3. **"Tôi có thể dùng moving average để forecast nhiều tháng tới bằng cách copy công thức xuống xa hơn."** ❌ Sai — SMA/WMA là phương pháp one-step-ahead; kéo xa hơn, chúng bắt đầu lấy trung bình các forecast thay vì dữ liệu thực và chất lượng giảm nhanh.
4. **"Centered Moving Average chỉ là một cách khác để forecast."** ❌ Sai — nó cần dữ liệu từ *sau* điểm bạn đang tính, nên không thể dùng thời gian thực. Đây là công cụ phân tích, thường dùng để trích xuất seasonal index từ lịch sử.
5. **"Window dài hơn luôn 'chính xác hơn.'"** ❌ Sai — nó mượt hơn, không phải chính xác hơn. Mượt hơn giúp ích khi dữ liệu của bạn chủ yếu là noise; nó gây hại khi có thay đổi trend/level thật đang xảy ra, vì nó trễ theo những thay đổi đó lâu hơn.

---

**Tiếp theo:** Triple Exponential Smoothing (Holt-Winters) — tiếp tục ngay từ chỗ đã dừng, giờ đã có toàn bộ bức tranh moving average này làm nền tảng vững chắc bên dưới.

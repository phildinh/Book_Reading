# Forecasting Notes — Playbook Chọn Phương Pháp & Kết Hợp (Tổng Quan)

*Notebook đồng hành với MA/WMA, Exponential Smoothing, và Diagnostic Tools. Đây là phần tổng hợp: với dữ liệu thực tế của một sản phẩm, làm sao bạn thực sự quyết định dùng phương pháp nào, và làm sao kết hợp các phương pháp thay vì chỉ chọn một? Đây là bản **tổng quan** — khái niệm, quy tắc quyết định, mẹo ghi nhớ. Bản **chi tiết** (kèm ví dụ tính tay đầy đủ) sẽ đi kèm riêng sau.*

---

## Block 0 — Thay đổi tư duy cốt lõi

Mỗi notebook trước đó dạy bạn **một phương pháp tại một thời điểm, riêng lẻ**. Công việc forecasting thực tế thì ngược lại: bạn hiếm khi biết trước phương pháp nào phù hợp, và gần như không bao giờ chỉ dựa vào một phương pháp duy nhất. Playbook này nói về **chính quy trình ra quyết định** — chẩn đoán → chọn phương pháp → kết hợp → giám sát — không phải một công thức mới.

🧠 **Câu duy nhất cần nhớ: chọn phương pháp mà bằng chứng ủng hộ, không phải phương pháp bạn thích nhất hay mạnh nhất có sẵn.** Độ phức tạp không được biện minh bằng bằng chứng là một cái giá phải trả (nhiều hằng số cần tinh chỉnh hơn, cần nhiều lịch sử hơn, rủi ro overfitting cao hơn), không phải một nâng cấp miễn phí.

---

## Block 1 — Quy trình 6 bước (quy trình vận hành của bạn)

1. **Kiểm tra độ dài lịch sử.** Không có lịch sử nào → chuyển thẳng sang forecast theo phán đoán (sản phẩm tương tự, ước lượng từ đội sales) cho tới khi có dữ liệu thật tích lũy.
2. **Chạy các công cụ chẩn đoán** (từ notebook Diagnostic Tools): ACF ở lag-1 và lag-p, p-value của regression-theo-thời-gian cho trend, ANOVA giữa các nhóm mùa vụ cho seasonality.
3. **Để chẩn đoán + độ dài lịch sử chọn (các) phương pháp ứng viên** — xem bảng quyết định bên dưới.
4. **Fit 2–3 ứng viên, giữ lại actual gần nhất, so sánh sai số** trên dữ liệu mà phương pháp chưa từng thấy (các chỉ số độ chính xác forecast — notebook tiếp theo). Đừng chỉ tin một phương pháp vì giả định.
5. **Áp dụng lớp điều chỉnh theo phán đoán (judgmental overlay)** — kết quả thống kê là một *baseline*, không bao giờ là câu trả lời cuối cùng. Con người biết về tương lai (hợp đồng mới, khuyến mãi, tạm dừng sản xuất) mà không model lịch sử nào có thể thấy được.
6. **Chẩn đoán lại định kỳ.** Thêm lịch sử có thể tiết lộ một mẫu hình mà trước đây không phát hiện được (vd: vượt qua ngưỡng 2 mùa đầy đủ cần cho Holt-Winters).

🧠 **Mẹo:** bước 1–3 trả lời "phương pháp nào đủ điều kiện." Bước 4 trả lời "trong số các phương pháp đủ điều kiện, cái nào thực sự hoạt động tốt nhất trên dữ liệu của tôi." Đừng bao giờ nhảy thẳng từ bước 1 sang bước 5 — bằng chứng trước, rồi mới đến phán đoán con người, theo đúng thứ tự đó.

---

## Block 2 — Bảng quyết định

| Kết quả chẩn đoán | Lịch sử có sẵn | Phương pháp |
|---|---|---|
| Không có trend thật, không có seasonality thật | Bất kỳ | SMA hoặc SES |
| Trend thật, không có seasonality thật | ≥2 kỳ | Holt's method |
| Trend thật + seasonality thật | ≥2 mùa đầy đủ | Holt-Winters |
| Nghi ngờ có trend + seasonality | <2 mùa đầy đủ | Chưa thể chạy Holt-Winters — dùng Holt's + điều chỉnh mùa vụ thủ công thô, hoặc chờ thêm dữ liệu |
| Có driver bên ngoài mạnh, hợp lý về nhân quả | Bất kỳ | Regression — riêng lẻ hoặc kết hợp với một phương pháp time-series |

🧠 **Mẹo — "cổng điều kiện" luôn cần kiểm tra trước:** Holt-Winters chỉ *đủ điều kiện* khi bạn vượt qua yêu cầu lịch sử 2-mùa-đầy-đủ (cái bẫy khởi tạo từ notebook Exponential Smoothing). Đừng để "dữ liệu trông có vẻ theo mùa" lấn át "tôi chưa đủ lịch sử để chứng minh hoặc dùng nó an toàn."

---

## Block 3 — Bốn chiến lược kết hợp (phần quan trọng)

| Loại kết hợp | Kết hợp cái gì | Tại sao nó hiệu quả |
|---|---|---|
| **Thống kê + Phán đoán** (S&OP consensus forecast) | Baseline của model + kiến thức của con người về tương lai | Model chỉ bao giờ thấy quá khứ; con người có thể thêm thông tin mà dữ liệu về cấu trúc không thể chứa (hợp đồng mới, tạm dừng sản xuất) |
| **Ensemble** | Hai hoặc nhiều forecast thống kê, lấy trung bình | Các phương pháp khác nhau tạo ra các loại sai số khác nhau; lấy trung bình phần nào triệt tiêu chúng — thường vượt qua cả model "tốt nhất" đơn lẻ |
| **Time-series + Regression** | Mẫu hình nội tại (trend/season) + một driver bên ngoài | Nắm bắt cả "demand tự nó làm gì" và "một yếu tố bên ngoài đang tác động gì lên nó" |
| **Phân đoạn (Segmented)** | Các phương pháp khác nhau cho các sản phẩm khác nhau, có chủ đích | Không phải SKU nào cũng xứng đáng với cùng mức độ nỗ lực forecasting — khớp độ phức tạp model với mức độ quan trọng kinh doanh |

🧠 **Mẹo — điều phản trực giác cần nhớ:** bạn không cần tìm "phương pháp tốt nhất duy nhất." **Lấy trung bình hai phương pháp tầm trung thường vượt qua việc chọn một phương pháp tốt nhất**, vì sai số của chúng có xu hướng chỉ theo hướng khác nhau và phần nào triệt tiêu lẫn nhau. Đây là một phát hiện đã được ghi nhận trong thực tiễn forecasting, không chỉ là một lối tắt khi bạn không chắc chắn.

🧠 **Mẹo — nỗ lực tương xứng:** một SKU giá trị cao, khối lượng lớn xứng đáng với Holt-Winters + regression + giám sát chặt chẽ. Một SKU ổn định, giá trị thấp thì dùng SMA đơn giản là đủ — khớp nỗ lực với mức độ quan trọng chính là một phần của việc làm tốt việc này, không phải một lối tắt bạn đang chọn.

---

## Block 4 — Toàn bộ phả hệ, một lần nữa, giờ là một bản đồ duy nhất

```
SMA → WMA → SES → Holt's → Holt-Winters
   (trọng số bằng nhau → trọng số tùy ý → tự giảm dần → +trend → +seasonality)

        ↓ được chẩn đoán và chọn bởi ↓

  ACF (Công cụ 1)  +  Kiểm định giả thuyết Regression/ANOVA (Công cụ 2)

        ↓ không bao giờ tin dùng một mình — luôn hoàn thiện bằng ↓

  Lớp điều chỉnh phán đoán  +  có thể kết hợp ensemble/regression
        ↓
  Forecast đồng thuận cuối cùng
```

---

## Block 5 — Các bẫy phổ biến

1. **"Phương pháp mạnh nhất (Holt-Winters) luôn là lựa chọn mặc định an toàn nhất."** ❌ Độ phức tạp không được biện minh có nguy cơ overfitting và cần lịch sử mà bạn có thể chưa có.
2. **"Tôi đã tìm ra phương pháp tốt nhất — không cần xem xét cái khác nữa."** ❌ Kết hợp (blending) thường vượt qua hẳn "phương pháp tốt nhất đơn lẻ."
3. **"Forecast thống kê là con số cuối cùng."** ❌ Nó là một baseline. Lớp điều chỉnh phán đoán là một bước bắt buộc, không phải một điều tốt nếu có thêm.
4. **"Mọi SKU của tôi đều xứng đáng với cùng mức độ chặt chẽ forecasting."** ❌ Nỗ lực nên tỷ lệ với mức độ quan trọng kinh doanh, không nên áp dụng đồng đều.

---

**Tiếp theo:** bản chi tiết đồng hành — cùng cấu trúc, kèm ví dụ tính tay đầy đủ áp dụng khung này vào các tình huống thực tế của Supagas. Sau đó: Forecast Accuracy & Uncertainty (MAPE/MAD/RMSE), công cụ vận hành Bước 4 ở trên.

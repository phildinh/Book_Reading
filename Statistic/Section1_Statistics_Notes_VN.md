# Section 1 — Giới thiệu về Thống kê: Ghi chú tham khảo

*Được xây dựng từ slide bài giảng, chỉnh sửa và mở rộng qua quá trình hỏi-đáp. Tư duy nền tảng trước tiên — mọi khái niệm đều quay về câu hỏi: "tôi có thể tin con số này không, hay cần xem xét kỹ hơn?"*

---

## Khối 1 — Thống kê là gì? Thống kê mô tả và Thống kê suy luận

**Thống kê (Statistics)** là ngành học về việc suy luận từ dữ liệu trong điều kiện không chắc chắn. Khác với **toán học (mathematics)** — vốn xử lý các mối quan hệ chính xác, luôn đúng (2 + 2 = 4, không có ngoại lệ) — thống kê làm việc với dữ liệu thực tế vốn luôn biến động, và nhiệm vụ của nó là đưa ra kết luận đáng tin cậy dù dữ liệu có dao động.

**Thống kê mô tả (Descriptive statistics)** tóm tắt dữ liệu bạn đã có: trung bình (mean), trung vị (median), yếu vị (mode), độ phân tán (spread), và hình dạng phân phối của một tập dữ liệu. *Ví dụ: "Doanh số trung bình mỗi ngày năm ngoái là 500 đô la."*

**Thống kê suy luận (Inferential statistics)** sử dụng một mẫu (sample) để đưa ra nhận định về một tổng thể (population) lớn hơn hoặc một tương lai chưa biết, và thể hiện trung thực mức độ không chắc chắn của nhận định đó. *Ví dụ: "Nhu cầu tháng tới có khả năng nằm trong khoảng 480–520 đơn vị, với độ tin cậy 95%."*

**Tại sao điều này quan trọng:** không ai có kiến thức hoàn hảo về tương lai. Dự báo (forecasting) tồn tại chính vì khoảng cách giữa những gì bạn biết (một mẫu dữ liệu lịch sử) và những gì bạn cần quyết định (tương lai) — khoảng cách này chính là lý do công việc lập kế hoạch nhu cầu (demand planning) tồn tại.

---

## Khối 2 — Loại dữ liệu & Thang đo

Dữ liệu có thể là **định tính (qualitative)** (danh mục, không phải số — ví dụ: loại sản phẩm) hoặc **định lượng (quantitative)** (số, có thể đo lường — ví dụ: số lượng bán ra).

Dữ liệu số được chia thành bốn **thang đo (measurement scales)**:

- **Định danh (Nominal)** — nhãn/định danh không có thứ tự. *Ví dụ: mã SKU.* Ngay cả khi nhãn là một con số, nó cũng không phải là một đại lượng.
- **Thứ bậc (Ordinal)** — có thứ tự, nhưng khoảng cách giữa các hạng không đảm bảo bằng nhau. *Ví dụ: mức độ ưu tiên giao hàng (Thấp/Trung bình/Cao).*
- **Khoảng (Interval)** — dạng số với khoảng cách đều nhau, nhưng **không có điểm 0 thực sự** — số 0 chỉ là một điểm mốc tham chiếu, không có nghĩa là "không có gì". *Ví dụ: nhiệt độ theo độ C; bạn không thể nói 20°C "nóng gấp đôi" 10°C.*
- **Tỷ lệ (Ratio)** — dạng số với khoảng cách đều nhau **và** có điểm 0 thực sự nghĩa là "không có gì". *Ví dụ: số tháng dự trữ (months of cover), doanh thu, cân nặng.* Câu "gấp đôi" có ý nghĩa thực sự ở đây.

**Tại sao điều này quan trọng:** thang đo quyết định phép thống kê nào thực sự hợp lệ để tính trên cột dữ liệu đó. Bạn có thể tính trung bình một cách có ý nghĩa cho dữ liệu doanh số (ratio), nhưng tính trung bình cho điểm hài lòng thang 1–5 (ordinal) về mặt kỹ thuật giả định các khoảng cách bằng nhau giữa các hạng — điều không được đảm bảo.

---

## Khối 3 — Xu hướng trung tâm: Trung bình, Trung vị, Yếu vị

**Xu hướng trung tâm (central tendency)** trả lời câu hỏi "giá trị điển hình là gì?" — và đây chính là hạt giống của dự báo: một dự báo bằng trung bình trượt (moving average) chỉ đơn giản là "dùng xu hướng trung tâm của dữ liệu gần đây làm dự đoán tiếp theo."

**Trung bình (Mean)** = tổng ÷ số lượng. Đơn giản, nhưng nhạy cảm với giá trị ngoại lai (outliers) và với các tổng thể bị trộn lẫn — nó có thể cho ra một giá trị không phản ánh thực tế nào cả. *Ví dụ: trộn 8 đơn hàng bán lẻ nhỏ (1–2 bình) với 5 đơn hàng sỉ lớn (38–45 bình) cho ra trung bình ≈16.5 — một kích cỡ mà không đơn hàng thực tế nào trong ngày đó có.*

**Trung vị (Median)** = giá trị nằm giữa khi dữ liệu được sắp xếp. Ít bị ảnh hưởng bởi giá trị ngoại lai. Khi trung bình ≈ trung vị, đó là dấu hiệu tốt cho thấy dữ liệu khá đối xứng và một con số có thể mô tả tốt cho nó; khi hai giá trị này lệch nhau, đó là dấu hiệu số học của **độ lệch (skew)** (xem Khối 6).

**Yếu vị (Mode)** = giá trị xuất hiện nhiều nhất. Đây là thước đo xu hướng trung tâm duy nhất hợp lệ cho dữ liệu định tính. *Ví dụ: lý do hủy đơn hàng phổ biến nhất.* Với dữ liệu số, giá trị thực sự của nó là công cụ chẩn đoán: khi thấy nhiều yếu vị, hoặc yếu vị cách xa trung bình/trung vị, đó là dấu hiệu bạn đang nhìn vào nhiều hơn một tổng thể bị trộn lẫn với nhau (ví dụ: khách lẻ và khách sỉ). Cách khắc phục là **phân khúc (segment)** theo yếu tố thực sự gây ra khác biệt (kênh khách hàng, mùa vụ, ngày trong tuần) và phân tích hoặc dự báo riêng cho từng nhóm, rồi hợp nhất lại — đây cũng chính là nguyên lý đằng sau dự báo từ dưới lên (bottom-up forecasting).

---

## Khối 4 — Độ phân tán: Khoảng biến thiên, Phương sai, Độ lệch chuẩn

Xu hướng trung tâm một mình có thể đánh lừa bạn — hai tập dữ liệu có thể có cùng trung bình nhưng độ tin cậy khác nhau hoàn toàn. **Độ phân tán (dispersion)** đo lường mức độ bạn có thể tin tưởng vào "giá trị điển hình" như một dự đoán cho điều sắp xảy ra.

**Khoảng biến thiên (Range)** = Giá trị lớn nhất − Giá trị nhỏ nhất. Đơn giản, nhưng bỏ qua mọi thứ ở giữa và hoàn toàn nhạy cảm với giá trị ngoại lai.

**Phương sai (Variance)** = trung bình của bình phương độ lệch so với trung bình. Việc bình phương — thay vì lấy trung bình độ lệch thô (luôn cho tổng bằng 0 và vô dụng) hoặc dùng giá trị tuyệt đối — loại bỏ dấu âm trong khi vẫn giữ tính "trơn" về mặt toán học, và nó phạt nặng hơn đối với các giá trị ngoại lai lớn — một đặc tính thường hữu ích để phát hiện rủi ro thực sự.

**Độ lệch chuẩn (Standard deviation)** = căn bậc hai của phương sai, đưa về lại đơn vị gốc để có thể so sánh trực tiếp với trung bình.

**Mẫu và tổng thể — số chia khác nhau:** khi làm việc với một **mẫu (sample)** — hầu như luôn đúng trong dự báo, vì bạn không bao giờ có "toàn bộ nhu cầu tương lai" — hãy chia cho `(n − 1)`, không phải `n` (đây gọi là **hiệu chỉnh Bessel — Bessel's correction**). Việc dùng trung bình của chính mẫu đó để tính độ lệch khiến dữ liệu trông có vẻ gần trung tâm hơn thực tế so với trung bình tổng thể thật, nên việc chia cho số nhỏ hơn `(n − 1)` sẽ điều chỉnh sai lệch này. Trong Excel: `STDEV.S` / `VAR.S` cho mẫu; `STDEV.P` / `VAR.P` chỉ khi bạn thực sự có toàn bộ tổng thể.

*Ví dụ: khi gộp hai loại khách hàng khác nhau vào một tập dữ liệu, ta có trung bình ≈16.5 nhưng độ lệch chuẩn ≈20.2 — độ lệch chuẩn lớn hơn cả trung bình là một dấu hiệu cảnh báo. Khi phân tích riêng, đơn hàng bán lẻ có độ lệch chuẩn ≈0.46 và đơn hàng sỉ ≈2.65 — cả hai đều chặt chẽ và đáng tin cậy hơn nhiều.*

---

## Khối 5 — Tứ phân vị & Khoảng tứ phân vị (IQR)

**Tứ phân vị (Quartiles)** chia dữ liệu đã sắp xếp thành bốn phần bằng nhau. **Q1** = phân vị thứ 25, **Q3** = phân vị thứ 75.

**IQR = Q3 − Q1** = độ phân tán của 50% dữ liệu ở giữa. Ít bị ảnh hưởng bởi giá trị ngoại lai (khác với khoảng biến thiên/độ lệch chuẩn) vì các giá trị cực đoan nằm hoàn toàn bên ngoài nó.

**Có hai cách tính hợp lệ** — bao gồm (inclusive) hoặc không bao gồm (exclusive) trung vị khi chia dữ liệu thành hai nửa — và chúng có thể cho ra các giá trị Q1/Q3 khác nhau từ cùng một tập dữ liệu. Excel có các hàm riêng cho từng cách (`QUARTILE.INC` và `QUARTILE.EXC`); hãy chọn một cách và dùng nhất quán để số liệu của bạn không vô tình khác với đồng nghiệp.

**Cấu trúc biểu đồ hộp (Boxplot):** hộp = từ Q1 đến Q3, đường bên trong = **trung vị** (không phải trung bình), râu (whiskers) kéo dài đến điểm cực trị xa nhất vẫn còn nằm trong phạm vi **1.5×IQR** so với hộp, bất cứ điều gì vượt quá đó = được vẽ riêng như một **giá trị ngoại lai (outlier)**.

**Ứng dụng kinh doanh:** cho bạn một quy tắc khách quan, có thể tự động hóa để xác định "thế nào là bất thường lớn/nhỏ/trễ" thay vì phán đoán chủ quan. *Ví dụ: đánh dấu một thời gian giao hàng là ngoại lai thực sự thay vì chỉ nhìn bằng mắt và nói "cái này có vẻ chậm."*

---

## Khối 6 — Độ lệch (Skewness) & Độ nhọn (Kurtosis)

**Độ lệch (Skewness)** đo lường tính bất đối xứng — mức độ "lệch" của một phân phối. Về bản chất nó chỉ là một con số thể hiện khoảng cách giữa trung bình và trung vị mà bạn đã có thể nhận ra bằng mắt.
- Lệch dương/lệch phải: đuôi dài hơn về phía bên phải. *Ví dụ: phân phối thu nhập.*
- Lệch âm/lệch trái: đuôi dài hơn về phía bên trái. *Ví dụ: độ tuổi nghỉ hưu.*
- Gần như đối xứng: độ lệch trong khoảng −0.5 đến +0.5. Vượt quá ±1: lệch vừa đến lệch nghiêm trọng.

**Độ nhọn (Kurtosis)** đo lường độ dày của đuôi phân phối — khả năng xuất hiện giá trị cực đoan, so với phân phối chuẩn — và nó **hoàn toàn độc lập với độ lệch**: một tập dữ liệu đối xứng hoàn hảo vẫn có thể có đuôi dày hoặc mỏng.
- **Mesokurtic** (độ nhọn ≈ 3): hoạt động giống phân phối chuẩn.
- **Leptokurtic** (>3): đỉnh nhọn, đuôi dày — các sự kiện cực đoan xảy ra thường xuyên hơn so với dự đoán của đường cong chuẩn.
- **Platykurtic** (<3): đỉnh phẳng, đuôi mỏng — các giá trị cực đoan hiếm gặp hơn.

**Tại sao điều này quan trọng:** các công thức tồn kho an toàn (safety stock) và khoảng tin cậy (confidence interval) thường giả định phân phối chuẩn (mesokurtic). Nếu nhu cầu thực tế là leptokurtic, các công thức đó sẽ **đánh giá thấp rủi ro hết hàng thực sự** — ngay cả khi trung bình và độ lệch chuẩn của bạn trông có vẻ hoàn toàn hợp lý.

Excel: `=SKEW(vùng dữ liệu)`, `=KURT(vùng dữ liệu)`.

---

## Chỉ để tham khảo — Các thao tác kỹ thuật trong Excel (chưa học sâu)

Kỹ năng công cụ thuần túy, không phải khái niệm — học khi cần, ví dụ từ sách tham khảo *Excel Data Analysis For Dummies*:

- Data Analysis ToolPak: cài đặt qua File → Options → Add-Ins → Analysis ToolPak
- Tham chiếu tuyệt đối và tương đối trong ô (`$A$1`)
- Công thức mảng (array formulas) — kiểu cũ (Ctrl+Shift+Enter) và mảng động/spill (`SORT`, `UNIQUE`, `FILTER`)
- `COUNT` / `COUNTA` / `COUNTIF` / `COUNTIFS`, và các hàm D (`DSUM`, `DAVERAGE`, `DCOUNT`, v.v.)
- Lưu ý: "Range" trong Excel với nghĩa là một vùng ô được chọn (ví dụ: `A1:A10`) hoàn toàn khác với "range" trong thống kê (Giá trị lớn nhất − Giá trị nhỏ nhất) — đừng để hai khái niệm này lẫn lộn trong đầu.

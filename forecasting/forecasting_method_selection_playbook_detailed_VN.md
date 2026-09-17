# Forecasting Notes — Playbook Chọn Phương Pháp & Kết Hợp (Chi Tiết, kèm Ví Dụ Tính Tay)

*Đồng hành với playbook Tổng Quan. Cùng khung sườn, giờ áp dụng đầy đủ vào ba tình huống thực tế của Supagas, kèm số liệu tính tay đầy đủ — đây là bản "làm thế nào để thực sự áp dụng việc này vào sáng thứ Hai."*

---

## Nhắc lại nhanh khung sườn (chi tiết đầy đủ nằm ở notebook Tổng Quan)

1. Kiểm tra độ dài lịch sử → 2. Chạy chẩn đoán (ACF, kiểm định trend hồi quy, ANOVA) → 3. Chọn (các) phương pháp ứng viên từ bảng quyết định → 4. So sánh các ứng viên bằng sai số trên dữ liệu giữ lại → 5. Áp dụng lớp điều chỉnh phán đoán → 6. Chẩn đoán lại định kỳ.

🧠 **Mẹo cần giữ trong đầu xuyên suốt các ví dụ bên dưới:** khung sườn không bao giờ thay đổi — chỉ có *đầu vào* (bao nhiêu lịch sử, chẩn đoán nói gì, sản phẩm quan trọng tới mức nào) mới thay đổi kết quả đầu ra.

---

## Tình huống A — Dòng bình gas theo mùa, khối lượng lớn (4 năm dữ liệu tháng, tín hiệu mạnh)

**Bối cảnh:** một sản phẩm bình gas cốt lõi. Chẩn đoán đã chạy (đây là cùng bộ dữ liệu quý từ notebook Diagnostic Tools, đại diện cho "4 năm tín hiệu rõ ràng"): ACF lag-4 = 1.00 (seasonality được xác nhận, dù đã lưu ý mẫu nhỏ), ACF lag-1 = 0.17 (không có ý nghĩa), ANOVA F = 11.53 > ngưỡng 6.59 (seasonality được xác nhận theo một cách độc lập thứ hai).

**Bước 3 — chọn phương pháp:** Trend đã xác nhận + seasonality đã xác nhận + ≥2 mùa đầy đủ lịch sử → **Holt-Winters đủ điều kiện và được chỉ định.**

**Bước 4 — forecast baseline (đã xây ở notebook Exponential Smoothing):**

| Năm 3 | Forecast Holt-Winters |
|---|---|
| Q1 | 235.4 |
| Q2 | 298.0 |
| Q3 (đỉnh) | 363.3 |
| Q4 | 308.1 |

**Bước 4, tiếp theo — xây một ứng viên thứ hai để so sánh/kết hợp.** Dùng cùng L₈=282.9, T₈=6.96 nhưng **không có hệ số mùa vụ** (tức là Holt's method một mình sẽ nói gì):

| Năm 3 | Forecast chỉ dùng Holt's (L₈ + m×T₈) |
|---|---|
| Q1 | 289.9 |
| Q2 | 296.8 |
| Q3 | 303.8 |
| Q4 | 310.7 |

🧠 **Mẹo:** để ý forecast chỉ-Holt's gần như phẳng (289.9 → 310.7) — nó không biết Q3 nên tăng vọt. Đây là cái giá trực tiếp, có thể thấy được, của việc bỏ qua seasonality trên dữ liệu có seasonality thật.

**Kết hợp Ensemble — lấy trung bình hai ứng viên:**

| Năm 3 | Holt-Winters | Chỉ-Holt's | **Trung bình Ensemble** |
|---|---|---|---|
| Q1 | 235.4 | 289.9 | **262.7** |
| Q2 | 298.0 | 296.8 | **297.4** |
| Q3 | 363.3 | 303.8 | **333.6** |
| Q4 | 308.1 | 310.7 | **309.4** |

**Tại sao bạn có thể thực sự muốn kết hợp này, chứ không chỉ dùng con số Holt-Winters thuần túy:** nhớ lại correlation lag-4 của ACF là 1.00 đã được lưu ý là chỉ đến từ 4 cặp dữ liệu — một cơ sở mong manh để tin tưởng hoàn toàn vào seasonal index. Kết hợp thêm góc nhìn phẳng hơn của chỉ-Holt's **giúp phòng ngừa khả năng dao động mùa vụ bị phóng đại** do quá ít lịch sử. Khi có thêm nhiều năm dữ liệu và seasonal index được xác nhận trên nhiều dữ liệu hơn, bạn sẽ nghiêng trọng số kết hợp trở lại về phía Holt-Winters thuần túy.

**Bước 5 — lớp điều chỉnh phán đoán, cụ thể hóa:** giả sử đội Sales xác nhận một khách hàng lớn sẽ tăng khối lượng đặt hàng 15%, bắt đầu từ Q3. Áp dụng vào con số thống kê:

**Forecast đồng thuận cuối cùng cho Q3 = 363.3 × 1.15 ≈ 417.8** (dùng con số Holt-Winters ở đây, vì đây là một sự kiện tương lai *đã biết, đã xác nhận*, không phải một biện pháp phòng ngừa cho sự không chắc chắn của dữ liệu — điều chỉnh phán đoán nên đặt lên trên ước lượng thống kê tốt nhất của bạn, không phải trên một con số đã kết hợp/phòng ngừa).

🧠 **Mẹo — đừng nhầm lẫn hai loại điều chỉnh:** kết hợp ensemble phòng ngừa **sự không chắc chắn thống kê trong chính model**. Lớp điều chỉnh phán đoán đưa vào **thông tin thật về tương lai mà model không thể thấy**. Cả hai đều hợp lý, nhưng chúng trả lời các câu hỏi khác nhau — đừng áp dụng điều chỉnh mở rộng khách hàng vào một con số đã phòng ngừa/trung bình "để an toàn hơn" mà không suy nghĩ xem bạn thực sự đang giải quyết sự không chắc chắn nào.

---

## Tình huống B — Sản phẩm hoàn toàn mới, 3 tháng lịch sử, tương tự một dòng sản phẩm đã có

**Bối cảnh:** một loại bình gas kích thước mới, chỉ có 3 tháng dữ liệu thực tế — chưa đủ gần cho bất kỳ phương pháp thống kê nào (tất cả đều cần lịch sử thật; Holt-Winters cụ thể cần 2+ mùa đầy đủ). Nhưng nó tương tự về mặt hóa học/chức năng với một dòng sản phẩm đã có, ổn định, có hình dạng seasonal đã biết (dùng lại các chỉ số trước đó: S₁=0.80, S₂=1.00, S₃=1.20, S₄=1.00).

**Bước 1 nói dừng lại và dùng forecast theo phán đoán — nhưng "phán đoán" không nhất thiết phải là "đoán mò thuần túy."** Đây là nơi **analog/proxy forecasting** (forecast tương tự/đại diện) phát huy tác dụng: mượn *hình dạng* của một mẫu hình mùa vụ đã được thiết lập từ một sản phẩm tương tự, rồi tỷ lệ hóa nó bằng số liệu thực tế ban đầu của chính sản phẩm mới.

**Ví dụ tính tay:** quý đầu tiên (Q1) của sản phẩm mới bán được 50 đơn vị. Chỉ số mùa vụ Q1 của dòng sản phẩm đã có là 0.80 (Q1 luôn chạy thấp hơn 20% so với mức bình thường của chính nó). Dùng điều này để suy ra *mức bình thường ngầm định (đã khử mùa vụ)* cho sản phẩm mới:

**Mức ngầm định = 50 ÷ 0.80 = 62.5**

Giờ dự phóng phần còn lại của Năm 1 bằng *hình dạng mùa vụ của sản phẩm đã có*, áp dụng vào mức ngầm định này:

| Quý | Chỉ số mùa vụ mượn | Forecast (62.5 × chỉ số) |
|---|---|---|
| Q2 | 1.00 | 62.5 |
| Q3 (đỉnh) | 1.20 | 75.0 |
| Q4 | 1.00 | 62.5 |

🧠 **Mẹo — kỹ thuật này có tên gọi thật và giới hạn thật:** "analog forecasting" là thực hành tiêu chuẩn cho sản phẩm mới, nhưng nó chỉ hiệu quả nếu sản phẩm đối chiếu **thực sự có thể so sánh được** — cùng nhóm khách hàng, cùng mục đích sử dụng, cùng driver mùa vụ. Mượn hình dạng từ một sản phẩm chỉ *trông* tương tự bề ngoài (cùng vật liệu, khác phân khúc khách hàng) có thể tạo ra một forecast trông tự tin nhưng sai một cách tự tin. Quyết định phán đoán "đây có thực sự là một đối chiếu tốt không" quan trọng hơn phép tính ở đây.

**Bước 6, áp dụng:** khi sản phẩm mới này tích lũy đủ 2+ năm dữ liệu thực tế của chính nó, chạy lại các công cụ chẩn đoán (Công cụ 1/Công cụ 2) trên dữ liệu *của chính nó* và tiến lên model Holt-Winters riêng — đừng tiếp tục dựa vào hình dạng mượn mãi mãi.

---

## Tình huống C — SKU phụ tùng ổn định, khối lượng thấp, 6 năm lịch sử, không có gì đáng kể

**Bối cảnh:** 6 năm lịch sử (rất nhiều), nhưng ACF không cho thấy gì đáng kể ở bất kỳ lag nào, và p-value của regression-theo-thời-gian là 0.71 (không gần ý nghĩa chút nào).

**Bước 3 — chọn phương pháp:** Không có trend thật, không có seasonality thật → **SMA hoặc SES không chỉ "chấp nhận được," nó được bằng chứng chỉ định đúng đắn.** Dùng Holt-Winters ở đây nghĩa là fit ba thành phần (và ba hằng số tinh chỉnh) vào một mẫu hình thực chất chỉ là noise quanh một mức phẳng — rủi ro overfitting thuần túy mà không có lợi ích gì.

**Ví dụ tính tay — SMA 6 tháng trên actual gần đây:** 40, 38, 42, 41, 39, 40.

**SMA = (40+38+42+41+39+40)/6 = 240/6 = 40** → forecast tháng tới = **40**.

**Áp dụng "kết hợp phân đoạn" (nỗ lực tương xứng) một cách rõ ràng:** SKU này rõ ràng có giá trị thấp và khối lượng thấp. **Quyết định đúng ở đây là *không* đầu tư thêm nỗ lực** — không ensemble, không tìm driver regression, không tinh chỉnh phán đoán ngoài một bước kiểm tra hợp lý. Kỹ năng đang được thể hiện không phải là chạy toán phức tạp hơn; **mà là nhận ra đúng lúc khi nào sự phức tạp không cần thiết.**

🧠 **Mẹo — đây là tình huống người ta hay sai nhất, theo hướng ngược lại với Tình huống A:** cám dỗ là áp dụng cùng mức độ chặt chẽ cho mọi SKU "để thấu đáo." Đó không phải là thấu đáo — đó là phân bổ nỗ lực sai chỗ. Dành công sức chẩn đoán sâu + kết hợp cho những SKU mà độ chính xác forecast thực sự tác động tới kết quả kinh doanh (khối lượng cao, giá trị cao, rủi ro nguồn cung cao).

---

## Tổng hợp mẹo ghi nhớ qua cả ba tình huống

- 🧠 **Kết quả từ bảng quyết định là một ứng viên khởi đầu, không phải câu trả lời cuối cùng** — Tình huống A cho thấy ngay cả một forecast Holt-Winters "được chọn đúng" vẫn đáng để kết hợp với một ứng viên đơn giản hơn khi bằng chứng nền tảng (ACF mẫu nhỏ) có điểm yếu đã biết.
- 🧠 **Không có lịch sử ≠ không thể forecast** — Tình huống B cho thấy forecast theo phán đoán vẫn có thể dựa trên bằng chứng (mượn một hình dạng mùa vụ thật) thay vì đoán mò thuần túy.
- 🧠 **Vượt qua mọi kiểm định chẩn đoán với kết quả "không có gì đáng kể" tự nó là một kết quả hợp lệ, hữu ích** — Tình huống C cho thấy xác nhận "không có mẫu hình thật" đúng đắn hướng bạn tới phương pháp *đơn giản hơn*, tiết kiệm nỗ lực lẽ ra sẽ bị lãng phí.
- 🧠 **Kết hợp Ensemble phòng ngừa sự không chắc chắn của model/dữ liệu. Lớp điều chỉnh phán đoán đưa vào thông tin tương lai thật. Biết bạn đang làm cái nào, và đừng thay thế cái này bằng cái kia.**
- 🧠 **Nỗ lực nên tỷ lệ với mức độ quan trọng kinh doanh của sản phẩm, không nên áp dụng đồng đều trên toàn bộ danh mục** — cùng mức độ chặt chẽ forecasting đúng đắn cho Tình huống A sẽ bị lãng phí ở Tình huống C, và không đủ (nếu không có kỹ thuật analog) ở Tình huống B.

---

## Các bẫy phổ biến (mở rộng với các ví dụ này trong đầu)

1. **"Vì tôi có 6 năm dữ liệu (Tình huống C), tôi nên dùng phương pháp tinh vi nhất có sẵn."** ❌ Nhiều lịch sử hơn không tạo ra trend/seasonality nếu nó không tồn tại — chẩn đoán đã nói "không có gì đáng kể," và kết quả đó nên được tin tưởng, không bị ghi đè chỉ vì "có đủ dữ liệu để thử cái gì đó cầu kỳ hơn."
2. **"Một sản phẩm mới không có lịch sử thì không thể forecast được."** ❌ Analog forecasting (Tình huống B) cho bạn một điểm khởi đầu hợp lý, dựa trên bằng chứng — chỉ đừng coi nó đáng tin cậy ngang với một model xây trên lịch sử đã xác nhận của chính sản phẩm đó.
3. **"Nếu tôi đã kết hợp phòng ngừa (ensemble), tôi không cần thêm lớp điều chỉnh phán đoán nữa."** ❌ Chúng giải quyết các vấn đề khác nhau (sự không chắc chắn của model vs. thông tin tương lai thật) và thường cần cả hai, theo trình tự — phòng ngừa trước, rồi mới điều chỉnh cho các sự kiện tương lai đã biết lên trên.
4. **"Seasonal index từ Holt-Winters chắc chắn đúng một khi kiểm định F/ACF xác nhận seasonality tồn tại."** ❌ Xác nhận seasonality là *thật* (ANOVA/ACF ở Tình huống A) là một câu hỏi khác với xác nhận *độ lớn chính xác* của seasonal index có đáng tin không — kết hợp ensemble tồn tại chính xác để phòng ngừa sự không chắc chắn thứ hai, riêng biệt đó.

---

**Tiếp theo trong lộ trình:** Forecast Accuracy & Uncertainty (MAPE/MAD/RMSE) — công cụ biến "ứng viên nào hoạt động tốt hơn" (Bước 4 xuyên suốt playbook này) thành một phép so sánh đo lường thực sự thay vì một phán đoán chủ quan.

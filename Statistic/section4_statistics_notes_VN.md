# Section 4 — Correlation and Regression (Tương quan và Hồi quy)

*Bản tiếng Việt — giữ nguyên các thuật ngữ kỹ thuật tiếng Anh (correlation, regression, r, R², slope, intercept...) kèm giải thích tiếng Việt. Section này trả lời một câu hỏi khác với Section 1–3: thay vì "có sự khác biệt/hiệu ứng không?" (hypothesis testing), nó hỏi "hai biến số có di chuyển cùng nhau không, và mình có dùng biến này để dự đoán biến kia không?" Đây chính là cầu nối trực tiếp vào forecasting — regression thực chất là "forecasting bằng một biến driver" thay vì "forecasting chỉ dựa vào thời gian."*

---

## Block 1 — Hệ số tương quan (Correlation Coefficient — r)

**Bài toán kinh doanh:** trước khi xây bất kỳ model forecasting nào dùng thêm một biến số thứ hai (nhiệt độ, khuyến mãi, số ngày còn lại tới lễ, giá bán), bạn cần một cách để trả lời câu hỏi: **biến số này có thực sự di chuyển cùng với thứ mình đang forecast không, và mạnh tới mức nào?** Nếu không, thêm biến đó vào forecast chỉ là nhiễu (noise). Correlation là công cụ trả lời câu hỏi đó bằng một con số duy nhất.

**Định nghĩa:** Pearson correlation coefficient, **r**, đo lường độ mạnh và hướng của mối quan hệ **tuyến tính (linear)** giữa hai biến liên tục (continuous). r luôn nằm trong khoảng **−1 đến +1**.

| Giá trị r | Ý nghĩa |
|---|---|
| +1 | Quan hệ tuyến tính dương hoàn hảo |
| 0 | Không có quan hệ tuyến tính |
| −1 | Quan hệ tuyến tính âm hoàn hảo |
| Càng gần ±1 | Quan hệ càng mạnh |
| Càng gần 0 | Quan hệ càng yếu |

**Scatter plot thực tế trông như thế nào ở từng mức (đây là trực giác cần nhớ, không chỉ là con số):**
- **r = +1** → mọi điểm nằm chính xác trên một đường thẳng đi lên từ trái sang phải. Không có độ phân tán (scatter) nào — tín hiệu thuần túy.
- **r = −1** → mọi điểm nằm chính xác trên một đường thẳng đi xuống từ trái sang phải. Cùng kiểu "không phân tán," nhưng ngược hướng.
- **r = 0** → các điểm tạo thành một đám mây vô hình dạng. Không có xu hướng lên hay xuống nào cả — di chuyển trên trục x không cho biết gì về vị trí của y.
- **r khoảng ±0.7 đến ±0.9 (mức "mạnh" phổ biến trong thực tế)** → một dải chéo rõ ràng, có xu hướng lên hoặc xuống, nhưng vẫn có độ phân tán quanh đường thẳng — đây mới là hình dạng của phần lớn dữ liệu kinh doanh thực tế, không phải trường hợp hoàn hảo r = ±1.

**Ví dụ từ slide gốc — Ice Cream Sales vs. Temperature:**

| Nhiệt độ (°C) | Doanh số kem (units) |
|---|---|
| 15 | 120 |
| 18 | 135 |
| 20 | 150 |
| 22 | 165 |
| 25 | 200 |
| 28 | 240 |
| 30 | 260 |
| 32 | 280 |
| 35 | 320 |
| 37 | 400 |

**r = 0.8794** — quan hệ tuyến tính dương mạnh: nhiệt độ tăng thì doanh số kem cũng tăng, và mối quan hệ khá gần tuyến tính (dù không hoàn hảo).

**⚠️ Cái bẫy dễ mắc phải — dấu của correlation phụ thuộc hoàn toàn vào cách bạn *định nghĩa* hướng của biến số.** Điều này đáng để dừng lại suy nghĩ kỹ, vì nó sẽ lặp lại ở Block 5 khi đọc slope của regression.

Ví dụ: bạn đang kiểm tra xem nhu cầu (demand) mua cylinder có tăng khi ngày lễ đến gần không, dùng biến gọi là **"số ngày còn lại tới lễ" (days until the holiday).** Giả sử bạn tính được r = 0.85.

- "Số ngày còn lại tới lễ" **đếm ngược** khi lễ đến gần (còn 30 ngày → 29 → ... → 1 → 0).
- Nếu demand thực sự tăng *khi lễ đến gần*, thì demand đang tăng *trong khi* "số ngày còn lại" đang giảm. Đó là hai biến di chuyển **ngược chiều nhau** — nghĩa là correlation phải **âm** (r ≈ **−0.85**, không phải +0.85).
- Một r = **+0.85** dương giữa hai biến này thực ra lại kể câu chuyện ngược lại: demand tăng khi *càng xa* ngày lễ (khi "số ngày còn lại" càng lớn) — điều này mâu thuẫn với câu chuyện "demand tăng dần khi lễ đến gần."

**Bài học tổng quát:** đừng bao giờ đoán dấu của correlation chỉ dựa vào chủ đề ("lễ và demand chắc chắn phải tương quan dương"). Luôn truy ngược lại **hướng số học thực tế (literal, numeric direction)** của từng biến theo cách nó được định nghĩa. Tự hỏi: "khi con số của biến này tăng, con số của biến kia tăng (dương) hay giảm (âm)?" Nếu muốn xây trực giác bằng một biến đếm *lên* dần tới ngày lễ (ví dụ "số ngày kể từ khi bắt đầu lên kế hoạch"), hãy định nghĩa lại biến sao cho dấu khớp với câu chuyện bạn muốn kể — hoặc đơn giản là luôn kiểm tra lại hướng số học trước khi viết kết luận.

**Tại sao điều này quan trọng cho công việc forecasting ở Supagas:** bạn sẽ xây các driver kiểu "số ngày còn lại tới cuối tháng," "số tuần còn lại tới đợt bảo trì theo lịch," hoặc "khoảng cách tới ngày lễ" — tất cả đều là biến kiểu đếm ngược. Hiểu sai dấu ở đây sẽ khiến bạn hoặc thêm driver sai dấu vào regression (Block 5), hoặc đọc sai correlation matrix và rút ra kết luận kinh doanh ngược hoàn toàn so với dữ liệu thực.

---

## Block 2 — Các giả định của Pearson Correlation (Assumptions)

**Tại sao điều này quan trọng:** r là một công thức cụ thể được xây trên các giả định cụ thể. Tính r trên dữ liệu vi phạm các giả định đó sẽ cho ra một con số vẫn tính được về mặt kỹ thuật nhưng gây hiểu lầm trong thực tế — nó sẽ báo "không có quan hệ" trong khi thực ra có quan hệ *phi tuyến (non-linear)* mạnh, hoặc bị vài điểm cực trị (extreme points) làm sai lệch.

**Sáu giả định (theo slide):**

1. **Linearity (tính tuyến tính)** — quan hệ giữa hai biến phải thực sự là một đường thẳng, không phải đường cong. r chỉ đo quan hệ *tuyến tính*; một quan hệ cong mạnh (ví dụ hình chữ U) có thể cho ra r ≈ 0 dù hai biến rõ ràng có liên quan.
2. **Continuous data (dữ liệu liên tục)** — cả hai biến cần được đo ở thang interval hoặc ratio (con số có khoảng cách ý nghĩa — nhiệt độ, số lượng bán, đô la), không phải dữ liệu categorical hay ordinal.
3. **Homoscedasticity** — độ phân tán (scatter) của các điểm quanh đường thẳng phải tương đối ổn định trên toàn bộ khoảng của x. Nếu độ phân tán loe rộng ra hoặc thu hẹp lại khi x tăng, giả định này bị vi phạm.
4. **Normality (phân phối chuẩn)** — cả hai biến nên có phân phối gần chuẩn (điều này quan trọng nhất cho *kiểm định ý nghĩa (significance test)* của r, ít quan trọng hơn với bản thân hệ số thô).
5. **No outliers (không có điểm ngoại lệ)** — một điểm cực trị duy nhất có thể kéo r lệch đáng kể, vì công thức r được xây từ các độ lệch bình phương (squared deviations), rất nhạy với outlier (cùng lý do variance/std dev nhạy với outlier — xem lại Section 1).
6. **Paired observations (quan sát theo cặp)** — mỗi điểm dữ liệu phải là một cặp (x, y) thực sự từ cùng một đơn vị quan sát (ví dụ: nhiệt độ *và* doanh số của cùng một ngày) — không phải hai danh sách số không liên quan được xếp cạnh nhau ngẫu nhiên.

**Ứng dụng thực tế cho bạn:** trước khi tin vào một con số correlation trong Excel (`CORREL`), hãy nhìn scatter plot trước. Một con số mà không có biểu đồ đi kèm có thể đánh lừa bạn — đây là cùng bài học "luôn trực quan hóa trước khi tin vào con số tóm tắt" từ cảnh báo ở Section 1 về việc chỉ dựa vào mean/std dev (kiểu bẫy Anscombe's quartet).

---

## Block 3 — Correlation ≠ Causation (Tương quan không phải là nhân quả)

**Bài toán kinh doanh mà nguyên tắc này bảo vệ:** tìm được một r mạnh rất dễ gây ảo tưởng — cảm giác như bạn đã tìm ra *lý do tại sao* điều gì đó xảy ra. Nhưng r chỉ cho biết hai thứ di chuyển cùng nhau; nó không nói gì về cái gì gây ra cái gì, hay liệu có mối quan hệ nhân quả nào thực sự tồn tại hay không.

**Ba cách khác nhau mà một correlation mạnh có thể tồn tại mà không có nhân quả trực tiếp:**

1. **Nhân quả thật, nhưng bạn không thể biết chỉ từ r** — nhiệt độ *thực sự* gây ra doanh số kem tăng (nóng → người ta muốn ăn đồ lạnh). Correlation phù hợp với câu chuyện này, nhưng bản thân con số không chứng minh hướng nhân quả — nó cũng "phù hợp" như nhau với một thế giới nơi một yếu tố thứ ba gây ra cả hai.
2. **Confounding variable (biến gây nhiễu ẩn)** — một biến bạn chưa đo được đang thúc đẩy cả hai biến bạn đo, khiến chúng trông có vẻ liên quan với nhau trong khi thực ra không hề kết nối trực tiếp.
3. **Spurious correlation (tương quan giả, thuần túy trùng hợp)** — ví dụ từ slide: mức tiêu thụ chocolate bình quân đầu người theo quốc gia tương quan với số giải Nobel mà quốc gia đó giành được. Không có cơ chế thực sự nào liên kết hai thứ này — đây là trùng hợp (hoặc trong trường hợp này, cả hai đều liên quan lỏng lẻo tới mức độ giàu có chung của một quốc gia, một dạng confounder một phần), và coi đây là nhân quả sẽ là một sai lầm.

**Tại sao điều này quan trọng trực tiếp cho demand planning:** nếu bạn thấy "số lượt nhắc tới sản phẩm trên mạng xã hội trong tuần" tương quan với các đợt tăng doanh số, đừng vội kết luận là lượt nhắc *gây ra* đợt tăng đó — một chương trình khuyến mãi hay sự kiện theo mùa có thể đang thúc đẩy cả lượt nhắc lẫn doanh số một cách độc lập. Hành động dựa trên một correlation giả hoặc bị nhiễu (ví dụ tăng ngân sách quảng cáo vì một correlation trùng hợp) sẽ lãng phí ngân sách mà không tác động vào driver thực sự.

**Kỷ luật mà điều này tạo ra:** coi mỗi correlation là một *manh mối cần điều tra thêm*, không phải một *kết luận để hành động ngay*. Hãy tự hỏi "có cơ chế hợp lý nào không?" và "liệu có thứ gì khác có thể giải thích cả hai không?" trước khi xây một driver forecasting hay đưa ra khuyến nghị kinh doanh dựa trên nó.

---

## Block 4 — Hệ số xác định (Coefficient of Determination — R²)

**Bài toán kinh doanh:** r cho biết độ mạnh và hướng, nhưng các stakeholder thường muốn một con số dễ diễn giải trực tiếp hơn: **"bao nhiêu phần trăm sự biến động của thứ mình đang forecast thực sự được giải thích bởi biến số này?"** R² trả lời câu hỏi đó.

**Công thức (cho simple/one-variable regression):**

**R² = r²**

Vì là giá trị bình phương, R² luôn nằm trong khoảng **0 đến 1** (hoặc biểu diễn dạng 0%–100%), bất kể r ban đầu dương hay âm — R² bỏ qua thông tin về hướng và chỉ giữ lại "giải thích được bao nhiêu."

**Ví dụ từ slide:** r = 0.8 → R² = 0.8² = **0.64** → **"64% biến động của Y được giải thích bởi X."** 36% còn lại được giải thích bởi các yếu tố khác không nằm trong biến số này (các driver khác, nhiễu ngẫu nhiên, sai số đo lường).

**Áp dụng vào ví dụ ice cream:** r = 0.8794 → R² = 0.8794² ≈ **0.7734** → khoảng **77% biến động của doanh số kem được giải thích chỉ bởi nhiệt độ**. Khoảng 23% còn lại đến từ mọi thứ khác — ngày trong tuần, khuyến mãi, giá của đối thủ, và yếu tố ngẫu nhiên.

**Tại sao con số này quan trọng hơn r trong thực tế:** R² là con số bạn báo cáo cho stakeholder khi giải thích lý do dùng một model forecasting. "Nhiệt độ giải thích 77% biến động doanh số kem" là một câu nói cụ thể, có thể hành động, về mức độ bạn có thể tin vào một forecast dựa trên nhiệt độ, và mức độ bất định còn lại (~23%) cần được bù đắp bằng safety stock hoặc điều chỉnh theo kinh nghiệm (judgmental adjustment) — điều này kết nối trực tiếp với logic safety stock dựa trên PI ở Section 3, Block 6.

**Một sự đánh đổi cần ghi nhớ cho sau này (giới thiệu trước, sẽ khai triển đầy đủ khi học multiple regression):** thêm *nhiều* biến hơn vào một regression sẽ không bao giờ làm giảm R² (chỉ có thể giữ nguyên hoặc tăng), điều này khiến R² một mình không phải là cách tốt để quyết định "có nên thêm biến này không?" — một model có thể trông ngày càng tốt hơn theo R² trong khi thực ra lại kém hữu ích hơn khi forecast dữ liệu *mới* (overfitting). Đây là chủ đề cho khi bạn học multiple regression / adjusted R²; hiện tại chỉ cần biết R² trả lời "giải thích được bao nhiêu," không phải "có nên đưa biến này vào model không."

---

## Block 5 — Simple Linear Regression (Hồi quy tuyến tính đơn giản)

**Bài toán kinh doanh mà correlation một mình không giải quyết được:** correlation cho biết *rằng* hai biến di chuyển cùng nhau và *mạnh tới mức nào*. Nó không cho bạn một công thức dùng được để đưa vào một giá trị x mới và lấy ra một giá trị y dự đoán. Regression làm được điều đó — nó khớp (fit) một phương trình thực sự vào dữ liệu.

**Phương trình simple linear regression:**

**Y = β₀ + β₁X + ε**

| Thành phần | Ý nghĩa |
|---|---|
| Y | Biến phụ thuộc (dependent variable) — thứ bạn muốn dự đoán (ví dụ: doanh số kem, demand) |
| X | Biến độc lập (independent variable) — thứ bạn dùng để dự đoán Y (ví dụ: nhiệt độ) |
| β₀ | **Intercept (hệ số chặn)** — giá trị dự đoán của Y khi X = 0 |
| β₁ | **Slope (độ dốc)** — Y thay đổi bao nhiêu khi X tăng thêm 1 đơn vị |
| ε | **Error term (sai số)** — khoảng cách giữa giá trị đường thẳng dự đoán và giá trị thực tế xảy ra (mọi thứ đường thẳng không nắm bắt được — đây chính là phần "chưa giải thích" mà R² đo lường) |

**Đọc dấu của slope — cùng cái bẫy từ Block 1, giờ áp dụng cho regression:** β₁ dương nghĩa là Y tăng khi X tăng; β₁ âm nghĩa là Y giảm khi X tăng. Nếu X là biến kiểu đếm ngược ("số ngày còn lại tới lễ"), một β₁ âm mới là điều bạn kỳ vọng cho câu chuyện "demand tăng khi lễ đến gần" — luôn kiểm tra chéo dấu của slope với **hướng số học thực tế** của X, giống hệt cách bạn giờ làm với correlation.

**Cách chọn đường "best fit" thực tế — trực giác từ slide:** regression không chọn đường thẳng bằng cách quan sát hay thử-sai trong thực tế (Excel/phần mềm giải trực tiếp bằng công thức ở Block 6), nhưng việc xem các ứng viên tệ bị loại bỏ giúp hiểu *tại sao* một đường cụ thể là "tốt nhất." Dùng dataset Hours Studied (X) vs. Test Score % (Y) từ slide:

- **Ứng viên 1: Y = 0.00 + 1.00X** — rõ ràng quá dốc và bắt đầu quá thấp; dự đoán sai lệch nhiều với hầu hết các điểm.
- **Ứng viên 2: Y = 2.00 + 1.00X** — cùng độ dốc, dịch lên — vẫn quá dốc so với độ phân tán thực tế của dữ liệu.
- **Ứng viên 3: Y = 2.00 + 0.40X** — hình dạng tốt hơn, nhưng chưa tối thiểu hóa tổng sai số trên tất cả các điểm.
- **Đường best-fit cuối cùng: ŷ = 15.79 + 0.9760x** — đây là đường duy nhất tối thiểu hóa tổng bình phương khoảng cách theo chiều dọc giữa các điểm thực tế và đường thẳng (phương pháp này gọi là **"least squares"** — bình phương, một lần nữa, vì cùng lý do variance/std dev dùng bình phương: nó phạt nặng các sai số lớn hơn nhiều so với sai số nhỏ, và tránh việc sai số dương và âm triệt tiêu lẫn nhau).

**Diễn giải phương trình cuối, ŷ = 15.79 + 0.9760x:**
- **Intercept (15.79):** một học sinh học 0 giờ được dự đoán đạt điểm 15.79%. (Thường không có ý nghĩa thực tế nhiều — chỉ là điểm đường thẳng cắt trục — nhưng vẫn là một phần cần thiết của phương trình cho mọi dự đoán.)
- **Slope (0.9760):** mỗi giờ học thêm liên quan tới việc điểm số dự đoán tăng thêm **0.976 điểm phần trăm**.
- **Thực hiện một dự đoán:** thay bất kỳ giá trị X nào vào. Ví dụ: học 10 giờ → ŷ = 15.79 + 0.9760(10) = **25.55%**. (Chỉ mang tính minh họa — luôn kiểm tra dự đoán so với khoảng giá trị X thực tế đã dùng để xây model; dự đoán quá xa ngoài khoảng đó, gọi là *extrapolation*, sẽ không đáng tin cậy.)

---

## Block 6 — Tính đường Regression bằng tay (Calculating the Regression Line by Hand)

**Tại sao nên làm bằng tay ít nhất một lần:** các hàm `SLOPE`/`INTERCEPT`/`LINEST` của Excel (và công cụ Regression trong Data Analysis ToolPak) sẽ tính ngay lập tức, nhưng làm thủ công một lần giúp khắc sâu *cái gì* thực sự đang được tính — cùng nguyên tắc "hiểu động cơ trước khi tin vào lối tắt" xuyên suốt project này.

**Các công thức:**

**b₁ (slope) = (nΣxy − ΣxΣy) / (nΣx² − (Σx)²)**

**b₀ (intercept) = (Σy − b₁Σx) / n**

**Những gì cần chuẩn bị trước — năm cột chạy (running columns) từ dữ liệu thô:** x, y, x², y², và xy. Bạn tính tổng từng cột, sau đó thay năm tổng đó (n, Σx, Σy, Σx², Σxy) vào các công thức trên. (Σy² không cần cho hệ số, nhưng dùng sau nếu muốn tính R² bằng tay.)

**Ví dụ từ slide — cùng dataset Hours Studied / Test Score, n = 10:**

- Σx = 371 (tổng số giờ học của 10 học sinh)
- Σy = 520 (tổng điểm số của 10 học sinh)
- (Σx², Σxy được tính tương tự — tính tổng các cột x² và xy trên cả 10 dòng)

Thay vào công thức cho ra:

**b₁ = 0.9760**
**b₀ = 15.79**

→ **ŷ = 15.79 + 0.9760x** — chính xác cùng phương trình cuối cùng ở Block 5, giờ đạt được bằng tay thay vì bằng quan sát.

**Quy trình thực hiện từng bước, cho bất kỳ dataset mới nào bạn xây ở Supagas:**
1. Đặt x và y thành hai cột.
2. Thêm ba cột nữa: x², y², xy — tính theo từng dòng.
3. Tính tổng mỗi cột trong năm cột (n = số dòng).
4. Thay năm tổng vào công thức b₁ trước (b₁ chỉ phụ thuộc vào các tổng, không phụ thuộc b₀).
5. Thay b₁ vào công thức b₀.
6. Viết phương trình cuối ŷ = b₀ + b₁x.

---

## Section 4 kết nối với lộ trình forecasting như thế nào

Section này là cầu nối giữa "thống kê thuần túy" và các kỹ thuật forecasting sắp tới:

- **Regression-based forecasting** chính là phương trình ở Block 5, dùng theo chiều tiến: sau khi khớp ŷ = β₀ + β₁x trên dữ liệu lịch sử, bạn thay vào một x *tương lai* (chi tiêu khuyến mãi tháng tới, dự báo nhiệt độ tuần tới, số ngày còn lại tới một sự kiện đã biết) để có một forecast demand — đây là cách tiếp cận hoàn toàn khác với các phương pháp time-series (moving average, exponential smoothing) sắp học, vốn chỉ dùng *các giá trị demand quá khứ của chính nó* và bỏ qua hoàn toàn các driver bên ngoài.
- **Xác định các demand driver ở Supagas** bắt đầu từ correlation (Block 1): trước khi xây bất kỳ model regression nào, bạn sẽ tính correlation giữa các driver tiềm năng (thời tiết, ngày trong tuần, số ngày tới lễ, hoạt động khuyến mãi, thay đổi giá) với demand lịch sử, giữ lại những driver có quan hệ thực sự, hợp lý về mặt nhân quả (Block 3), và loại bỏ phần còn lại.
- **R² (Block 4) trở thành "bảng điểm chất lượng model"** của bạn — "model dựa trên driver này giải thích được 77% biến động demand" là một cách cụ thể để truyền đạt độ tin cậy của forecast cho stakeholder, và phần chưa giải thích được chính xác là thứ mà safety stock (thông qua logic PI ở Section 3, Block 6) cần bù đắp.
- **Phả hệ (genealogy) giới thiệu trước:** simple linear regression (một driver) là thành viên đơn giản nhất của một họ phương pháp mở rộng thành **multiple regression** (nhiều driver cùng lúc) và cuối cùng tới các model machine learning linh hoạt hơn — mỗi bước đánh đổi độ phức tạp tăng thêm để nắm bắt được nhiều hơn mẫu hình thực tế, với cái giá là cần nhiều dữ liệu hơn và cẩn thận hơn với overfitting (đã giới thiệu ở Block 4).

---

## Tham khảo — Các hàm Excel cho Section 4

| Hàm | Chức năng |
|---|---|
| `CORREL(array1, array2)` | Tính r trực tiếp |
| `RSQ(known_ys, known_xs)` | Tính R² trực tiếp |
| `SLOPE(known_ys, known_xs)` | Tính b₁ (slope) |
| `INTERCEPT(known_ys, known_xs)` | Tính b₀ (intercept) |
| `LINEST(known_ys, known_xs)` | Trả về toàn bộ mảng thống kê regression (slope, intercept, R², standard errors) trong một array formula |
| `TREND(known_ys, known_xs, new_xs)` | Trả về các giá trị y dự đoán cho các giá trị x mới, dùng đường regression đã khớp |
| `FORECAST.LINEAR(x, known_ys, known_xs)` | Dự đoán một giá trị y duy nhất cho một x mới — phiên bản "dự đoán một lần" trực tiếp của TREND |
| Data Analysis ToolPak → **Regression** | Toàn bộ kết quả regression trong một hộp thoại: coefficients, R², standard errors, p-values cho từng hệ số, bảng ANOVA — công cụ tất-cả-trong-một một khi bạn đã hiểu rõ từng phần từ notebook này |

---

**Tiếp theo (các phương pháp forecasting):** moving average → weighted moving average → exponential smoothing (simple → double/Holt → triple/Holt-Winters) → regression-based forecasting (tiếp nối trực tiếp Block 5–6 của section này). Mỗi phương pháp sẽ được dạy bằng cách chỉ ra nó khắc phục điều gì so với phương pháp trước, và nó vẫn còn sai ở đâu — cùng cách tiếp cận phả hệ (genealogy) đã dùng cho các hypothesis test ở Section 3.

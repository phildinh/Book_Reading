# Section 3 — Kiểm định giả thuyết (Hypothesis Testing): Tài liệu ghi chú

*Xây dựng từ slide Hypothesis Testing, đã sửa và bổ sung qua quá trình hỏi-đáp. Phần này chia làm hai nhánh: **Arc A (bên dưới)** bao gồm logic chung mà MỌI kiểm định (test) trong slide đều dùng lại. **Arc B** (sẽ bổ sung dần) đi qua từng kiểm định cụ thể trong 12 loại: dùng để làm gì → điều kiện áp dụng (assumptions) → công thức → ví dụ minh họa.*

---

# ARC A — NỀN TẢNG (logic mà mọi kiểm định đều dùng chung)

## Block 1 — H₀, H₁, Mức ý nghĩa (α), và P-value

**Bài toán kinh doanh:** mỗi ngày đi làm bạn sẽ gặp câu hỏi kiểu "điều này có thực sự thay đổi, hay chỉ là nhiễu ngẫu nhiên bình thường?" Hypothesis testing là quy trình chính thức, lặp lại được, để trả lời câu hỏi đó mà không tự đánh lừa bản thân.

**Vì sao H₀ phải là giả định "nhàm chán" mặc định, về mặt cơ chế — không chỉ vì lý do triết lý:** bạn chỉ có thể tính một xác suất (p-value) từ một giả thuyết nêu ra *một con số chính xác*. H₀ luôn phát biểu điều gì đó chính xác (μ = 150, p = 0,5, μ₁ = μ₂) — sự chính xác đó cho phép bạn dựng một phân phối xác suất cụ thể, đã biết, để so sánh dữ liệu của bạn với nó. H₁ thường là một khoảng ("có sự khác biệt"), không có một phân phối cụ thể nào để tính từ đó. Vậy nên đây không chỉ là sự thận trọng — bạn *không thể* tính p-value từ H₁ về mặt kỹ thuật.

**Mẹo cơ học để luôn khai báo H₀ đúng: H₀ luôn giữ dấu "="** (hoặc ≤/≥, vẫn bao gồm dấu bằng). H₁ luôn nhận phần còn lại — ≠, <, hoặc >. Hãy thử viết nghi ngờ của bạn thành một phát biểu chính thức với một con số và một dấu so sánh; bên nào tự nhiên chứa dấu "=" thì đó là H₀. Nếu nghi ngờ của bạn tự nó đã chứa ≠, <, hoặc > thì đó chính là H₁.

**Cách kiểm tra thứ hai, nói đơn giản: bất cứ điều gì bạn đang cố tìm bằng chứng ỦNG HỘ luôn luôn là H₁.** Giả định "không có gì mới xảy ra" — bất kể sự kiện nào được nhắc đến trước trong câu chuyện — luôn luôn là H₀.

**Lưu ý cho kiểm định một phía (one-tailed):** khi nghi ngờ của bạn có hướng cụ thể, H₀ nhận *toàn bộ khoảng đối lập*, không chỉ riêng giá trị "=". Kiểm định cho việc tăng → H₀: μ ≤ μ₀, H₁: μ > μ₀. Kiểm định cho việc giảm → H₀: μ ≥ μ₀, H₁: μ < μ₀. Kiểm định "khác nhau, không biết theo hướng nào" → H₀: μ = μ₀, H₁: μ ≠ μ₀ (hai phía / two-tailed).

**P-value** = P(thấy dữ liệu này, hoặc cực đoan hơn, NẾU H₀ đúng). **Quy tắc quyết định: p-value < α → reject H₀ (bác bỏ). p-value ≥ α → fail to reject H₀ (không đủ cơ sở bác bỏ).** Mẹo nhớ: **"Small p, problem for H₀"** (p nhỏ, là vấn đề cho H₀).

**Chúng ta không bao giờ "chấp nhận" H₀ — phép so sánh với tòa án:** "fail to reject" giống như một phán quyết "không có tội" — nó không chứng minh sự vô tội, nó nghĩa là bằng chứng chưa đủ mạnh để kết tội. H₀ vẫn có thể sai; mẫu của bạn có thể chỉ quá nhỏ hoặc quá nhiễu để phát hiện ra (đây chính xác là điều Type II error / power, ở Block 2–3, nói về).

**Điểm quan trọng cần chính xác — thống kê không bao giờ chứng minh điều gì, nó chỉ định lượng rủi ro:**
- Reject H₀ → *"Đủ bằng chứng để ủng hộ H1, với một rủi ro α đã biết rằng kết luận này là báo động giả."* Không phải "H1 đã được chứng minh là đúng."
- Fail to reject H₀ → *"Không đủ bằng chứng ủng hộ H1 trong dữ liệu này."* Không phải "H1 đã được chứng minh là sai" — hiệu ứng đó có thể có thật nhưng chưa bị phát hiện (rủi ro Type II error).
- Bằng chứng: nếu "reject H₀" có nghĩa là sự thật chắc chắn, thì Type I error (bác bỏ một H₀ đúng) không thể tồn tại. Nhưng nó có tồn tại — vậy nên việc bác bỏ luôn luôn là một quyết định xác suất, không bao giờ là sự đảm bảo.

---

## Block 2 — Type I Error và Type II Error

Không có kiểm định nào loại bỏ hoàn toàn rủi ro — bạn chỉ được chọn loại sai lầm nào mình sẵn sàng chấp nhận hơn.

**Mẹo nhớ: Type I = False Alarm (báo động giả). Type II = Missed Alarm (bỏ sót báo động).**

| | Thực tế: H₀ đúng | Thực tế: H₀ sai (H₁ đúng) |
|---|---|---|
| **Bạn reject H₀** | **Type I Error (α)** — báo động giả | Quyết định đúng (tỉ lệ này = Power) |
| **Bạn fail to reject H₀** | Quyết định đúng | **Type II Error (β)** — bỏ sót báo động |

**Ví dụ chuông báo cháy (từ slide):** H₀ = "không có cháy." Chuông reo, không có cháy → Type I. Chuông im lặng, có cháy thật → Type II.

**Sự đánh đổi:** giảm α (ít báo động giả hơn) sẽ làm tăng β (bỏ sót nhiều vấn đề thật hơn) — bạn không thể giảm cả hai chỉ bằng cách xoay cùng một "núm vặn." Đòn bẩy duy nhất cải thiện cả hai cùng lúc là **tăng cỡ mẫu (sample size)**.

**Sai lầm nào quan trọng hơn là một quyết định kinh doanh, không phải một sự thật thống kê** — nó phụ thuộc vào chi phí tương đối giữa một báo động giả và một lần bỏ sót phát hiện trong tình huống cụ thể đó (ví dụ: đổ lỗi sai cho một nhà cung cấp tốt sẽ làm hỏng mối quan hệ; bỏ sót một sự chậm trễ thật sẽ gây tổn thất liên tục). Đây chính là lý do α được lựa chọn có chủ đích theo từng bối cảnh, chứ không mặc định 0,05 một cách mù quáng.

---

## Block 3 — Power của kiểm định

**Power = 1 − β.** Mở rộng cùng câu chuyện chuông báo cháy: **β = tỉ lệ bỏ sót (miss rate). Power = tỉ lệ bắt được (catch rate).** Chúng luôn cộng lại thành 100% "số lần thực sự có cháy."

**Cần tưởng tượng hai thế giới riêng biệt để tính cái này** (khác với α/p-value, chỉ cần "nếu H₀ đúng"):
- **Thế giới 1 (H₀ đúng):** dữ liệu của bạn tập trung quanh μ₀. α = tần suất dữ liệu của thế giới này vẫn vượt qua ngưỡng cắt của bạn chỉ do ngẫu nhiên.
- **Thế giới 2 (H₁ đúng, tại một giá trị *cụ thể* mà bạn phải tự chọn — dữ liệu không thể tự đưa ra con số này cho bạn):** dữ liệu của bạn tập trung quanh một giá trị thay thế thật sự nào đó μ₁. Power = bao nhiêu phần của dữ liệu *thế giới này* rơi vào vùng "reject." β = bao nhiêu phần vẫn rơi vào vùng "fail to reject," chỉ vì lấy mẫu không may mắn.

**Điều gì làm tăng power:**
- **Cỡ mẫu lớn hơn** → độ trải rộng của cả hai đường cong ở hai thế giới đều hẹp lại → chồng lấn ít hơn → hiệu ứng thật ít có khả năng bị ẩn trong nhiễu.
- **Kích thước hiệu ứng (effect size) lớn hơn** (giá trị thật cách xa μ₀ hơn) → hai thế giới ít chồng lấn hơn → dễ phân biệt hơn.
- **α cao hơn** → cũng làm tăng power, nhưng phải trả giá bằng nhiều Type I error hơn — cùng một núm vặn, đánh đổi ngược chiều (không phải cải thiện miễn phí, khác với cỡ mẫu).

**Power mục tiêu (chuẩn của slide): 0,80** — 80% cơ hội phát hiện một hiệu ứng thật, nếu nó tồn tại.

**Ứng dụng thực tế — điều luôn cần nhớ:** một kết quả "fail to reject H₀" từ một kiểm định có **power thấp** là bằng chứng yếu cho "không có gì xảy ra" — nó có thể chỉ có nghĩa là kiểm định chưa đủ nhạy. Hãy tin một kết quả "không có gì" nhiều hơn khi power của kiểm định đó cao.

*(Cách tính β một cách chính thức — dời tâm phân phối lấy mẫu về một μ₁ đã chọn và tìm bao nhiêu phần rơi vào phía "sai" của ngưỡng cắt gốc — là cơ chế sâu hơn, tạm gác lại. Quay lại khi có một bài toán lập kế hoạch cỡ mẫu thực tế cần đến nó; phần khái niệm ở trên là đủ để diễn giải kết quả kiểm định trong công việc hàng ngày.)*

---

## Block 4 — Statistical Significance vs. Practical Significance

Hai câu hỏi riêng biệt, rất dễ nhầm lẫn:
- **Statistical significance** — "sự khác biệt này có thật, hay chỉ là ngẫu nhiên?" (p-value so với α)
- **Practical significance** — "sự khác biệt này có đủ lớn để đáng bỏ chi phí/công sức ra hành động không?" (kích thước hiệu ứng + đánh giá kinh doanh, không có công thức)

**Cái bẫy:** với cỡ mẫu đủ lớn, ngay cả một khác biệt nhỏ, vô nghĩa trong thực tế cũng có thể trở thành statistically significant (p-value nhỏ đi chỉ vì n lớn). *Ví dụ từ slide: cải thiện doanh số 0,1% có thể có ý nghĩa thống kê thật, nhưng không đáng để bỏ chi phí thay đổi.* Luôn hỏi cả hai câu trước khi bỏ nguồn lực ra — được chứng minh là có thật không đồng nghĩa với đáng để làm.

---

## Block 5 — Tính cỡ mẫu (Sample Size Calculation)

**Bài toán kinh doanh:** mọi CI/kiểm định đều cần dữ liệu mà bạn phải lên kế hoạch trước — quá ít thì margin of error vô dụng; quá nhiều thì lãng phí thời gian và tiền bạc.

**Công thức (cho một mean):**

**n = (Z_α/2 × σ / E)²**

- **E** = margin of error (mức độ chính xác bạn muốn)
- **Z_α/2** = giá trị Z ứng với mức độ tin cậy (confidence level) của bạn (1,96 cho 95%, 2,576 cho 99%)
- Luôn làm tròn kết quả **LÊN** — một cỡ mẫu lẻ không đạt được mục tiêu của bạn.

**Hai "núm vặn" độc lập, cả hai đều làm n tăng:**
- **E nhỏ hơn (độ chính xác chặt hơn)** → n lớn hơn. E bị *bình phương* ở mẫu số — giảm E đi một nửa sẽ khiến n tăng gần gấp **bốn** lần.
- **Mức độ tin cậy cao hơn** → Z_α/2 lớn hơn → n lớn hơn.

**Ví dụ minh họa (từ slide):** margin of error = 1 giờ, độ tin cậy 95%, σ = 4 giờ → n = (1,96×4/1)² = 61,46 → **làm tròn lên thành 62**.

**Phiên bản cho proportion:** n = (Z_α/2² × p(1−p)) / E² — cùng dạng, chỉ thay σ² bằng p(1−p). *Ví dụ: p=0,5, E=0,05, độ tin cậy 95% → n = 385.*

**Cách nghĩ cần giữ:** đây không phải là "lấy càng nhiều dữ liệu càng tốt" — mà là "tìm cỡ mẫu nhỏ nhất vẫn đạt được độ chính xác và độ tin cậy mình cần" trước khi bỏ nguồn lực ra thu thập.

---

## Block 6 — Point Estimate và Interval Estimate: Confidence Interval vs. Prediction Interval

**Vì sao điều này quan trọng trực tiếp cho forecasting:** một point estimate ("demand sẽ là 500 đơn vị") che giấu mức độ nên tin tưởng nó bao nhiêu. Forecasting về bản chất là một bài toán interval estimate, và chọn *đúng loại* interval mới là kỹ năng thực sự.

**Phép thử để phân biệt: câu hỏi đang nói về TRUNG BÌNH/hành vi chung của cả nhóm (CI), hay về MỘT trường hợp cá nhân/tương lai cụ thể (PI)?** Gợi ý ngôn ngữ: "trung bình," "chung," "tỉ lệ thật," "trên toàn bộ X" → CI. "Cái này," "cái tiếp theo," "một cái cụ thể," "sẽ là bao nhiêu" → PI.

| | Confidence Interval (CI) | Prediction Interval (PI) |
|---|---|---|
| Công thức | x̄ ± Z_α/2 × (σ/√n) | x̄ ± Z_α/2 × σ × √(1 + 1/n) |
| Trả lời | "TRUNG BÌNH THẬT nằm ở đâu?" | "MỘT giá trị MỚI sẽ rơi vào đâu?" |
| Khi n → ∞ | Thu hẹp dần về 0 (sự không chắc chắn khi ước lượng có thể giảm được) | Không bao giờ thu hẹp dưới một mức sàn do σ quyết định (sự biến động cá nhân là không thể giảm được — dù lấy trung bình bao nhiêu cũng không loại bỏ được) |
| Ví dụ sử dụng | "Số dòng sản phẩm trung bình thật trên mỗi đơn hàng, tính trên toàn bộ đơn hàng" | "Số dòng sản phẩm trên đơn hàng tiếp theo" / "demand tháng tới" |

**Ví dụ minh họa (từ slide):** mean=70, s=10, n=25, độ tin cậy 95% → CI = [66,08; 73,92]; PI = [50,01; 89,99] (rộng hơn nhiều, cùng một bộ dữ liệu).

**Sự hiểu lầm phổ biến nhất — đừng mắc phải sai lầm này:** một CI 95% **KHÔNG** có nghĩa là "95% các giá trị cá nhân nằm trong khoảng này." Nó có nghĩa là "tin tưởng 95% rằng TRUNG BÌNH THẬT nằm trong khoảng này." Với đủ dữ liệu, một CI có thể trở nên rất hẹp ngay cả khi các giá trị cá nhân trải rộng — CI chỉ phản ánh sự không chắc chắn về vị trí của trung bình, không phải độ trải rộng của các cá thể. Chỉ có PI mới mô tả nơi các giá trị cá nhân rơi vào.

**Forecasting (hầu như luôn luôn) là bài toán PI, không phải bài toán CI** — "demand tháng tới" là một giá trị tương lai cụ thể, không phải trung bình dài hạn. Dùng CI ở đây sẽ đánh giá thấp rủi ro thật, và sẽ khiến việc tính safety stock trở nên quá chặt một cách nguy hiểm.

**Cách điều này thực sự được dùng trong bối cảnh S&OP:** **point estimate** của mô hình thống kê trở thành con số forecast cơ sở (baseline); **độ rộng của PI** đưa trực tiếp vào việc tính safety stock (PI rộng → cần buffer lớn hơn); **"consensus forecast"** cuối cùng kết hợp baseline thống kê với ý kiến đánh giá từ Sales/Marketing (các deal đã biết, khuyến mãi, biến động thị trường mà dữ liệu lịch sử không thấy được) — thống kê đưa ra một điểm khởi đầu đáng tin cậy và một cảm nhận trung thực về mức độ không chắc chắn, chứ không phải lời cuối cùng.

---

## Block 7 — Quy trình Kiểm định Giả thuyết Tổng quát & Bảng phân loại các Test

**Mọi kiểm định cụ thể trong Arc B đều là cùng một khung 5 bước, chỉ khác công thức:**

1. **Nêu H₀ và H₁**
2. **Chọn α**
3. **Tính test statistic** — một giá trị chuẩn hóa: (giá trị quan sát được − giá trị H₀ dự đoán) / mức biến động ngẫu nhiên thông thường. Đây là một Z-score được tổng quát hóa; khi σ thật không biết (hầu như luôn vậy), nó trở thành **t-statistic** — cùng ý tưởng, nhưng phân phối có đuôi béo hơn để bù cho sự không chắc chắn thêm vào.
4. **Tìm critical value** — ngưỡng cắt trên phân phối đó, được xác định bởi α (cùng khái niệm với ngưỡng cắt dùng trong bức tranh β/power).
5. **Quyết định** — test statistic so với critical value, hoặc tương đương, p-value so với α.

**Bản đồ tổng quát — ba câu hỏi để chọn đúng test:**
1. Một mẫu, hai mẫu, hay từ 3 mẫu trở lên?
2. Đang kiểm định một mean, một proportion, hay một variance?
3. Variance của population đã biết, hay chỉ biết của sample?

| | Kiểm định... | Test |
|---|---|---|
| **Một mẫu** | Mean (σ đã biết, hoặc n≥30) | One-Sample Z |
| | Mean (σ chưa biết, n nhỏ) | One-Sample t |
| | Proportion | One-Proportion |
| | Variance | One-Variance (Chi-square) |
| **Hai mẫu** | Means (σ đã biết/n lớn) | Two-Sample Z |
| | Means (σ chưa biết) | Two-Sample t (pooled hoặc Welch's) |
| | Means, dữ liệu liên quan/phụ thuộc | Paired t |
| | Proportions | Two-Proportions |
| | Variances | Two-Variances (F) |
| **Từ 3 mẫu trở lên** | Means | ANOVA (F) |

*(Hai nhóm Chi-square khác xuất hiện sau trong slide: Goodness-of-Fit và Contingency Tables/Independence — dùng để kiểm định quy luật trong dữ liệu phân loại (categorical) thay vì một tham số số học cụ thể.)*

---

## Chỉ để tham khảo — Hàm Excel cho Section 3, Arc A

- Critical values: `NORM.S.INV(probability)` (Z), `T.INV(probability, df)` (t), `CHISQ.INV(probability, df)`, `F.INV(probability, df1, df2)`
- Xác suất/CDF: `NORM.S.DIST(z, TRUE)`, `T.DIST(t, df, TRUE)`, `CHISQ.DIST(x, df, TRUE)`, `F.DIST(x, df1, df2, TRUE)`
- Hàm hỗ trợ kiểm định trực tiếp: `Z.TEST`, `T.TEST`, `CHISQ.TEST`, `F.TEST` (hàm tắt trả về p-value trực tiếp)
- Confidence interval: `CONFIDENCE.NORM(alpha, std_dev, size)` (σ đã biết), `CONFIDENCE.T(alpha, std_dev, size)` (σ chưa biết)

---

# ARC B — Sắp tới

12 kiểm định cụ thể, mỗi cái sẽ được trình bày theo: dùng để làm gì → điều kiện áp dụng → công thức → ví dụ minh họa. Chưa được đề cập — sẽ bổ sung dần vào đây:

8. One-Sample Z Test
9. One-Sample t Test
10. One-Proportion Test
11. One-Variance Test (Chi-square)
12. Two-Sample Z Test
13. Two-Sample t Test (pooled vs. Welch's)
14. Paired t Test
15. Two-Proportions Test
16. Two-Variance Test (F-test)
17. ANOVA
18. Goodness-of-Fit Test (Chi-square)
19. Contingency Tables / Chi-square Test for Independence

# Section 3 — Hypothesis Testing: Arc B (12 kiểm định cụ thể)

*Tài liệu đi kèm với Section 3 — Arc A (nền tảng: H₀/H₁, α, p-value, Type I/II Error, Power, cỡ mẫu, CI/PI, và quy trình 5 bước + bảng phân loại — đọc file đó trước nếu thuật ngữ nào bên dưới còn lạ). File này đi qua từng kiểm định trong 12 loại, sắp xếp lại theo mức độ bạn thực sự sẽ dùng đến trong công việc demand planning (thay vì theo đúng thứ tự slide gốc), mỗi cái kèm một ví dụ minh họa theo phong cách Supagas.*

---

## Trước khi bắt đầu: Từ ví dụ sách vở sạch sẽ đến dữ liệu thật lộn xộn

Phần này đáng đọc trước khi vào các kiểm định — đây chính là khoảng cách giữa "hiểu kiểm định" và "áp dụng nó vào một bộ dữ liệu thật," và đó là một khoảng cách có thật, không phải dấu hiệu bạn đang thiếu sót gì.

**Bài toán sách vở luôn cho bạn biết sẵn thiết kế dữ liệu.** Chúng nói cho bạn biết "đây là paired" hay "đây là independent." Dữ liệu thật thì không — bạn phải tự tìm ra điều đó, và nếu sai thì kiểm định sẽ không còn giá trị.

**Cách phân biệt paired/dependent với independent trong dữ liệu thật — câu hỏi thực sự cần đặt ra:** *"Cùng một đối tượng thực tế được đo hai lần (paired), hay đây là hai đối tượng hoàn toàn khác nhau (independent)?"*
- Tỉ lệ lỗi của cùng 20 SKU, đo dưới phương pháp cũ rồi đo lại dưới phương pháp mới → **paired** (cùng một SKU, đo hai lần).
- Demand của kho Sydney so với demand của kho Melbourne → **independent** (hai đối tượng hoàn toàn khác nhau).
- Nếu không chắc: tự hỏi "nếu tôi sắp xếp cả hai danh sách, thì dòng thứ 3 ở danh sách A và dòng thứ 3 ở danh sách B có lý do thực sự để liên kết với nhau không (cùng SKU, cùng khách hàng, cùng kỳ)?" Nếu có → paired. Nếu việc khớp dòng 3 với dòng 3 là tùy tiện → independent.

**Vấn đề phức tạp mà chính bài học trước của bạn đã cảnh báo — autocorrelation (tự tương quan).** Bạn đã học ở Section 2 rằng các khoảng thời gian liên tiếp là *dependent* (demand tháng này liên quan đến tháng trước) — đó chính là lý do forecasting hoạt động được. Nhưng điều đó cũng là một vi phạm thực sự đối với giả định "các quan sát độc lập" mà hầu hết các kiểm định này yêu cầu. Trong thực tế: so sánh "demand tháng 1 vs. demand tháng 2" như hai sample độc lập là khá thiếu vững chắc về mặt kỹ thuật, vì chúng không phải là hai lần lấy mẫu độc lập — chúng là các điểm liên tiếp trong một chuỗi phụ thuộc. Điều này không có nghĩa là các kiểm định vô dụng trên dữ liệu time-series, nhưng nó có nghĩa là p-value từ các kiểm định áp dụng ngây thơ (naive) lên time-series thô có thể **quá tự tin** (hẹp hơn thực tế) — đây là một giới hạn đã biết, đáng nêu ra chứ không nên lờ đi, đặc biệt là trước khi đưa ra một tuyên bố chắc chắn với quản lý của bạn.

**Cái bẫy của bộ dữ liệu lớn (sắc nét hơn trong thực tế so với phiên bản ở Arc A):** với hàng ngàn dòng dữ liệu giao dịch thật, gần như *mọi thứ* đều trở thành statistically significant, kể cả những khác biệt nhỏ đến mức không đáng quan tâm. Không bao giờ chỉ báo cáo một p-value từ một bộ dữ liệu lớn — luôn đi kèm với độ lớn thực sự của sự khác biệt và tự hỏi liệu nó có đáng để hành động không.

**Danh sách kiểm tra thực tế trước khi chạy bất kỳ kiểm định nào trên dữ liệu thật:**
1. Đây là thiết kế paired hay independent? (câu hỏi "cùng một đối tượng hai lần?" ở trên)
2. n lớn cỡ nào? (Dưới 30 → dựa vào t-distribution và kiểm tra tính normal bằng histogram; từ 30 trở lên → CLT cho bạn nhiều dư địa hơn)
3. Nếu so sánh mean của hai nhóm, variance của chúng có tương đối giống nhau không? (Một F-test, ở Tier 3 bên dưới, trả lời chính thức câu này — hoặc đơn giản là nhìn sơ qua hai độ lệch chuẩn mẫu trước)
4. Dữ liệu này có thực sự là một time series mà giả định "các quan sát độc lập" đáng ngờ không? Nếu có, hãy xem bất kỳ p-value nào chỉ là một tín hiệu sơ bộ, không phải một sự đảm bảo chính xác, và nói rõ điều đó khi trình bày.

---

## TIER 1 — Nên học kỹ (bạn sẽ có khả năng dùng đến những cái này)

### 1. Paired t Test

**Dùng để làm gì:** so sánh hai phép đo liên quan/phụ thuộc trên *cùng* một đối tượng — công cụ trực tiếp cho câu hỏi "phương pháp mới của tôi có thực sự hiệu quả không?"

**Khi nào dùng:** dữ liệu ghép cặp (matched pairs) — cùng SKU, cùng ngày, cùng khách hàng, được đo hai lần (trước/sau, phương pháp cũ/phương pháp mới).

**Điều kiện áp dụng:** dữ liệu paired (phụ thuộc theo thiết kế), các hiệu số (differences) xấp xỉ phân phối chuẩn, lấy mẫu ngẫu nhiên.

**Công thức:** t = d̄ / (s_d/√n), df = n−1, trong đó d̄ = trung bình các hiệu số, s_d = độ lệch chuẩn của các hiệu số.

**Ví dụ minh họa — phương pháp forecasting mới có giảm sai số không?**

Bạn kiểm định phương pháp exponential smoothing mới so với phương pháp moving-average cũ trên cùng 10 SKU, so sánh MAPE (%) của cùng một tháng:

| SKU | MAPE cũ | MAPE mới | Hiệu số (Cũ−Mới) |
|---|---|---|---|
| 1 | 18 | 15 | 3 |
| 2 | 22 | 19 | 3 |
| 3 | 15 | 16 | −1 |
| 4 | 25 | 20 | 5 |
| 5 | 20 | 18 | 2 |
| 6 | 19 | 17 | 2 |
| 7 | 30 | 25 | 5 |
| 8 | 17 | 16 | 1 |
| 9 | 21 | 22 | −1 |
| 10 | 24 | 19 | 5 |

- H₀: μ_d ≤ 0 (phương pháp mới không giảm sai số). H₁: μ_d > 0 (phương pháp mới giảm sai số). Một phía (one-tailed), α = 0,05.
- d̄ = 2,4, s_d = 2,27
- t = 2,4 / (2,27/√10) = 2,4 / 0,718 = **3,34**
- df = 9, critical t (one-tailed, α=0,05) = 1,833
- 3,34 > 1,833 → **reject H₀**. Việc giảm MAPE của phương pháp mới có ý nghĩa thống kê thật — bằng chứng thực sự để cân nhắc chuyển đổi (nhớ kiểm tra cả practical significance: mức giảm MAPE trung bình ~2,4 điểm có đáng với chi phí chuyển đổi không?).

**Excel:** `=T.TEST(vùng_cũ, vùng_mới, 1, 1)` (tails=1 một phía, type=1 paired) trả về p-value trực tiếp.

---

### 2. Two-Sample t Test

**Dùng để làm gì:** so sánh trung bình giữa hai nhóm *độc lập* — các kho khác nhau, các nhà cung cấp khác nhau, các khu vực khác nhau.

**Khi nào dùng:** các sample độc lập, variance của population chưa biết. Có hai biến thể: **pooled** (giả định variance bằng nhau) hoặc **Welch's** (không giả định bằng nhau — lựa chọn an toàn hơn theo mặc định, nên kiểm tra bằng F-test trước, xem Tier 3).

**Công thức (pooled):** t = (x̄₁−x̄₂) / (s_p × √(1/n₁+1/n₂)), trong đó s_p² = ((n₁−1)s₁² + (n₂−1)s₂²) / (n₁+n₂−2), df = n₁+n₂−2.

**Ví dụ minh họa — hai kho có demand trung bình hàng tháng khác nhau không?**

Sydney: n=12 tháng, mean=500, variance=400. Melbourne: n=12 tháng, mean=460, variance=350.

- H₀: μ_Sydney = μ_Melbourne. H₁: μ_Sydney ≠ μ_Melbourne. Hai phía (two-tailed), α=0,05.
- s_p² = (11×400 + 11×350) / 22 = 375, s_p = 19,36
- t = (500−460) / (19,36 × √(1/12+1/12)) = 40 / 7,91 = **5,06**
- df = 22, critical t (two-tailed, α=0,05) ≈ 2,074
- 5,06 > 2,074 → **reject H₀**. Hai kho thực sự khác nhau về demand trung bình.

**Excel:** `=T.TEST(vùng1, vùng2, 2, 2)` cho pooled (type=2), `=T.TEST(vùng1, vùng2, 2, 3)` cho Welch's (type=3).

---

### 3. ANOVA (Analysis of Variance)

**Dùng để làm gì:** so sánh trung bình của **3 nhóm trở lên** cùng một lúc — ví dụ ba kho, ba nhà cung cấp, ba nhóm sản phẩm — mà không làm tăng Type I Error như khi chạy nhiều t-test riêng lẻ. *(Chạy 3 t-test riêng biệt ở α=0,05 mỗi cái thực ra mang tổng rủi ro báo động giả khoảng 14%, không phải 5% — đây chính là luận điểm của slide giải thích vì sao ANOVA tồn tại.)*

**Khi nào dùng:** từ 3 nhóm độc lập trở lên, mỗi nhóm xấp xỉ normal, variance giữa các nhóm tương đối bằng nhau.

**Công thức:** F = MSB/MSW (Mean Square Between ÷ Mean Square Within). Reject H₀ nếu F tính được vượt quá critical F.

**Ví dụ minh họa — ba kho có demand trung bình hàng ngày khác nhau không?** *(dùng lại đúng số liệu từ ví dụ có sẵn trong slide)*

| Kho A | Kho B | Kho C |
|---|---|---|
| 150 | 153 | 156 |
| 151 | 152 | 154 |
| 152 | 148 | 155 |
| 152 | 151 | 156 |
| 151 | 149 | 157 |
| 150 | 152 | 155 |
| Mean=151,00 | Mean=150,83 | Mean=155,50 |

Grand mean = 152,44.

- H₀: μ_A = μ_B = μ_C. H₁: có ít nhất một kho khác biệt. α=0,05.
- SSB = 84,12, SSW = 28,33
- MSB = 84,12/(3−1) = 42,06, MSW = 28,33/(18−3) = 1,89
- F = 42,06/1,89 = **22,25**
- df₁=2, df₂=15, critical F = 3,68
- 22,25 > 3,68 → **reject H₀**. Có ít nhất một kho thực sự khác biệt — đáng làm thêm một post-hoc test để xem cụ thể là kho nào.

**Excel:** Data Analysis ToolPak → ANOVA: Single Factor (cho ra bảng đầy đủ gồm cả F và p-value trực tiếp).

---

### 4. Goodness-of-Fit Test (Chi-square)

**Dùng để làm gì:** dữ liệu quan sát của bạn có thực sự tuân theo phân phối lý thuyết mà các công thức forecasting/safety-stock của bạn giả định không? Đây là bước kiểm tra thống kê trực tiếp đằng sau ưu tiên "distributions" của bạn — không bao giờ giả định Normal hay Poisson phù hợp mà không kiểm tra.

**Khi nào dùng:** dữ liệu phân loại/binned, expected frequency ≥5 mỗi nhóm.

**Công thức:** χ² = Σ (O_i − E_i)² / E_i, df = k−1 (k = số nhóm).

**Ví dụ minh họa — stockout hàng tháng có phân bố đều trên 5 nhóm sản phẩm, hay tập trung vào một nhóm?**

Bạn kỳ vọng (dựa trên tỉ trọng sản phẩm bằng nhau) stockout sẽ chia đều 20% mỗi nhóm trên tổng 100 lần stockout quý trước:

| Nhóm | Observed (O) | Expected (E) |
|---|---|---|
| A | 25 | 20 |
| B | 15 | 20 |
| C | 20 | 20 |
| D | 18 | 20 |
| E | 22 | 20 |

- H₀: stockout phân bố đều trên các nhóm (20% mỗi nhóm). H₁: không phân bố đều. α=0,05.
- χ² = (25−20)²/20 + (15−20)²/20 + 0 + (18−20)²/20 + (22−20)²/20 = 1,25+1,25+0+0,20+0,20 = **2,90**
- df=4, critical value = 9,49
- 2,90 < 9,49 → **fail to reject H₀**. Không có bằng chứng cho thấy pattern stockout tập trung — phù hợp với việc phân bố đều.

**Excel:** `=CHISQ.TEST(vùng_observed, vùng_expected)` trả về p-value trực tiếp.

---

## TIER 2 — Biết công thức và khi nào dùng (dùng thỉnh thoảng)

### 5. One-Sample t Test

**Dùng để làm gì:** trung bình mẫu của bạn có khác với một target/SLA đã biết không, khi bạn không biết σ thật của population?

**Ví dụ minh họa:** SLA target lead time = 5 ngày. Mẫu n=16 lô hàng gần đây: mean=5,6 ngày, s=1,2 ngày. Kiểm định xem lead time có *tăng* không (one-tailed).

- H₀: μ≤5, H₁: μ>5, α=0,05
- t = (5,6−5)/(1,2/√16) = 0,6/0,3 = **2,0**
- df=15, critical t (one-tailed)=1,753
- 2,0 > 1,753 → **reject H₀** — lead time thực sự đã tăng vượt SLA.

**Excel:** `=T.TEST` cần hai vùng dữ liệu — với kiểm định one-sample so với một target cố định, tính t thủ công bằng công thức trên rồi so với `T.INV`.

---

### 6. One-Proportion Test

**Dùng để làm gì:** tỉ lệ thực tế của bạn (on-time %, defect %) có khác với tỉ lệ mục tiêu không?

**Ví dụ minh họa:** target on-time delivery = 95%. Mẫu: n=200 đơn giao, 178 đúng hạn (89%). Kiểm định xem tỉ lệ thực tế có *thấp hơn* target không (one-tailed).

- H₀: p≥0,95, H1: p<0,95, α=0,05. Kiểm tra: np₀=190≥5, n(1−p₀)=10≥5 ✓ (đủ điều kiện xấp xỉ normal)
- z = (0,89−0,95) / √(0,95×0,05/200) = −0,06/0,0154 = **−3,89**
- Critical z (one-tailed) = −1,645
- −3,89 < −1,645 → **reject H₀** — tỉ lệ on-time thấp hơn target 95% một cách có ý nghĩa thống kê.

**Excel:** tính z thủ công; `=NORM.S.DIST(z, TRUE)` cho ra p-value.

---

### 7. Two-Proportions Test

**Dùng để làm gì:** so sánh tỉ lệ (on-time %, defect %) giữa hai nhóm độc lập — ví dụ hai nhà cung cấp.

**Ví dụ minh họa:** Nhà cung cấp A: 90/100 đúng hạn (90%). Nhà cung cấp B: 76/100 đúng hạn (76%). Kiểm định xem chúng có khác nhau không (two-tailed).

- H₀: p_A = p_B, H1: p_A ≠ p_B, α=0,05
- Pooled p̄ = (90+76)/200 = 0,83
- z = (0,90−0,76) / √(0,83×0,17×(1/100+1/100)) = 0,14/0,0531 = **2,64**
- Critical z (two-tailed) = ±1,96
- 2,64 > 1,96 → **reject H₀** — hai nhà cung cấp thực sự khác nhau về tỉ lệ on-time.

**Excel:** tính z thủ công như trên.

---

### 8. Contingency Tables / Chi-square Test for Independence

**Dùng để làm gì:** các câu hỏi tìm nguyên nhân gốc (root-cause) — hai yếu tố phân loại có liên quan với nhau, hay độc lập với nhau?

**Ví dụ minh họa:** lý do stockout (Supply Delay vs. Demand Spike) có liên quan đến nhóm sản phẩm (A vs. B) không?

|  | Supply Delay | Demand Spike | Total |
|---|---|---|---|
| Nhóm A | 30 | 10 | 40 |
| Nhóm B | 20 | 40 | 60 |
| Total | 50 | 50 | 100 |

- H₀: lý do stockout độc lập với nhóm sản phẩm. H₁: chúng có liên quan. α=0,05.
- Expected: E_A,Delay=20, E_A,Spike=20, E_B,Delay=30, E_B,Spike=30
- χ² = (30−20)²/20 + (10−20)²/20 + (20−30)²/30 + (40−30)²/30 = 5+5+3,33+3,33 = **16,67**
- df=(2−1)(2−1)=1, critical value=3,84
- 16,67 > 3,84 → **reject H₀** — lý do stockout THỰC SỰ liên quan đến nhóm sản phẩm (đáng điều tra vì sao nhóm B nghiêng nhiều về demand spike).

**Excel:** `=CHISQ.TEST(bảng_observed, bảng_expected)` trả về p-value trực tiếp.

---

## TIER 3 — Nền tảng bổ trợ (hiếm khi là công cụ chính được dùng riêng lẻ)

### 9. One-Sample Z Test

**Dùng để làm gì:** cùng chức năng với One-Sample t Test, nhưng chỉ hợp lệ khi *σ thật của population* đã biết (hiếm — cần dữ liệu lịch sử ổn định lâu dài) hoặc n≥30.

**Ví dụ minh họa:** dữ liệu lịch sử lâu năm cho thấy σ trọng lượng đóng gói=0,5kg. Mean tuyên bố=10kg. Mẫu n=50, mean=10,15kg. Kiểm định xem có khác 10kg không (two-tailed).

- Z = (10,15−10)/(0,5/√50) = 0,15/0,0707 = **2,12**
- Critical Z (two-tailed) = ±1,96 → 2,12 > 1,96 → **reject H₀**, trọng lượng đóng gói khác với spec.

---

### 10. Two-Sample Z Test

**Dùng để làm gì:** cùng chức năng với Two-Sample t, nhưng chỉ hợp lệ khi *cả hai* variance của population đều đã biết — hiếm gặp với dữ liệu kinh doanh thật.

**Ví dụ minh họa:** hai khu vực lớn, σ đã biết lâu năm σ₁=50, σ₂=45, n₁=n₂=40, mean₁=520, mean₂=500.

- Z = (520−500)/√(50²/40+45²/40) = 20/10,64 = **1,88**
- Critical Z (two-tailed)=±1,96 → 1,88 < 1,96 → **fail to reject H₀** — không phát hiện khác biệt có ý nghĩa (với cỡ mẫu này).

---

### 11. One-Variance Test (Chi-square)

**Dùng để làm gì:** độ biến động (không phải trung bình) của một quy trình có khác spec không? Phổ biến hơn trong QC sản xuất so với demand planning.

**Ví dụ minh họa:** spec variance của máy đóng gói=4 gram². Mẫu n=10, sample variance=7. Kiểm định xem variance có tăng không (one-tailed).

- χ² = (n−1)s²/σ₀² = 9×7/4 = **15,75**
- df=9, critical value (one-tailed, α=0,05) = 16,92 → 15,75 < 16,92 → **fail to reject H₀** — chưa xác nhận được variance tăng có ý nghĩa thống kê.

---

### 12. Two-Variance Test (F-test)

**Dùng để làm gì:** trong thực tế, chủ yếu là một bước kiểm tra nhanh — "nên dùng bản pooled hay Welch's của Two-Sample t Test?" — hơn là một câu hỏi kinh doanh độc lập.

**Ví dụ minh họa:** so sánh độ biến động thời gian giao hàng giữa hai đơn vị vận chuyển. Đơn vị A: n=10, s²=6. Đơn vị B: n=12, s²=2,5.

- F = 6/2,5 = **2,4**
- df₁=9, df₂=11, critical F(0,05,9,11) = 2,90 → 2,4 < 2,90 → **fail to reject H₀** — không có khác biệt có ý nghĩa về độ biến động, an toàn khi dùng bản *pooled* của Two-Sample t Test.

**Excel:** `=F.TEST(vùng1, vùng2)` trả về p-value trực tiếp; Data Analysis ToolPak → F-Test Two-Sample for Variances cho ra kết quả đầy đủ.

---

## Tổng kết Section 3 — Tóm tắt một trang

- **Mọi kiểm định = cùng 5 bước** (Arc A, Block 7), chỉ công thức ở bước 3 thay đổi.
- **Chọn kiểm định bằng 3 câu hỏi:** bao nhiêu sample? mean/proportion/variance? variance đã biết hay chưa?
- **Dữ liệu thật thêm câu hỏi thứ 4 mà bạn phải tự trả lời:** paired hay independent — và kiểm tra điều đó trước khi tin vào kết quả.
- **Bộ dữ liệu thật lớn khiến gần như mọi thứ đều "significant"** — luôn đi kèm p-value với effect size và một đánh giá kinh doanh.
- **Dữ liệu time-series (như demand hàng tháng) không hoàn toàn độc lập** — xem p-value từ các kiểm định ngây thơ trên nó là một tín hiệu, không phải một sự đảm bảo.

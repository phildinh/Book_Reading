# Section 2 — Xác suất (Probability): Tài liệu ghi chú

*Xây dựng từ slide Probability, đã sửa và bổ sung qua quá trình hỏi-đáp. Luôn bắt đầu từ gốc rễ — mọi khái niệm đều quay về câu hỏi "tại sao dữ liệu quá khứ lại cho phép tôi nói điều gì đó về tương lai?"*

---

## Block 1 — Ký hiệu Population vs. Sample, và Central Limit Theorem (Định lý Giới hạn Trung tâm)

**Ký hiệu:** slide dùng hai bộ ký hiệu riêng cho population (tổng thể) và sample (mẫu) để không bao giờ nhầm lẫn giữa "giá trị thật, thường không thể biết được" và "ước lượng của mình về nó": **μ** (mean của population) vs. **x̄** (mean của sample), **σ** vs. **s**, **N** vs. **n**. Trong thực tế, hầu hết công việc forecasting đều tính toán trên *sample* — bạn không bao giờ có "toàn bộ nhu cầu tương lai" để đo trực tiếp.

**Central Limit Theorem (CLT):** nếu bạn lấy tổng (sum) hoặc trung bình (average) của đủ nhiều giá trị ngẫu nhiên độc lập, thì tổng/trung bình đó sẽ tiến gần đến **normal distribution (phân phối chuẩn, hình chuông) — bất kể hình dạng dữ liệu gốc ban đầu là gì.**

*Minh họa bằng xúc xắc:* kết quả của một viên xúc xắc hoàn toàn phẳng (uniform) — mỗi mặt có xác suất bằng nhau. Tổng của 2 viên xúc xắc đã tạo thành hình tam giác. Tổng của 3 viên xúc xắc đã trông gần giống một đường cong hình chuông mượt. Lý do: để đạt một tổng *cực đoan* (ví dụ tất cả xúc xắc đều ra 6) thì mọi viên xúc xắc phải "xếp hàng" giống nhau, và xác suất đó giảm rất nhanh khi thêm xúc xắc `(1/6)ⁿ`. Trong khi để đạt một tổng ở *giữa* thì có rất nhiều cách kết hợp khác nhau để ra được kết quả đó, và số cách đó tăng còn nhanh hơn khi thêm xúc xắc — vậy nên các giá trị cực đoan trở nên hiếm hơn, giá trị ở giữa trở nên dày đặc hơn, và hình dạng mượt dần thành đường cong hình chuông.

**Tại sao điều này quan trọng:** dữ liệu nhu cầu (demand) hàng ngày có thể lộn xộn, lệch (skewed), hoặc có nhiều đỉnh (multimodal) như thực tế vốn vậy (nhớ lại ví dụ retail/bulk lẫn lộn ở Section 1). Nhưng *trung bình* của nhu cầu đó qua nhiều ngày — một sample mean — sẽ ngày càng gần với normal khi bạn gộp nhiều ngày hơn, dù dữ liệu thô hàng ngày chưa bao giờ là normal. Đây chính là lý do **các công cụ dựa trên normal distribution (z-score, confidence interval) vẫn dùng được trên dữ liệu kinh doanh lộn xộn ngoài đời thật.**

**Hệ quả trực tiếp:** một confidence interval xây từ trung bình 60 ngày nhu cầu đáng tin cậy hơn một cái xây từ trung bình 3 ngày — càng nhiều dữ liệu trong trung bình thì càng gần normal, càng đáng tin cậy hơn. Đây chính là trực giác (mẫu nhỏ = cần thận trọng hơn) sẽ quay lại một cách chính thức ở Block 7, là lý do vì sao t-distribution tồn tại.

---

## Block 2 — Ngôn ngữ cơ bản của xác suất

**Sample space (không gian mẫu)** = toàn bộ các kết quả có thể xảy ra. **Event (biến cố)** = một kết quả (hoặc một nhóm kết quả) mà bạn quan tâm. **Complement (biến cố bù, A′)** = mọi thứ *không* thuộc A, nên P(A) + P(A′) = 1.

**Mutually exclusive (loại trừ lẫn nhau)** = các biến cố không thể cùng xảy ra trong một lần thử (không chồng lấn nhau trên biểu đồ Venn) — ví dụ: một SKU trong ngày chỉ có thể là còn hàng (in-stock) hoặc hết hàng (stockout), không thể cả hai. **Independent (độc lập)** = biết biến cố này xảy ra không làm thay đổi xác suất của biến cố kia — ví dụ: xe tải giao hàng bị hỏng máy và việc khách hàng quyết định đặt hàng hôm đó. Đây là câu trả lời cho *hai câu hỏi khác nhau*, và hai biến cố loại trừ lẫn nhau thực sự luôn luôn **dependent (phụ thuộc)** — không bao giờ độc lập — vì biết biến cố này xảy ra đồng nghĩa biến cố kia chắc chắn không xảy ra.

Trường hợp quan trọng nhất trong demand planning là **dependent nhưng không mutually exclusive** — cả hai biến cố có thể xảy ra cùng lúc, và biết biến cố này thay đổi ước lượng của bạn về biến cố kia. *Ví dụ: "tháng trước demand cao" và "tháng này demand cao" rõ ràng có thể cùng xảy ra, và chúng dependent — tháng trước mạnh thực sự làm thay đổi ước lượng của bạn cho tháng này.*

**Đây chính là lý do toàn bộ forecasting hoạt động được:** forecasting chỉ có ý nghĩa vì các khoảng thời gian liên tiếp *dependent*, không independent. Nếu demand tháng này thực sự độc lập với mọi tháng trước — giống như những lần tung xúc xắc riêng biệt — thì sẽ chẳng có quy luật nào để học, và lịch sử sẽ vô dụng trong việc dự đoán tương lai. Mọi kỹ thuật trong lộ trình forecasting (moving average, exponential smoothing, và mọi thứ sau đó) thực chất là một phương pháp để **đo lường và khai thác sự phụ thuộc giữa các khoảng thời gian.**

**Union (A∪C, "HOẶC")** = mọi thứ thuộc A hoặc C hoặc cả hai — giống như `UNION` / full join trong SQL. **Intersection (A∩C, "VÀ")** = chỉ những gì thuộc cả hai cùng lúc, trên *một* đối tượng duy nhất — giống như `INNER JOIN` trong SQL. *Ví dụ: Đơn hàng #123 vừa là đơn của khách quen (repeat customer) vừa là đơn cuối tuần (weekend) — đó là một đơn hàng thỏa cả hai điều kiện cùng lúc, không phải so sánh giữa hai đơn hàng khác nhau.*

**Bẫy khi đếm:** nếu A và C không loại trừ lẫn nhau, bạn không thể chỉ cộng số lượng của A và C để ra union — bạn sẽ đếm trùng phần chồng lấn. *Ví dụ: 40 đơn từ khách quen, 25 đơn cuối tuần, 10 đơn thuộc cả hai → union thực sự không phải 40 + 25 = 65, mà là 40 + 25 − 10 = 55.* Phép trừ đó chính là addition rule sẽ được chính thức hóa ở Block 3.

---

## Block 3 — Addition Rule, Multiplication Rule, và Conditional Probability

**Addition rule ("HOẶC"):** P(A∪B) = P(A) + P(B) − P(A∩B). Phép trừ tồn tại chỉ để tránh đếm trùng phần chồng lấn. Nếu A và B mutually exclusive, P(A∩B) = 0, nên công thức rút gọn thành P(A) + P(B) — không cần điều chỉnh vì không có phần chồng lấn.

**Multiplication rule ("VÀ"):** luôn luôn nhân — câu hỏi duy nhất là *dùng số hạng thứ hai nào*:
- **Independent (độc lập):** P(A∩B) = P(A) × P(B). *Ví dụ: tung xúc xắc và tung đồng xu — 1/6 × 1/2 = 1/12.*
- **Dependent (phụ thuộc):** P(A∩B) = P(A) × P(B|A). *Ví dụ: rút liên tiếp 2 viên kẹo màu vàng mà không trả lại — 1/5 × 1/9 = 1/45; xác suất lần rút thứ hai thay đổi vì lần rút đầu đã lấy đi một viên kẹo khỏi tổng số.*

**Conditional probability (xác suất có điều kiện):** P(A|B) = P(A∩B) / P(B) — "cho biết B đã xảy ra, thì tỉ lệ nào trong đó cũng có A?" Đây là *cùng một mối quan hệ* với multiplication rule cho trường hợp dependent ở trên, chỉ là được sắp xếp lại về mặt đại số — không phải một ý tưởng riêng biệt cần học thuộc lần nữa.

---

## Block 4 — Permutation vs. Combination

**Vì sao cần các công thức này:** với các trường hợp nhỏ, bạn có thể liệt kê hết mọi khả năng bằng tay (như 2 viên xúc xắc). Nhưng các bài toán thực tế — chọn 3 sản phẩm trong số 50 để quảng bá, hoặc đếm số mã PIN 4 chữ số có thể có — có quá nhiều cách sắp xếp để liệt kê thủ công. Permutation và combination là các công thức đếm có hệ thống cho câu hỏi "có bao nhiêu cách để việc này xảy ra," từ đó dẫn thẳng vào xác suất (số kết quả thuận lợi ÷ tổng số kết quả).

**Chỉ một câu hỏi quyết định tất cả: thứ tự có quan trọng không?**
- **Thứ tự quan trọng → Permutation.** P(n,r) = n! / (n−r)!. *Ví dụ: chọn hạng 1, 2, 3 từ 8 người thi đấu — về hạng 1 và hạng 2 là hai kết quả hoàn toàn khác nhau.*
- **Thứ tự không quan trọng → Combination.** C(n,r) = n! / [r! × (n−r)!]. *Ví dụ: chọn 3 người vào một ủy ban (committee) từ 8 người — không có xếp hạng, chỉ là thành viên của nhóm.*

**Mối quan hệ, được cụ thể hóa:** một nhóm 3 người bất kỳ (ví dụ Alice, Bob, Carol) có thể được sắp xếp theo 3! = 6 thứ tự khác nhau (Alice-Bob-Carol, Alice-Carol-Bob, Bob-Alice-Carol, Bob-Carol-Alice, Carol-Alice-Bob, Carol-Bob-Alice). Cả 6 cách sắp xếp đó là 6 *permutation* khác nhau, nhưng chúng đều gộp lại thành đúng **1 combination**: vẫn là ba người đó, chỉ là bỏ qua thứ tự. Vì vậy: **C(n,r) = P(n,r) / r!** — mỗi combination bị "phóng đại" lên r! lần khi thứ tự bắt đầu được tính đến, và chia cho r! sẽ loại bỏ sự phóng đại đó.

*Ví dụ minh họa: C(8,4) = P(8,4) / 4! = 1.680 / 24 = 70 — mỗi trong số 70 ủy ban đó tương ứng với 4! = 24 cách sắp xếp khác nhau.*

**Phép thử cho bài toán thực tế (sẽ dùng lại ở Block 5):** hỏi rằng "việc đổi thứ tự ai đứng trước có tạo ra một kết quả thực tế khác biệt, phân biệt được không?" Nếu các đối tượng chỉ chia sẻ một trạng thái (như "bị lỗi" — một sản phẩm không có thứ hạng kiểu "lỗi thứ nhất," "lỗi thứ hai," nó chỉ đơn giản là lỗi hoặc không lỗi) → combination. Nếu chúng là một chuỗi các sự kiện thực sự phân biệt được theo thứ tự (hạng 1/2/3) → permutation.

**Có lặp lại (allowed with repetition) — ít gặp hơn, nhưng có trong slide:**
- Permutation có lặp lại: n^r. *Ví dụ: mã PIN 4 chữ số, từ 0–9, cho phép lặp → 10⁴ = 10.000 mã PIN có thể có.*
- Combination có lặp lại: C(n+r−1, r). *Ví dụ: chọn 3 vá kem (scoop) từ 5 vị, cho phép chọn trùng vị → C(7,3) = 35.*

Excel: `PERMUT(n,r)`, `PERMUTATIONA(n,r)` (có lặp lại), `COMBIN(n,r)`, `COMBINA(n,r)` (có lặp lại).

---

## Block 5 — Phân phối Binomial và Poisson

Cả hai phân phối này thực chất chỉ là Block 3 (multiplication rule) và Block 4 (combination) được gộp lại thành một công thức — không phải ý tưởng mới cần học thuộc từ đầu.

**Bài toán kinh doanh làm nền tảng cho công thức Binomial:** một nhà sản xuất có tỉ lệ lỗi (defect rate) 12%. Người mua kiểm tra ngẫu nhiên 20 sản phẩm và chỉ chấp nhận lô hàng nếu có 2 sản phẩm lỗi trở xuống. Bạn cần tính P(đúng 2 sản phẩm lỗi trong 20).

**Xây dựng công thức từ những gì đã biết:**
1. **Một cách sắp xếp cụ thể** (giả sử sản phẩm #1 và #2 bị lỗi, 18 cái còn lại tốt) chỉ đơn giản là multiplication rule: P = p² × (1−p)¹⁸ = 0,12² × 0,88¹⁸.
2. **Nhưng bạn không quan tâm *sản phẩm nào* bị lỗi** — chỉ cần đúng 2 trong 20 bị lỗi. Số cặp khác nhau có thể là "hai sản phẩm lỗi" là một câu hỏi combination: C(20,2) = 190.
3. Mỗi trong 190 cách sắp xếp đó có *cùng* xác suất, và chúng loại trừ lẫn nhau (chỉ có đúng một cặp cụ thể thực sự bị lỗi trong một lô hàng thật), nên bạn cộng tất cả lại — tương đương với việc nhân.

**Công thức Binomial: P(X=x) = C(n,x) × pˣ × (1−p)ⁿ⁻ˣ**

*Kết quả tính được:* P(X=0) ≈ 7,76%, P(X=1) ≈ 21,15%, P(X=2) ≈ 27,40% → P(chấp nhận lô hàng) = P(0)+P(1)+P(2) ≈ **56,3%**. Với tỉ lệ lỗi 12% và mức chấp nhận chỉ 2 lỗi trên 20 sản phẩm, đây thực sự là một tiêu chuẩn khá khắt khe — lô hàng bị từ chối gần một nửa số lần.

**Điều kiện để dùng Binomial:** số lần thử cố định (n), mỗi lần thử là nhị phân (thành công/thất bại), xác suất không đổi (p), các lần thử độc lập với nhau. Mean = n×p, Variance = n×p×(1−p).

**Phân phối Poisson** — dùng để đếm số sự kiện xảy ra trong một khoảng liên tục (thời gian hoặc không gian) khi bạn biết **tỉ lệ trung bình (λ)**, mà **không có "số lần thử" tự nhiên** để xác định. P(X=k) = (λᵏ × e⁻λ) / k!. Đặc biệt, mean = variance = λ.

**Cách chọn giữa hai phân phối:** có thể nêu ra một tập hợp số lần thử cố định, hữu hạn, đếm được không? → Binomial. Đây có phải là "các sự kiện xảy ra theo thời gian/không gian với một tỉ lệ trung bình," không có số lần thử tự nhiên? → Poisson. Nếu ép một bài toán dạng Poisson vào khuôn Binomial thì sẽ phải bịa ra một n tùy ý và giảm p để bù lại — trong khi Poisson bỏ qua hoàn toàn bước đó.

**Đánh giá thực tế mức độ liên quan đến công việc ở Supagas:**
- Cả hai đều không phải công cụ forecasting cốt lõi — moving average, exponential smoothing, và regression mới là các phương pháp chủ lực cho time series, và chúng không dùng trực tiếp toán Binomial/Poisson.
- **Binomial** phù hợp cho các câu hỏi có kết quả có/không, không phải mức độ demand: *"xác suất hơn 2 trong 20 chuyến giao bình gas trễ hạn trong tháng này là bao nhiêu?"* (một câu hỏi về SLA), hoặc lấy mẫu kiểm tra chất lượng (quality acceptance sampling) — chính là ví dụ đã tính ở trên.
- **Poisson** liên quan trực tiếp hơn, nhưng cho một phần việc cụ thể: mô hình hóa **số lượng đơn hàng cho các SKU nhu cầu chậm hoặc không liên tục (slow-moving/intermittent-demand)**, nơi mà một "trung bình mỗi ngày" mượt mà không thực sự áp dụng được. Điều này nối thẳng về bài toán phân khúc demand lộn xộn (lumpy demand) từ Section 1 — các phương pháp dựa trên Poisson là một phần cách dân chuyên nghiệp xử lý loại sản phẩm đặc thù đó, khác với các sản phẩm có demand liên tục, mượt mà mà moving average/exponential smoothing xử lý tốt.

Cả hai đều là phần nền tảng (mục 1 trong lộ trình) và đôi khi là công cụ đúng cho một tình huống cụ thể — không phải công cụ forecasting hàng ngày, cái đó sẽ đến sau trong lộ trình.

---

## Block 6 — Normal Distribution, Z-score, và Empirical Rule

Đây là điểm đến mà Central Limit Theorem ở Block 1 đã hướng tới — chính thức hóa hình dạng đó để bạn có thể thực sự tính xác suất từ nó.

**Tính chất của Normal distribution:**
- **Symmetric (đối xứng)** — hoàn toàn phản chiếu nhau ở hai bên tâm.
- **Mean = Median = Mode**, tất cả đều nằm đúng tại tâm của đường cong. (Đây cũng là một cách chẩn đoán: nếu mean, median, mode của một bộ dữ liệu thực tế nằm gần nhau, đó là dấu hiệu dữ liệu có thể xấp xỉ normal — cùng phép thử đã giới thiệu ở Section 1, Block 3.)
- **Tổng diện tích dưới đường cong = 1** (100%).

**Empirical rule (68-95-99,7)** — với normal distribution, tỉ lệ phần trăm dữ liệu nằm trong một số lượng độ lệch chuẩn (standard deviation) nhất định so với mean:
- **68%** nằm trong **1σ** so với mean (μ ± 1σ)
- **95%** nằm trong **2σ** (μ ± 2σ)
- **99,7%** nằm trong **3σ** (μ ± 3σ)

*Ví dụ: chiều cao nam giới trưởng thành, mean = 70 inch, std dev = 3 inch → 68% nam giới cao từ 67–73 inch, 95% cao từ 64–76 inch, 99,7% cao từ 61–79 inch. Hầu như không ai nằm ngoài ±3σ — đó là vùng "gần như không bao giờ xảy ra."*

**Khoảng trống mà empirical rule để lại:** nó chỉ cho số tròn tại đúng 1, 2, hoặc 3 độ lệch chuẩn. Với bất kỳ giá trị nào ở giữa — ví dụ "cao hơn 74 inch" khi mean = 70, std dev = 3 — bạn cần **Z-score**.

**Công thức Z-score: Z = (X − μ) / σ**

*Ví dụ: Z = (74 − 70) / 3 = 1,33 → 74 inch cao hơn mean 1,33 độ lệch chuẩn.*

**Đọc dấu của Z (suy ra thẳng từ công thức, không cần học thuộc quy tắc riêng):**
- Z âm = giá trị nằm **dưới** mean (X − μ âm).
- Z dương = giá trị nằm **trên** mean.
- Z = 0 = giá trị nằm đúng tại mean.

**Z-score thực sự dùng để làm gì:** nó chuyển đổi *bất kỳ* giá trị nào, từ *bất kỳ* normal distribution nào (bất kỳ mean, std dev nào), về một thang đo chung duy nhất là "cách mean bao nhiêu độ lệch chuẩn." Nghĩa là chỉ cần một bảng tham chiếu chung (hoặc một hàm Excel) là dùng được cho mọi normal distribution, thay vì cần một bảng riêng cho từng cặp mean/std dev. Đây chính là cơ chế thực sự đằng sau confidence interval và hypothesis testing.

*Ví dụ đầy đủ: Z = (65 − 70)/3 = −1,67. Nhờ tính đối xứng, P(Z < −1,67) = P(Z > +1,67) = 0,0475 → **4,75% nam giới thấp hơn 65 inch.***

Excel: `NORM.DIST(x, mean, std_dev, TRUE)` để tính xác suất trực tiếp từ giá trị thô; `NORM.S.DIST(z, TRUE)` khi đã chuẩn hóa về Z. Chú ý dấu ngoặc đơn — thứ tự thực hiện phép tính sẽ làm sai công thức Z một cách âm thầm nếu bỏ sót.

---

## Block 7 — t-distribution, Chi-square, F-distribution

Ba phân phối này ít khi được dùng trực tiếp một mình — công việc thực sự của chúng là "vận hành" các hypothesis test ở deck tiếp theo (t-test, chi-square test, ANOVA). Mục tiêu ở đây là hiểu *mỗi cái dùng để làm gì*, để việc chọn test sau này có lý do rõ ràng thay vì cảm giác tùy tiện.

**t-distribution — xây dựng trực tiếp trên Block 1.** Công thức Z-score giả định bạn biết độ lệch chuẩn *thật của population*, σ. Trong thực tế — ví dụ ước lượng thời gian giao hàng trung bình (delivery lead time) từ 15 đơn hàng gần đây — bạn hầu như không bao giờ biết σ thật; bạn chỉ có `s`, độ lệch chuẩn tính từ chính sample của mình. Dùng `s` của một sample nhỏ để thay cho σ thật sẽ tạo ra thêm sự không chắc chắn mà công thức Z thông thường không tính đến (cùng trực giác "sample nhỏ = kém tin cậy hơn" từ Block 1 với xúc xắc). t-distribution có đuôi (tail) béo hơn normal curve chính là để hấp thụ sự không chắc chắn thêm đó.

**Câu hỏi tổ chức không phải là "sample lớn hay nhỏ → chọn t hay F."** Mà là **"mình đang hỏi loại câu hỏi gì?"**

| Phân phối | Câu hỏi nó trả lời | Vai trò của cỡ mẫu |
|---|---|---|
| **t** vs **Z** | Ước lượng/kiểm định một **mean** | Dùng t khi σ không biết (dù sample nhỏ HAY lớn); chỉ dùng Z nếu biết σ thật, hoặc n đủ lớn để s ≈ σ |
| **Chi-square** | Dữ liệu phân loại (categorical) — kiểm định tính độc lập, goodness-of-fit — hoặc kiểm định một claim về **variance** | Không phụ thuộc cỡ mẫu — phụ thuộc *loại* câu hỏi |
| **F** | So sánh **variance hoặc mean giữa nhiều nhóm** (nền tảng của ANOVA) | Không phụ thuộc cỡ mẫu — phụ thuộc việc so sánh giữa các nhóm |

**Liên hệ với forecasting/Supagas:**
- t → confidence interval quanh một demand trung bình đã forecast.
- Chi-square → "lý do hết hàng (stockout reason) có độc lập với loại sản phẩm không?" hoặc "variance của demand có thay đổi không?"
- F → "các kho khác nhau có độ biến động demand (demand variability) khác nhau đáng kể không?" (thuộc phạm vi ANOVA)

Cơ chế đầy đủ (degrees of freedom, cách thiết lập test cụ thể) thuộc về deck Hypothesis Testing — block này chỉ cần vẽ ra bản đồ trước.

---

## Sơ đồ phả hệ Section 2 — Mỗi Block nối với Block tiếp theo như thế nào

1. **Population vs. sample + CLT (Block 1)** đặt nền móng cho toàn bộ section: bạn chỉ luôn tính toán trên sample statistics, nhưng CLT chính là lý do vì sao bạn được phép tin rằng trung bình của một sample hoạt động giống normal — kể cả khi dữ liệu thô thì không.

2. **Ngôn ngữ xác suất cơ bản (Block 2)** cung cấp từ vựng (event, mutually exclusive, independent, union, intersection) cần thiết trước khi có thể kết hợp bất cứ điều gì — và tiết lộ lý do thực sự vì sao forecasting hoạt động: các khoảng thời gian liên tiếp *dependent*, không independent.

3. **Addition/multiplication rule + conditional probability (Block 3)** là các công cụ chính thức để *kết hợp* các biến cố đã nêu ở Block 2 — thực chất chỉ là chính thức hóa cách sửa lỗi đếm trùng mà bạn sẽ tự nhiên nghĩ ra.

4. **Permutation vs. combination (Block 4)** là bộ công cụ đếm trả lời câu hỏi "có bao nhiêu cách để việc này xảy ra" cho các trường hợp quá lớn để liệt kê tay — chính là nguyên liệu mà Block 5 cần.

5. **Binomial và Poisson (Block 5)** là multiplication rule của Block 3 và combination của Block 4 *được gộp thành một công thức*, áp dụng cho hai dạng bài toán thực tế khác nhau: số lần thử cố định (Binomial) so với tỉ lệ theo thời gian/không gian (Poisson).

6. **Normal distribution, Z-score, empirical rule (Block 6)** là điểm đến chính thức mà CLT ở Block 1 luôn hướng tới — biến "tổng/trung bình tiến gần normal" thành một công cụ xác suất thực sự dùng được, thông qua một thang Z chung duy nhất.

7. **t, Chi-square, F (Block 7)** là cơ chế lấy logic normal distribution của Block 6 và điều chỉnh cho trường hợp thực tế lộn xộn hơn, khi độ lệch chuẩn thật của population không biết (t), câu hỏi là về phân loại hoặc variance (Chi-square), hoặc bạn đang so sánh nhiều nhóm cùng lúc (F) — chuẩn bị trực tiếp cho deck Hypothesis Testing.

**Tóm gọn trong một câu:** Section 1 là "làm sao mô tả trung thực những gì mình đã quan sát." Section 2 là "làm sao lý luận về sự không chắc chắn và kết hợp các xác suất" — và điểm đến của nó (normal distribution, đạt được thông qua Central Limit Theorem) chính là căn cứ toán học lý giải vì sao confidence interval và hypothesis test được phép tồn tại — đây là nơi deck tiếp theo sẽ tiếp tục từ đó.

---

## Chỉ để tham khảo — Hàm Excel cho Section 2 (chưa đi sâu)

- Đếm: `PERMUT(n,r)`, `PERMUTATIONA(n,r)`, `COMBIN(n,r)`, `COMBINA(n,r)`
- Binomial: `BINOM.DIST(x, n, p, cumulative)`
- Poisson: `POISSON.DIST(x, mean, cumulative)`
- Normal: `NORM.DIST(x, mean, std_dev, TRUE)`, `NORM.S.DIST(z, TRUE)`, `NORM.S.INV(probability)` (tra ngược: xác suất → Z)
- t / Chi-square / F (mới giới thiệu ở đây, dùng chính thức ở Hypothesis Testing): `T.DIST`, `CHISQ.DIST`, `F.DIST` và các hàm nghịch đảo (`.INV`) tương ứng

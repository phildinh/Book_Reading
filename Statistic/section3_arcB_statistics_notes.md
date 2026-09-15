# Section 3 — Hypothesis Testing: Arc B (The 12 Specific Tests)

*Companion file to Section 3 — Arc A (foundations: H₀/H₁, α, p-value, Type I/II errors, power, sample size, CI/PI, and the 5-step process + taxonomy — read that one first if any term below feels unfamiliar). This file walks through each of the 12 specific tests, reordered by how often you'll actually reach for them in demand planning rather than the deck's original order, each with a Supagas-flavored worked example.*

---

## Before You Start: From Clean Textbook Examples to Messy Real Data

This is worth reading before the tests themselves — it's the exact gap between "understand the test" and "apply it to a real dataset," and it's a real gap, not a sign you're missing something.

**Textbook problems hand you the design for free.** They tell you "these are paired" or "these are independent." Real datasets don't — you have to figure that out yourself, and getting it wrong invalidates the test.

**How to tell paired/dependent vs. independent in real data — the actual question to ask:** *"Is the same real-world entity being measured twice (paired), or are these two genuinely different entities (independent)?"*
- Same 20 SKUs' error rate, measured under the old method and then the new method → **paired** (same SKU, twice).
- Sydney warehouse's demand vs. Melbourne warehouse's demand → **independent** (different entities entirely).
- If you're not sure: ask "if I sorted both lists, would row 3 in list A and row 3 in list B have a real reason to be connected (same SKU, same customer, same period)?" If yes → paired. If matching row 3 to row 3 is arbitrary → independent.

**The complication your own earlier lesson already warned you about — autocorrelation.** You learned back in Section 2 that consecutive time periods are *dependent* (this month's demand relates to last month's) — that's the entire reason forecasting works. But it's also a real violation of the "independent observations" assumption most of these tests require. In practice: comparing "January demand vs. February demand" as two independent samples is technically shaky, because they're not independent draws — they're consecutive points in one dependent series. This doesn't mean the tests are useless on time-series data, but it means p-values from naive tests on raw time series can be **overconfident** (falsely narrow), and it's a known limitation worth flagging rather than ignoring, especially before making a confident claim to your manager.

**The large-dataset trap (sharper in practice than in Arc A's version):** with thousands of rows of real transactional data, almost *everything* becomes statistically significant, including differences too small to matter. Never report a p-value alone from a large dataset — always pair it with the actual size of the difference and ask if it's worth acting on.

**A practical checklist before running any test on real data:**
1. Is this a paired or independent design? (the "same entity twice?" question above)
2. Roughly how large is n? (Under 30 → lean on t-distribution and check normality with a histogram; 30+ → CLT gives more breathing room)
3. If comparing two groups' means, are their variances roughly similar? (An F-test, Tier 3 below, formally answers this — or just eyeball the two sample standard deviations first)
4. Is the data actually a time series where "independent observations" is questionable? If yes, treat any p-value as a rough signal, not a precise guarantee, and say so when you present it.

---

## TIER 1 — Learn These Well (you'll likely use these)

### 1. Paired t Test

**What it's for:** comparing two related/dependent measurements on the *same* subjects — the direct tool for "did my new method actually work?"

**When to use:** matched pairs — same SKUs, same days, same customers, measured twice (before/after, old method/new method).

**Assumptions:** paired data (dependent by design), differences roughly normally distributed, random sampling.

**Formula:** t = d̄ / (s_d/√n), df = n−1, where d̄ = mean of the differences, s_d = std dev of the differences.

**Worked example — did the new forecasting method reduce error?**

You test the new exponential smoothing method against the old moving-average method on the same 10 SKUs, comparing MAPE (%) for the same month:

| SKU | Old MAPE | New MAPE | Difference (Old−New) |
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

- H₀: μ_d ≤ 0 (new method doesn't reduce error). H₁: μ_d > 0 (new method reduces error). One-tailed, α = 0.05.
- d̄ = 2.4, s_d = 2.27
- t = 2.4 / (2.27/√10) = 2.4 / 0.718 = **3.34**
- df = 9, critical t (one-tailed, α=0.05) = 1.833
- 3.34 > 1.833 → **reject H₀**. The new method's MAPE reduction is statistically significant — real evidence to justify the switch (check practical significance too: is a ~2.4-point average MAPE drop worth the switching cost?).

**Excel:** `=T.TEST(old_range, new_range, 1, 1)` (tails=1 one-tailed, type=1 paired) returns the p-value directly.

---

### 2. Two-Sample t Test

**What it's for:** comparing averages between two *independent* groups — different warehouses, different suppliers, different regions.

**When to use:** independent samples, population variances unknown. Two variants: **pooled** (assume equal variances) or **Welch's** (don't assume equal — safer default, check with an F-test first, Tier 3).

**Formula (pooled):** t = (x̄₁−x̄₂) / (s_p × √(1/n₁+1/n₂)), where s_p² = ((n₁−1)s₁² + (n₂−1)s₂²) / (n₁+n₂−2), df = n₁+n₂−2.

**Worked example — do two warehouses have different average monthly demand?**

Sydney: n=12 months, mean=500, variance=400. Melbourne: n=12 months, mean=460, variance=350.

- H₀: μ_Sydney = μ_Melbourne. H₁: μ_Sydney ≠ μ_Melbourne. Two-tailed, α=0.05.
- s_p² = (11×400 + 11×350) / 22 = 375, s_p = 19.36
- t = (500−460) / (19.36 × √(1/12+1/12)) = 40 / 7.91 = **5.06**
- df = 22, critical t (two-tailed, α=0.05) ≈ 2.074
- 5.06 > 2.074 → **reject H₀**. The two warehouses genuinely differ in average demand.

**Excel:** `=T.TEST(range1, range2, 2, 2)` for pooled (type=2), `=T.TEST(range1, range2, 2, 3)` for Welch's (type=3).

---

### 3. ANOVA (Analysis of Variance)

**What it's for:** comparing means across **3 or more** groups at once — e.g., three warehouses, three suppliers, three product categories — without inflating Type I error by running many separate t-tests. *(Running 3 separate t-tests at α=0.05 each actually carries a combined false-alarm risk of about 14%, not 5% — the deck's own point on why ANOVA exists.)*

**When to use:** 3+ independent groups, roughly normal within each group, roughly equal variances across groups.

**Formula:** F = MSB/MSW (Mean Square Between ÷ Mean Square Within). Reject H₀ if the calculated F exceeds the critical F.

**Worked example — do three warehouses have different average daily demand?** *(numbers reused directly from the deck's own worked example)*

| Warehouse A | Warehouse B | Warehouse C |
|---|---|---|
| 150 | 153 | 156 |
| 151 | 152 | 154 |
| 152 | 148 | 155 |
| 152 | 151 | 156 |
| 151 | 149 | 157 |
| 150 | 152 | 155 |
| Mean=151.00 | Mean=150.83 | Mean=155.50 |

Grand mean = 152.44.

- H₀: μ_A = μ_B = μ_C. H₁: at least one warehouse differs. α=0.05.
- SSB = 84.12, SSW = 28.33
- MSB = 84.12/(3−1) = 42.06, MSW = 28.33/(18−3) = 1.89
- F = 42.06/1.89 = **22.25**
- df₁=2, df₂=15, critical F = 3.68
- 22.25 > 3.68 → **reject H₀**. At least one warehouse genuinely differs — worth a follow-up (post-hoc) test to see which one.

**Excel:** Data Analysis ToolPak → ANOVA: Single Factor (gives the full table including F and p-value directly).

---

### 4. Goodness-of-Fit Test (Chi-square)

**What it's for:** does your observed data actually follow the theoretical distribution your forecasting/safety-stock formulas assume? This is the direct statistical check behind your "distributions" priority — never assume Normal or Poisson fits without checking.

**When to use:** categorical/binned data, expected frequencies ≥5 per category.

**Formula:** χ² = Σ (O_i − E_i)² / E_i, df = k−1 (k = number of categories).

**Worked example — are monthly stockouts evenly spread across 5 product categories, or concentrated in one?**

You expect (based on equal product mix) stockouts to split 20% per category across 100 total stockout events last quarter:

| Category | Observed (O) | Expected (E) |
|---|---|---|
| A | 25 | 20 |
| B | 15 | 20 |
| C | 20 | 20 |
| D | 18 | 20 |
| E | 22 | 20 |

- H₀: stockouts are evenly distributed across categories (20% each). H₁: not evenly distributed. α=0.05.
- χ² = (25−20)²/20 + (15−20)²/20 + 0 + (18−20)²/20 + (22−20)²/20 = 1.25+1.25+0+0.20+0.20 = **2.90**
- df=4, critical value = 9.49
- 2.90 < 9.49 → **fail to reject H₀**. No evidence the stockout pattern is concentrated — it's consistent with an even spread.

**Excel:** `=CHISQ.TEST(observed_range, expected_range)` returns the p-value directly.

---

## TIER 2 — Know the Formula and When to Use It (occasional use)

### 5. One-Sample t Test

**What it's for:** is your sample's average different from a known target/SLA, when you don't know the true population σ?

**Worked example:** SLA target lead time = 5 days. Sample of n=16 recent shipments: mean=5.6 days, s=1.2 days. Testing if lead time has *increased* (one-tailed).

- H₀: μ≤5, H₁: μ>5, α=0.05
- t = (5.6−5)/(1.2/√16) = 0.6/0.3 = **2.0**
- df=15, critical t (one-tailed)=1.753
- 2.0 > 1.753 → **reject H₀** — lead time has genuinely increased beyond the SLA.

**Excel:** `=T.TEST` needs two ranges — for a one-sample test against a fixed target, compute t manually with the formula above and check against `T.INV`.

---

### 6. One-Proportion Test

**What it's for:** is your actual rate (on-time %, defect %) different from a target rate?

**Worked example:** target on-time delivery = 95%. Sample: n=200 deliveries, 178 on-time (89%). Testing if the actual rate is *below* target (one-tailed).

- H₀: p≥0.95, H1: p<0.95, α=0.05. Check: np₀=190≥5, n(1−p₀)=10≥5 ✓ (normal approximation valid)
- z = (0.89−0.95) / √(0.95×0.05/200) = −0.06/0.0154 = **−3.89**
- Critical z (one-tailed) = −1.645
- −3.89 < −1.645 → **reject H₀** — on-time rate is significantly below the 95% target.

**Excel:** compute z manually; `=NORM.S.DIST(z, TRUE)` gives the p-value.

---

### 7. Two-Proportions Test

**What it's for:** comparing rates (on-time %, defect %) between two independent groups — e.g., two suppliers.

**Worked example:** Supplier A: 90/100 on-time (90%). Supplier B: 76/100 on-time (76%). Testing if they differ (two-tailed).

- H₀: p_A = p_B, H1: p_A ≠ p_B, α=0.05
- Pooled p̄ = (90+76)/200 = 0.83
- z = (0.90−0.76) / √(0.83×0.17×(1/100+1/100)) = 0.14/0.0531 = **2.64**
- Critical z (two-tailed) = ±1.96
- 2.64 > 1.96 → **reject H₀** — the suppliers genuinely differ in on-time rate.

**Excel:** compute z manually as above.

---

### 8. Contingency Tables / Chi-square Test for Independence

**What it's for:** root-cause style questions — are two categorical factors related, or independent of each other?

**Worked example:** is stockout reason (Supply Delay vs. Demand Spike) related to product category (A vs. B)?

|  | Supply Delay | Demand Spike | Total |
|---|---|---|---|
| Category A | 30 | 10 | 40 |
| Category B | 20 | 40 | 60 |
| Total | 50 | 50 | 100 |

- H₀: stockout reason is independent of product category. H₁: they're related. α=0.05.
- Expected: E_A,Delay=20, E_A,Spike=20, E_B,Delay=30, E_B,Spike=30
- χ² = (30−20)²/20 + (10−20)²/20 + (20−30)²/30 + (40−30)²/30 = 5+5+3.33+3.33 = **16.67**
- df=(2−1)(2−1)=1, critical value=3.84
- 16.67 > 3.84 → **reject H₀** — stockout reason IS related to product category (worth investigating why Category B skews toward demand spikes).

**Excel:** `=CHISQ.TEST(observed_table, expected_table)` returns the p-value directly.

---

## TIER 3 — Foundational Scaffolding (rarely the main event on their own)

### 9. One-Sample Z Test

**What it's for:** same job as the One-Sample t Test, but only valid when the *true population σ* is actually known (rare — needs long, stable historical records) or n≥30.

**Worked example:** long-standing records show fill-weight σ=0.5kg. Claimed mean=10kg. Sample n=50, mean=10.15kg. Testing if different from 10kg (two-tailed).

- Z = (10.15−10)/(0.5/√50) = 0.15/0.0707 = **2.12**
- Critical Z (two-tailed) = ±1.96 → 2.12 > 1.96 → **reject H₀**, fill weight differs from spec.

---

### 10. Two-Sample Z Test

**What it's for:** same job as Two-Sample t, but only valid when *both* population variances are known — uncommon with real business data.

**Worked example:** two large regions, long-known σ₁=50, σ₂=45, n₁=n₂=40, mean₁=520, mean₂=500.

- Z = (520−500)/√(50²/40+45²/40) = 20/10.64 = **1.88**
- Critical Z (two-tailed)=±1.96 → 1.88 < 1.96 → **fail to reject H₀** — no significant difference detected (with these sample sizes).

---

### 11. One-Variance Test (Chi-square)

**What it's for:** is a process's variability (not its average) different from spec? More common in manufacturing QC than demand planning.

**Worked example:** filling machine spec variance=4 grams². Sample n=10, sample variance=7. Testing if variance has increased (one-tailed).

- χ² = (n−1)s²/σ₀² = 9×7/4 = **15.75**
- df=9, critical value (one-tailed, α=0.05) = 16.92 → 15.75 < 16.92 → **fail to reject H₀** — variance increase not statistically confirmed.

---

### 12. Two-Variance Test (F-test)

**What it's for:** in practice, mostly a quick pre-check — "should I use pooled or Welch's version of the Two-Sample t Test?" — rather than a standalone business question.

**Worked example:** comparing delivery-time variability between two couriers. Courier A: n=10, s²=6. Courier B: n=12, s²=2.5.

- F = 6/2.5 = **2.4**
- df₁=9, df₂=11, critical F(0.05,9,11) = 2.90 → 2.4 < 2.90 → **fail to reject H₀** — no significant difference in variability, safe to use the *pooled* Two-Sample t Test.

**Excel:** `=F.TEST(range1, range2)` returns the p-value directly; Data Analysis ToolPak → F-Test Two-Sample for Variances gives the full output.

---

## Section 3 Wrap — The One Page Summary

- **Every test = the same 5 steps** (Arc A, Block 7), only the formula in step 3 changes.
- **Pick the test with 3 questions:** how many samples? mean/proportion/variance? variance known or not?
- **Real data adds a 4th question you must answer yourself:** paired or independent — and check it before trusting any result.
- **Large real datasets make almost everything "significant"** — always pair a p-value with effect size and a business judgment call.
- **Time-series data (like monthly demand) isn't fully independent** — treat p-values from naive tests on it as a signal, not a guarantee.

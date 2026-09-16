# Forecasting Notes — Diagnostic Tools (Before You Pick a Method)

*Companion notebook to the Moving Average and Exponential Smoothing notebooks. Those cover the forecasting methods themselves. This one covers the step that should happen **before** choosing among them: how do you actually know if your data has a real trend or real seasonality, instead of guessing from a chart? Two tools here, both built entirely from things you already know (Section 3 hypothesis testing, Section 4 correlation/regression) — no new statistics, just a new application of it.*

---

## Block 0 — Why this step exists

Every forecasting method so far (SMA through Holt-Winters) requires you to **decide in advance** whether trend and/or seasonality are present, because that decision determines which method is even appropriate:

- No trend, no seasonality → SMA/WMA/SES are fine.
- Real trend, no seasonality → need Holt's method.
- Real trend + real seasonality → need Holt-Winters.

**The trap:** eyeballing a chart is unreliable — noise can hide a real pattern, and your eye can also "see" a pattern in pure randomness that isn't really there. **These two tools turn "does this look seasonal to me?" into an actual number with a decision rule**, exactly the same way every hypothesis test in Section 3 replaced "does this feel significant?" with a p-value.

---

## Block 1 — Tool 1: Autocorrelation (ACF)

**What it is:** correlation — the exact `CORREL()` from Section 4 — applied to a series **against a lagged copy of itself**, instead of two different variables.

### The core idea: building a "lag"

Take your data, make a second copy, and slide it over by k periods. Pair each value with the value from k periods earlier. Run `CORREL()` on the two resulting columns.

**Lag-1** = pair each period with the one right before it → tests short-term momentum/trend-like behavior.
**Lag-p** (p = suspected season length, e.g. 12 monthly / 4 quarterly) = pair each period with the same point in the previous cycle → tests seasonality.

### Tips & tricks to remember ACF

- 🧠 **"Lag = slide the strip over."** If a concept ever feels abstract, go back to this: write the series twice, slide one copy over by k boxes, correlate the two columns you now have.
- 🧠 **A perfect echo = perfect correlation, and you can often see it before calculating anything.** If two lagged columns move in obvious lockstep (each pair differs by roughly the same amount), you already know the correlation will be high — the arithmetic just confirms what your eyes caught.
- 🧠 **Check lag-1 AND lag-p — don't stop at lag-1.** A series can have weak lag-1 correlation (no simple day-to-day momentum) while still having a strong lag-p spike (real seasonality) — these are two separate questions, and seasonal data often looks "messy" at lag-1 while being very clean at lag-p.
- 🧠 **Significance band: ±1.96/√N** (N = total observations in the series). Any ACF value **inside** this band is not statistically distinguishable from zero — treat it as noise, not signal.
- 🧠 **⚠️ Small samples can produce dramatic-looking but unreliable spikes.** A "perfect" lag-p correlation calculated from only 2–3 pairs (i.e., only 2–3 years of history) should make you *more* cautious, not more confident — this is the same small-sample-power issue from Section 3. Prefer 3+ full cycles of history before trusting a seasonal ACF result.

### Worked example — small, clean, arithmetic-free intuition builder

Steadily rising data: 100, 105, 110, 115, 120.

| "Today" | "Yesterday" (lag-1) |
|---|---|
| 105 | 100 |
| 110 | 105 |
| 115 | 110 |
| 120 | 115 |

Every pair moves by the same +5 — an obvious perfect echo. **Correlation = 1.0**, no calculation needed to see it coming.

Repeating pattern (season length 3): 10, 20, 30, 10, 20, 30.

| "This period" | "3 periods ago" (lag-3) |
|---|---|
| 10 | 10 |
| 20 | 20 |
| 30 | 30 |

Identical pairs. **Correlation = 1.0** — the seasonality signature.

### Worked example — real Supagas quarterly data (200, 250, 300, 250, 220, 275, 330, 275)

**Lag-1 columns:**

| This quarter | Previous quarter |
|---|---|
| 250 | 200 |
| 300 | 250 |
| 250 | 300 |
| 220 | 250 |
| 275 | 220 |
| 330 | 275 |
| 275 | 330 |

Messy — sometimes up follows up, sometimes a big value is followed by a drop. **CORREL ≈ 0.17.**

**Lag-4 columns (same quarter, one year ago):**

| This quarter | Same quarter last year |
|---|---|
| 220 | 200 |
| 275 | 250 |
| 330 | 300 |
| 275 | 250 |

Every row moves the same consistent way. **CORREL = 1.00.**

**Significance check:** band = ±1.96/√8 ≈ ±0.69. Lag-1's 0.17 sits **inside** the band (not significant — no reliable short-term momentum signal). Lag-4's 1.00 sits **far outside** it (clearly significant — real seasonality), though flagged above as based on very few pairs, so treat it as a strong lead, not final proof, until more years of data confirm it.

---

## Block 2 — Tool 2a: Testing for Trend (Regression Slope + Hypothesis Test)

**Setup:** X = time index (1, 2, 3, ...), Y = demand. Fit simple linear regression (Section 4, Block 5–6). The slope b₁ is your trend estimate.

**The hypothesis test (Section 3's 5-step process, applied to a regression coefficient):**
- H₀: β₁ = 0 (the slope could just be noise)
- H₁: β₁ ≠ 0 (there's a real trend)
- **Excel's Regression tool / `LINEST` gives you the p-value for the slope directly** — no manual test-statistic derivation needed.
- Decision rule: same as always. p-value < 0.05 → reject H₀ → trend is real. p-value ≥ 0.05 → treat the slope as noise.

### Tips & tricks

- 🧠 **This is not a new test — it's the same 5-step hypothesis-testing logic from Section 3, just aimed at a regression coefficient instead of a mean or proportion.**
- 🧠 **A steep slope isn't automatically "real."** With very little data, even a genuinely steep-looking slope can fail to reach significance — small samples have low power (Section 3, Block 3), and trend detection is no exception.

### Worked example — the contrast that makes the decision rule concrete

| Flat / noisy data (100, 98, 102, 99, 101) | Clearly rising data (100, 110, 120, 130, 140) |
|---|---|
| Slope ≈ 0.3 | Slope = 10 |
| p-value ≈ 0.85 → fail to reject H₀ | p-value ≈ 0.0001 → reject H₀ |
| Treat as no real trend | Real, statistically confirmed trend |

---

## Block 3 — Tool 2b: Testing for Seasonality (ANOVA Across Groups)

**Setup:** group your data by "position in the cycle" (e.g., by quarter, or by month) and ask the exact question ANOVA already answers from Section 3b: **do the group means differ by more than random chance would explain?**

- H₀: all groups (e.g., all 4 quarters) share the same true average demand — no seasonal effect.
- H₁: at least one group's average genuinely differs.

### Tips & tricks

- 🧠 **The intuition, before the formula:** compare the spread **between** group averages to the spread **within** each group (across different years, same quarter). If quarters were not really different, those two kinds of spread should look similar in size. A much bigger between-group spread than within-group spread is the seasonality signal.
- 🧠 **This reuses Section 3b's ANOVA formula exactly** — F = MS_between / MS_within, compared to F-critical for your degrees of freedom.

### Worked example — real Supagas data, grouped by quarter

| Quarter | Year 1 | Year 2 | Quarter average |
|---|---|---|---|
| Q1 | 200 | 220 | 210.0 |
| Q2 | 250 | 275 | 262.5 |
| Q3 | 300 | 330 | 315.0 |
| Q4 | 250 | 275 | 262.5 |

Grand average = 262.5. Quarter averages swing as far as ±52.5 from the grand average; within the same quarter, year-to-year gaps are only 20–30. That imbalance shows up as:

| Source | SS | df | MS | F |
|---|---|---|---|---|
| Between quarters | 11,025 | 3 | 3,675 | **11.53** |
| Within quarters | 1,275 | 4 | 318.75 | |

F-critical (α=0.05, df=3,4) ≈ 6.59. Since 11.53 > 6.59: **reject H₀ — seasonality is statistically confirmed.**

---

## Block 4 — Quick-recall cheat sheet

| Tool | Reuses | Tests for | Result on Supagas example |
|---|---|---|---|
| ACF lag-1 | `CORREL()` (Section 4) | Short-term momentum/trend hint | 0.17 — not significant |
| ACF lag-p | `CORREL()` (Section 4) | Repeating seasonal cycle | 1.00 — significant (but small-sample caveat) |
| Regression slope test | Simple linear regression + hypothesis test (Sections 3+4) | Whether a real trend exists | Needs more periods to test reliably |
| ANOVA on groups | ANOVA (Section 3b) | Whether seasonal groups genuinely differ | F=11.53 > 6.59 — significant |

**When two independent tools (e.g., ACF and ANOVA) agree on the same conclusion, that agreement is what makes the finding trustworthy — not any single test alone.**

---

## Block 5 — Excel reference

| Task | How |
|---|---|
| Build a lag column | Simply offset a column reference by k rows (e.g., `=A2` copied starting one row down for lag-1) |
| Autocorrelation | `=CORREL(original_range, lagged_range)` |
| Significance band for ACF | `=1.96/SQRT(N)` — compare your ACF value against ± this |
| Trend test | Data Analysis add-in → **Regression**, with X = a time index column; read the p-value next to the X coefficient |
| Seasonality test | Data Analysis add-in → **ANOVA: Single Factor**, with each column being one "position in the cycle" (e.g., a column per quarter) |

---

## Block 6 — Common pitfalls

1. **"A high lag-1 correlation always means seasonality."** ❌ Lag-1 tests short-term momentum/trend-like behavior. Seasonality is specifically tested at **lag = season length**, not lag-1.
2. **"A perfect ACF spike from 2 years of data is solid proof."** ❌ Very few pairs can produce a dramatic-looking but statistically fragile result — treat it as a lead, confirm with more history.
3. **"If the chart looks flat, there's definitely no trend."** ❌ And the reverse — a chart that looks like it's trending can still fail a formal significance test with too little data. Let the p-value decide, not the eye.
4. **"ANOVA and regression trend tests are brand-new statistics."** ❌ They're not — both are direct reapplications of Section 3 (hypothesis testing, ANOVA) and Section 4 (regression), just aimed at a forecasting diagnosis question instead of a general business question.

---

**Next up:** the method-selection & blending playbook — bringing together every forecasting method and every diagnostic tool covered so far into one practical decision guide.

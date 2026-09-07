# Section 1 — Introduction to Statistics: Notebook Reference

*Built from the course slides, corrected and expanded through Q&A. Foundation-first — every concept ties back to "can I trust this number, or do I need to look closer?"*

---

## Block 1 — What Is Statistics? Descriptive vs. Inferential

Statistics is the discipline of reasoning from data under uncertainty. Unlike mathematics — which deals with exact, guaranteed relationships (2 + 2 = 4, always) — statistics works with real-world data that naturally varies, and its job is to draw reliable conclusions despite that variation.

**Descriptive statistics** summarizes data you already have: mean, median, mode, spread, and shape of a dataset. *Example: "Average daily sales last year was $500."*

**Inferential statistics** uses a sample to make a claim about a larger population or an unknown future, and honestly states how uncertain that claim is. *Example: "Next month's demand is likely 480–520 units, with 95% confidence."*

**Why it matters:** nobody has perfect knowledge of the future. Forecasting exists precisely because of the gap between what you know (a sample of history) and what you need to decide (the future) — this gap is the whole reason a demand planning role exists.

---

## Block 2 — Data Types & Measurement Scales

Data is either **qualitative** (categories, not numbers — e.g., product category) or **quantitative** (numeric, measurable — e.g., units sold).

Numeric data further splits into four measurement scales:

- **Nominal** — labels/identity with no order. *Example: a SKU code.* Even if the label is a number, it's not a quantity.
- **Ordinal** — ordered, but the gaps between ranks aren't guaranteed equal. *Example: delivery priority (Low/Medium/High).*
- **Interval** — numeric with equal gaps, but **no true zero** — zero is just a reference point, not "none of it." *Example: temperature in Celsius; you can't say 20°C is "twice as hot" as 10°C.*
- **Ratio** — numeric with equal gaps **and** a true zero that means "none." *Example: months of cover, sales revenue, weight.* "Twice as much" is a meaningful statement here.

**Why it matters:** the scale decides which statistics are even mathematically valid on that column. You can meaningfully average ratio-scale sales data, but averaging a 1–5 satisfaction rating (ordinal) technically assumes equal gaps between ranks that aren't guaranteed to exist.

---

## Block 3 — Central Tendency: Mean, Median, Mode

Central tendency answers "what's typical?" — and it's literally the seed of forecasting: a moving-average forecast is just "use the central tendency of recent history as your next guess."

**Mean** = sum ÷ count. Simple, but sensitive to outliers and to mixed populations — it can land on a value that describes nothing real. *Example: mixing 8 small retail orders (1–2 cylinders) with 5 bulk orders (38–45 cylinders) gives a mean of ≈16.5 — a size no actual order that day was.*

**Median** = the middle value when sorted. Resistant to outliers. When mean ≈ median, that's a good sign the data is roughly symmetric and one number describes it well; when they diverge, that's the numeric signature of skew (see Block 6).

**Mode** = the most frequent value. It's the only central-tendency measure valid for categorical data. *Example: the most common order-cancellation reason.* For numeric data, its real value is diagnostic: seeing multiple modes, or a mode far from the mean/median, signals you're looking at more than one population mixed together (e.g., retail vs. bulk customers). The fix is to **segment by the real driver** (customer channel, season, day-of-week) and analyze or forecast each group separately, then consolidate — the same principle behind bottom-up forecasting.

---

## Block 4 — Dispersion: Range, Variance, Standard Deviation

Central tendency alone can mislead you — two datasets can share the same mean but have wildly different reliability. Dispersion measures how much to trust the "typical value" as a predictor of what happens next.

**Range** = Max − Min. Simple, but ignores everything in between and is fully outlier-sensitive.

**Variance** = the average of the *squared* deviations from the mean. Squaring — rather than averaging raw deviations (which always sums to exactly zero and is useless) or using absolute value — removes negative signs while staying mathematically smooth, and it punishes large outliers harder than small ones, which is often a desirable property for spotting real risk.

**Standard deviation** = √variance, brought back into the original units so it's directly comparable to the mean.

**Sample vs. population divisor:** when working from a sample — almost always true in forecasting, since you never have "all future demand" — divide by `n − 1`, not `n` (this is Bessel's correction). Using the sample's own mean to calculate deviations makes the data look artificially closer to center than it really is relative to the true population mean, so dividing by the smaller number `n − 1` corrects that bias. Excel: `STDEV.S` / `VAR.S` for a sample; `STDEV.P` / `VAR.P` only if you truly have the entire population.

*Example: pooling two different customer types into one dataset gave mean ≈16.5 but std dev ≈20.2 — a std dev bigger than the mean itself is a red flag. Analyzed separately, retail orders had std dev ≈0.46 and bulk orders ≈2.65 — both far tighter and far more trustworthy.*

---

## Block 5 — Quartiles & IQR

Quartiles split sorted data into four equal parts. **Q1** = 25th percentile, **Q3** = 75th percentile.

**IQR = Q3 − Q1** = spread of the middle 50% of the data. Robust to outliers (unlike range or std dev) because extreme values sit outside it entirely.

**Two valid calculation conventions exist** — inclusive vs. exclusive of the median when splitting the data into halves — and they can give different Q1/Q3 numbers from the *same* dataset. Excel has separate functions for each (`QUARTILE.INC` vs. `QUARTILE.EXC`); pick one and stay consistent so your numbers don't silently disagree with a colleague's.

**Boxplot anatomy:** box = Q1 to Q3, line inside = **median** (not mean), whiskers extend to the most extreme point still within **1.5×IQR** of the box, anything beyond that = plotted separately as an outlier.

**Business use:** gives you an objective, automatable rule for "what counts as unusually big/small/late" instead of a subjective judgment call. *Example: flagging a delivery lead time as a genuine outlier rather than eyeballing "that looks slow."*

---

## Block 6 — Skewness & Kurtosis

**Skewness** measures asymmetry — how lopsided a distribution is. It's really just a number for the mean-vs-median gap you can already spot by eye.
- Positive/right skew: longer tail to the right. *Example: income distributions.*
- Negative/left skew: longer tail to the left. *Example: age at retirement.*
- Roughly symmetric: skewness between about −0.5 and +0.5. Beyond ±1: moderately to severely skewed.

**Kurtosis** measures tail thickness — how likely extreme values are, compared to a normal distribution — and it is **completely independent of skewness**: a perfectly symmetric dataset can still have fat or thin tails.
- **Mesokurtic** (kurtosis ≈ 3): behaves like a normal distribution.
- **Leptokurtic** (>3): sharp peak, fat tails — extreme events happen *more often* than a normal curve predicts.
- **Platykurtic** (<3): flat top, thin tails — extremes are rarer.

**Why it matters:** safety stock and confidence interval formulas typically assume a normal (mesokurtic) distribution. If real demand is leptokurtic, those formulas will **understate your real stockout risk** — even when your average and standard deviation both look perfectly reasonable.

Excel: `=SKEW(range)`, `=KURT(range)`.

---

## Reference only — Excel mechanics (not yet covered in depth)

Pure tool skills, not concepts — pick these up as needed, e.g. from the *Excel Data Analysis For Dummies* reference:

- Data Analysis ToolPak: install via File → Options → Add-Ins → Analysis ToolPak
- Absolute vs. relative cell references (`$A$1` syntax)
- Array formulas — old-style (Ctrl+Shift+Enter) vs. dynamic arrays/spill (`SORT`, `UNIQUE`, `FILTER`)
- `COUNT` / `COUNTA` / `COUNTIF` / `COUNTIFS`, and the D-functions (`DSUM`, `DAVERAGE`, `DCOUNT`, etc.)
- Note: Excel's "Range" as a *cell selection* (e.g. `A1:A10`) is a completely different meaning from the statistical "range" (Max − Min) — don't let the two collide in your head.

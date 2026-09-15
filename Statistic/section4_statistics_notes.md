# Section 4 — Correlation and Regression

*Built from the Correlation and Regression slides, corrected and expanded through Q&A. This section answers a different question from Sections 1–3: instead of "is there a difference/effect?" (hypothesis testing), it asks "do two variables move together, and can I use one to predict the other?" This is the direct bridge into forecasting — regression is literally "forecasting using a driver variable" instead of "forecasting using time alone."*

---

## Block 1 — The Correlation Coefficient (r)

**The business problem:** before you build any forecasting model that uses a second variable (temperature, promotions, days-to-holiday, price), you need a way to answer one question first: **does this variable actually move together with what I'm trying to forecast, and how strongly?** If it doesn't, adding it to a forecast is just noise. Correlation is the tool that answers that, as a single number.

**Definition:** the Pearson correlation coefficient, **r**, measures the strength and direction of the *linear* relationship between two continuous variables. It always falls between **−1 and +1**.

| r value | Meaning |
|---|---|
| +1 | Perfect positive linear relationship |
| 0 | No linear relationship |
| −1 | Perfect negative linear relationship |
| Closer to ±1 | Stronger relationship |
| Closer to 0 | Weaker relationship |

**What the scatter plot actually looks like at each extreme (this is the intuition to hold onto, not just the number):**
- **r = +1** → every point sits exactly on a straight line that goes up left-to-right. No scatter at all — pure signal.
- **r = −1** → every point sits exactly on a straight line that goes down left-to-right. Same "no scatter," opposite direction.
- **r = 0** → the points form a shapeless cloud. No visible upward or downward trend at all — moving left to right on the x-axis tells you nothing about where y will be.
- **r around ±0.7 to ±0.9 (the common "strong" real-world range)** → a clear diagonal band of points, trending up or down, but with visible scatter around the line — this is what most real business data looks like, not the perfect r = ±1 case.

**Deck's worked example — Ice Cream Sales vs. Temperature:**

| Temp (°C) | Ice Cream Sales (units) |
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

**r = 0.8794** — a strong positive linear relationship: as temperature rises, ice cream sales rise with it, and the relationship is close to (but not perfectly) linear.

**⚠️ The trap that's easy to fall into — correlation sign depends entirely on how you've *defined* your variable's direction.** This is worth sitting with, because it will trip you up again in Block 5 when you read a regression slope.

Take this example: you're testing whether cylinder demand rises as a public holiday approaches, using a variable called **"days until the holiday."** Suppose you calculate r = 0.85.

- "Days until the holiday" **counts down** as the holiday approaches (30 days out → 29 → ... → 1 → 0).
- If demand is genuinely rising *as the holiday approaches*, then demand is rising *while "days until holiday" is falling*. That's two variables moving in **opposite** directions — which is a **negative** correlation (r ≈ **−0.85**, not +0.85).
- A **positive** r = +0.85 between these two specific variables would actually mean the opposite story: demand rises *the further away* you are from the holiday (as "days until" gets bigger) — which contradicts the "demand builds toward the holiday" narrative.

**The general lesson:** never interpret a correlation sign from the topic alone ("holidays and demand should obviously correlate positively"). Always trace it through the *literal, numeric direction* of each variable as it's defined. Ask yourself: "as this specific variable's *number* goes up, does the other variable's *number* go up (positive) or down (negative)?" If you'd rather build intuition with a variable that counts *up* toward the holiday (e.g., "days since planning started" or simply flipping to "days remaining is small" as a proximity measure), reframe it so the sign matches the story you're telling — or just always double check the numeric direction before writing the sentence.

**Why this matters for your Supagas forecasting work:** you'll be building drivers like "days to end of month," "weeks until scheduled maintenance shutdown," or "distance from a public holiday" — all of these are countdown-style variables. Get the sign wrong here and you'll either add a driver with the wrong sign into a regression (Block 5) or misread a correlation matrix and draw the exact opposite business conclusion from what the data says.

---

## Block 2 — Assumptions of Pearson Correlation

**Why this matters:** r is a specific formula built on specific assumptions. Calculate it on data that violates those assumptions and you get a number that is technically computable but practically misleading — it will report "no relationship" when a strong *non-linear* one exists, or get distorted by a handful of extreme points.

**The six assumptions (deck):**

1. **Linearity** — the relationship between the two variables must actually be a straight line, not a curve. r specifically measures *linear* association; a strong curved (e.g., U-shaped) relationship can produce r ≈ 0 even though the variables are clearly related.
2. **Continuous data** — both variables need to be measured on an interval or ratio scale (numbers with meaningful distance between them — temperature, sales units, dollars), not categorical or ordinal data.
3. **Homoscedasticity** — the spread (scatter) of points around the line should stay roughly constant across the whole range of x. If the scatter fans out or narrows as x increases, this is violated.
4. **Normality** — both variables should be roughly normally distributed (this matters most for the *significance test* on r, less for the raw coefficient itself).
5. **No outliers** — a single extreme point can drag r substantially toward or away from a true relationship, because the formula for r is built from squared deviations, which are highly sensitive to outliers (same reason variance/std dev are outlier-sensitive — see Section 1).
6. **Paired observations** — every data point must be a genuine (x, y) pair from the same observational unit (e.g., the same day's temperature *and* that same day's sales) — not two unrelated lists of numbers lined up by coincidence.

**Practical read for you:** before trusting a correlation number in Excel (`CORREL`), glance at the scatter plot first. A number without the plot can lie to you — this is the same "always visualize before you trust the summary statistic" lesson as Section 1's warning about relying on mean/std dev alone (Anscombe's-quartet-style trap).

---

## Block 3 — Correlation ≠ Causation

**The business problem this guards against:** finding a strong r is seductive — it feels like you've found *why* something happens. But r only ever tells you two things moved together; it says nothing about what's causing what, or whether anything is causing anything at all.

**Three distinct ways a strong correlation can exist without direct causation:**

1. **Genuine causation, but you can't tell from r alone** — temperature *does* cause ice cream sales to rise (heat → people want cold food). Correlation is consistent with this story, but the number itself doesn't prove the causal direction — it's equally "consistent" with a world where a third factor drives both.
2. **Confounding variable (a hidden third cause)** — a variable you haven't measured is driving both of the variables you did measure, making them look related to each other when they're not directly connected at all.
3. **Spurious correlation (pure coincidence)** — deck's example: chocolate consumption per capita, by country, correlates with number of Nobel Prizes won. There is no real mechanism linking the two — it's a coincidence (or in this case, both are loosely tied to a country's general wealth level, which is itself a partial confounder), and treating it as causal would be a mistake.

**Why this matters directly for demand planning:** if you find that "weeks with high social media mentions of your product" correlates with sales spikes, don't automatically conclude the mentions *caused* the spike — a promotion or seasonal event could be driving both the mentions and the sales independently. Acting on a spurious or confounded correlation (e.g., increasing ad spend because of a coincidental correlation) wastes budget without moving the real driver.

**The discipline this creates:** treat every correlation as a *lead to investigate*, not a *conclusion to act on*. Ask "is there a plausible mechanism?" and "could something else explain both?" before building a forecast driver or a business recommendation on top of it.

---

## Block 4 — Coefficient of Determination (R²)

**The business problem:** r tells you strength and direction, but stakeholders usually want a more directly interpretable number: **"how much of the variation in what I'm forecasting is actually explained by this variable?"** R² answers that.

**Formula (for simple/one-variable regression):** 

**R² = r²**

Because it's a squared value, R² is always between **0 and 1** (or expressed as 0%–100%), regardless of whether r was positive or negative — R² throws away the direction information and keeps only "how much is explained."

**Deck's worked example:** r = 0.8 → R² = 0.8² = **0.64** → **"64% of the variance in Y is explained by X."** The remaining 36% is explained by other factors not captured by this one variable (other drivers, randomness, measurement noise).

**Applying it to the ice cream example:** r = 0.8794 → R² = 0.8794² ≈ **0.7734** → about **77% of the variation in ice cream sales is explained by temperature alone**. The other ~23% comes from everything else — day of week, promotions, competitor pricing, and randomness.

**Why this number matters more in practice than r itself:** R² is what you report to a stakeholder when justifying a forecasting model. "Temperature explains 77% of our ice cream sales variation" is a concrete, actionable statement about how much you can trust a temperature-based forecast, and how much uncertainty (~23%) still needs to be covered by safety stock or judgmental adjustment — this connects directly to the PI-based safety stock logic from Section 3, Block 6.

**A trade-off to hold onto for later (previewed here, developed fully once you reach multiple regression):** adding *more* variables into a regression will never decrease R² (it can only stay the same or go up), which makes R² alone a poor way to decide "should I add this variable?" — a model can look better and better on R² while actually becoming less useful for forecasting *new* data (overfitting). That's a topic for when you get into multiple regression / adjusted R²; for now, just know R² answers "how much is explained," not "should this variable be in my model."

---

## Block 5 — Simple Linear Regression

**The business problem correlation can't solve on its own:** correlation tells you *that* two variables move together and *how strongly*. It doesn't give you a usable formula to plug a new x-value into and get a predicted y-value out. Regression does — it fits an actual equation to the data.

**The simple linear regression equation:**

**Y = β₀ + β₁X + ε**

| Term | Meaning |
|---|---|
| Y | The dependent variable — what you're trying to predict (e.g., ice cream sales, demand) |
| X | The independent variable — what you're using to predict it (e.g., temperature) |
| β₀ | The **intercept** — the predicted value of Y when X = 0 |
| β₁ | The **slope** — how much Y changes for every 1-unit increase in X |
| ε | The **error term** — the gap between what the line predicts and what actually happened (everything the line doesn't capture — this is the same "unexplained" chunk that R² quantifies) |

**Reading the slope's sign — the same trap from Block 1, now applied to regression:** a positive β₁ means Y increases as X increases; a negative β₁ means Y decreases as X increases. If X is a countdown-style variable ("days until holiday"), a negative β₁ is what you'd expect for "demand rises as the holiday approaches" — always sanity-check the slope sign against the *literal* direction of X, exactly like you now do for correlation.

**How the "best fit" line is actually chosen — the deck's iterative intuition:** regression doesn't fit a line by inspection or trial and error in practice (Excel/software solves it directly with the formula in Block 6), but it's useful to see *why* a specific line is "best" by watching bad candidates get ruled out. Using the deck's Hours Studied (X) vs. Test Score % (Y) dataset:

- **Candidate 1: Y = 0.00 + 1.00X** — clearly too steep and starts too low; predictions miss badly for most points.
- **Candidate 2: Y = 2.00 + 1.00X** — same slope, shifted up — still too steep for this data's actual spread.
- **Candidate 3: Y = 2.00 + 0.40X** — better shape, but not yet minimizing the total error across all points.
- **Final best-fit line: ŷ = 15.79 + 0.9760x** — this is the one specific line that minimizes the sum of all the squared vertical distances between the actual points and the line (this method is called **"least squares"** — squared, again, for the same reason variance/std dev use squares: it penalizes big misses much more than small ones, and it avoids positive and negative errors canceling out).

**Interpreting the final equation, ŷ = 15.79 + 0.9760x:**
- **Intercept (15.79):** a student who studied 0 hours is predicted to score 15.79%. (Often not practically meaningful on its own — it's just where the line crosses the axis — but it's part of the equation you need for any prediction.)
- **Slope (0.9760):** each additional hour of studying is associated with a **0.976-percentage-point increase** in predicted test score.
- **Making a prediction:** plug in any X. E.g., 10 hours studied → ŷ = 15.79 + 0.9760(10) = **25.55%**. (Illustrative only — always sanity-check a prediction against the actual range of X used to build the model; predicting far outside that range, called *extrapolation*, is unreliable.)

---

## Block 6 — Calculating the Regression Line by Hand

**Why work through this by hand at least once:** Excel's `SLOPE`/`INTERCEPT`/`LINEST` functions (and the Data Analysis ToolPak's Regression tool) will do this instantly, but doing it manually once cements *what* those functions are actually computing — the same "understand the engine before trusting the shortcut" principle behind this whole project.

**The formulas:**

**b₁ (slope) = (nΣxy − ΣxΣy) / (nΣx² − (Σx)²)**

**b₀ (intercept) = (Σy − b₁Σx) / n**

**What you need to build first — five running columns from your raw data:** x, y, x², y², and xy. You sum each column, then plug the five sums (n, Σx, Σy, Σx², Σxy) into the formulas above. (Σy² isn't needed for the coefficients themselves, but it's used later if you want to compute R² by hand too.)

**Deck's worked example — the same Hours Studied / Test Score dataset, n = 10:**

- Σx = 371 (sum of hours studied across all 10 students)
- Σy = 520 (sum of test scores across all 10 students)
- (Σx², Σxy computed the same way — summing the x² and xy columns across all 10 rows)

Plugging into the formulas produces:

**b₁ = 0.9760**
**b₀ = 15.79**

→ **ŷ = 15.79 + 0.9760x** — the exact same final equation from Block 5, now reached by hand instead of by inspection.

**The mechanical process, step by step, for any new dataset you build at Supagas:**
1. Lay out x and y as two columns.
2. Add three more columns: x², y², xy — computed row by row.
3. Sum each of the five columns (n = row count).
4. Plug the five sums into the b₁ formula first (b₁ depends only on the sums, not on b₀).
5. Plug b₁ into the b₀ formula.
6. Write the final equation ŷ = b₀ + b₁x.

---

## How Section 4 connects to the forecasting roadmap

This section is the bridge between "pure statistics" and the forecasting techniques coming next:

- **Regression-based forecasting** is exactly this section's Block 5 equation, used forward: once you've fit ŷ = β₀ + β₁x on historical data, you plug in a *future* x (next month's promotional spend, next week's temperature forecast, days remaining until a known event) to get a demand forecast — this is a fundamentally different approach from the time-series methods (moving average, exponential smoothing) coming next, which use *only past values of the demand itself* and ignore external drivers entirely.
- **Identifying demand drivers at Supagas** starts with correlation (Block 1): before building any regression model, you'd correlate candidate drivers (weather, day-of-week, days-to-holiday, promotional activity, price changes) against historical demand, keep the ones with real, sensible (causally plausible — Block 3) relationships, and discard the rest.
- **R² (Block 4) becomes your model-quality report card** — "this driver-based model explains 77% of demand variation" is a concrete way to communicate forecast reliability to stakeholders, and the unexplained portion is exactly what safety stock (via the PI logic from Section 3, Block 6) needs to cover.
- **The genealogy preview:** simple linear regression (one driver) is the simplest member of a family that extends to **multiple regression** (several drivers at once) and eventually to more flexible machine learning models — each step trading added complexity for the ability to capture more of the real-world pattern, at the cost of needing more data and more care against overfitting (previewed in Block 4).

---

## Reference — Excel functions for Section 4

| Function | What it does |
|---|---|
| `CORREL(array1, array2)` | Calculates r directly |
| `RSQ(known_ys, known_xs)` | Calculates R² directly |
| `SLOPE(known_ys, known_xs)` | Calculates b₁ (the slope) |
| `INTERCEPT(known_ys, known_xs)` | Calculates b₀ (the intercept) |
| `LINEST(known_ys, known_xs)` | Returns the full regression statistics array (slope, intercept, R², standard errors) in one array formula |
| `TREND(known_ys, known_xs, new_xs)` | Returns predicted y-values for new x-values, using the fitted regression line |
| `FORECAST.LINEAR(x, known_ys, known_xs)` | Predicts a single y-value for one new x — the direct "make one forecast" version of TREND |
| Data Analysis ToolPak → **Regression** | Full regression output in one dialog: coefficients, R², standard errors, p-values for each coefficient, ANOVA table — the all-in-one tool once you're comfortable with what each piece means from this notebook |

---

**Next up (forecasting methods):** moving average → weighted moving average → exponential smoothing (simple → double/Holt → triple/Holt-Winters) → regression-based forecasting (the direct continuation of this section's Block 5–6). Each one will be taught by showing what it fixes about the method before it, and what it still gets wrong — the same genealogy approach used for Section 3's hypothesis tests.

# Forecasting Notes — The Moving Average Family (SMA, WMA, and other variants)

*Companion notebook to the Section 4 (Correlation & Regression) notes — this is the first block of the forecasting-methods roadmap. Covers every "moving average" variant worth knowing, not just the two we walked through interactively, with tips/tricks, worked examples, and a quick-recall cheat sheet at the end.*

---

## Block 0 — Foundation recap (the one idea every variant below is built on)

Historical demand = **signal** (real underlying level, possibly with trend/seasonality) + **noise** (random fluctuation with no predictive value). **Averaging cancels noise** because random ups and downs partially offset each other when summed. Every "moving average" method below is a different strategy for *which* values to average and *how much weight* to give each one — that's the entire design space.

**The universal trade-off to remember, no matter which variant:** more smoothing (bigger window / older data weighted more) → less noise, but slower to react to real change. Less smoothing (smaller window / recent data weighted more) → faster reaction, but more noise gets through. **You cannot get both for free — every choice here is a dial, not a fix.**

---

## Block 1 — Simple Moving Average (SMA)

**What it is:** the average of the last **n** actual periods, with every period counted **equally**.

**Formula:**

**SMA_t = (X_(t-1) + X_(t-2) + ... + X_(t-n)) / n**

The forecast for the next period = this average.

### Tips & tricks to remember SMA

- 🧠 **"Oldest drops, newest joins."** Each time you move the window forward one period, you drop the oldest value in the window and add the newest actual — the window always has exactly n values in it.
- 🧠 **"n periods before your first forecast."** You cannot produce a forecast until you have n actual periods of history. A 12-month MA needs a full year of clean data before it can output anything — this is the direct cost of choosing a longer window.
- 🧠 **"A level shift takes exactly n periods to fully absorb."** If demand jumps to a new sustained level, the SMA forecast will not catch up immediately — it needs n periods for every stale value in the window to be replaced by the new level. **This is the single most important fact to remember about SMA**, because it's easy to think "MA reacts within its window" when it actually only reacts gradually across the window.
- 🧠 **Window length is a trade-off, not a "correct" number:**

| Shorter n (e.g., 3) | Longer n (e.g., 12) |
|---|---|
| Reacts fast to real changes | Reacts slowly — lags behind real shifts |
| Still noisy/jumpy | Very smooth |
| Needs less history to start | Needs more history to start |

- 🧠 **SMA is only a one-step-ahead method in practice.** You can technically copy the formula further down, but past the first forecast period it starts averaging its own forecasts instead of real actuals — the deck's book calls this "stretching it too far," and the quality degrades fast. Don't use SMA to forecast multiple periods into the future.
- 🧠 **What it structurally can't do:** trend and seasonality. Every period in the window counts the same, so a genuine upward trend always makes SMA **under-forecast**, and a genuine downward trend always makes it **over-forecast** — predictably, every time, not just as bad luck.

### Worked example — Supagas cylinder demand, 3-month SMA

| Month | Actual | 3-Month SMA (= forecast for next month) |
|---|---|---|
| Jan | 500 | — |
| Feb | 540 | — |
| Mar | 480 | — |
| Apr | 610 | (500+540+480)/3 = **506.7** |
| May | 550 | (540+480+610)/3 = **543.3** |
| Jun | 520 | (480+610+550)/3 = **546.7** |

**Level-shift proof (why "n periods to fully absorb" matters):** if July jumps to a new sustained 700 and stays there:

| Month | Actual | 3-Month SMA forecast |
|---|---|---|
| Aug | 700 | (550+520+700)/3 = **590** |
| Sep | 700 | (520+700+700)/3 = **640** |
| Oct | 700 | (700+700+700)/3 = **700** ✅ (took exactly 3 months = n) |

---

## Block 2 — Weighted Moving Average (WMA)

**What it fixes about SMA:** SMA treats a value from 3 months ago exactly the same as last month's — but recent data is almost always more informative. WMA assigns **bigger weights to more recent periods**.

**Formula:**

**WMA_t = (w₁·X_(t-1) + w₂·X_(t-2) + ... + wₙ·X_(t-n)) / (w₁ + w₂ + ... + wₙ)**

### Tips & tricks to remember WMA

- 🧠 **"Biggest weight always goes to the most recent period."** If you're ever unsure which end of your weight list is "recent," this is the rule — get it backwards and you make the forecast *worse* than plain SMA, not just less good (proven in the worked example below).
- 🧠 **Always divide by the sum of the weights**, not by n. If your weights are 3, 2, 1 (they don't have to sum to 1), you divide by 6 — this is what keeps it a true weighted *average* rather than an inflated total. If you pick weights that already sum to 1 (like 0.5, 0.3, 0.2), you can skip the division — it's already built in.
- 🧠 **Common weighting schemes** you'll see in practice:
  - **Linear decreasing** — e.g., 3/2/1 (most recent gets 3x weight of the oldest). Simple, easy to justify to a stakeholder.
  - **Percentage-based** — e.g., 50%/30%/20%, chosen by judgment or by testing which mix gives the lowest historical forecast error.
  - There's no formula that hands you the "correct" weights — **choosing them is a judgment call**, and that's exactly WMA's weakness (see below).
- 🧠 **The cost of faster reaction: more noise gets through too.** Because WMA leans hardest on the newest data, a single freak recent month (a data error, a one-off bulk order) now has outsized influence on the forecast — you've traded away some of SMA's noise-cancelling power to gain responsiveness. Neither is "better" — it's context-dependent.

### Worked example — same data, weights 3(most recent)/2/1(oldest)

Forecasting August, using May=550, Jun=520, Jul=700:

**WMA_Aug = (3×700 + 2×520 + 1×550) / (3+2+1) = (2100+1040+550) / 6 = 3690/6 = 615**

Compare to plain SMA's 590 on the same three months — WMA already sits closer to the new level, in the *same first period*, because July counts 3x as much as May.

### ⚠️ The trap — reversed weights make things worse, not neutral

If you accidentally weight the **oldest** month heaviest (1-recent/2/3-oldest) instead:

**WMA_Aug = (1×700 + 2×520 + 3×550) / 6 = (700+1040+1650)/6 = 3390/6 = 565**

565 is *further* from the true new level (700) than even plain SMA's 590. **Getting the weighting direction backwards doesn't just fail to help — it actively underperforms the simpler method.** Always double-check which end of your weight list lines up with "most recent" before trusting the output.

---

## Block 3 — Other variants in the Moving Average family (as requested — the full list)

These are less commonly used for day-to-day demand forecasting than SMA/WMA, but each solves a specific problem and is worth knowing exists.

### 3a. Double Moving Average (DMA) — "MA-of-MA," a trend-corrected fix for SMA's lag

**The problem it targets:** you already proved SMA under/over-forecasts a real trend, predictably. Double Moving Average fixes this **without leaving the moving-average family at all** — it's a classical alternative to Holt's method, using only averages.

**How it works — take a moving average of the moving average:**
1. Compute **M1_t** = a normal n-period SMA of the raw data (exactly like Block 1).
2. Compute **M2_t** = an n-period SMA of the M1 series itself (average the averages).
3. Combine them to back out a de-lagged level and an explicit trend estimate:

**Level: a_t = 2×M1_t − M2_t**
**Trend (per period): b_t = (2/(n−1)) × (M1_t − M2_t)**
**Forecast, m periods ahead: F_(t+m) = a_t + b_t × m**

- 🧠 **Tip to remember:** M2 always lags M1 the same way M1 lags the raw data. The gap between them (M1 − M2) is itself a measurement of "how much lag/trend is being hidden" — that's why doubling M1's distance from M2 and subtracting it out (2×M1 − M2) recovers a trend-corrected level, instead of the lagging one.
- **Why this matters conceptually:** it's the exact same insight as Holt's method (track level *and* trend separately) — just implemented with plain averages instead of exponential weighting. Good to know this exists as a bridge, but Holt's method (already covered) is generally preferred in practice because it needs less history and reacts more smoothly.

### 3b. Centered Moving Average (CMA) — for measuring, not forecasting

**The problem it targets:** SMA as defined above only looks *backward* (uses the last n periods to estimate "now"), which is fine for forecasting the future but introduces lag when your goal is instead to measure the **true underlying trend-cycle at a point you've already observed** — e.g., to later calculate a **seasonal index** (this is exactly the technique behind classical seasonal decomposition, and a preview of what Holt-Winters is doing internally).

**How it's different:** instead of averaging only the n periods *before* time t, you average periods **centered around** t — using data from both before and after. For an even n (like 12, for monthly data), this needs a small extra step (average two consecutive n-period averages together) to land exactly on a real time period instead of between two of them.

- 🧠 **Tip to remember:** CMA needs **future data relative to the point you're centering on** — so **you cannot use it for real-time forecasting** (you don't have next month's actual yet). It's a *measurement/analysis* tool, used after the fact to extract a clean trend-cycle line or to calculate seasonal indices from historical data — not a live forecasting method. Don't confuse it with SMA when someone says "moving average" in a seasonality context.

### 3c. Cumulative Moving Average — the "running average"

**What it is:** the average of **every single data point from the very start** up through the current period — the window never stops growing.

**Formula:** CMA_t = (X_1 + X_2 + ... + X_t) / t

- 🧠 **Tip to remember:** as t grows large, each *new* individual data point has less and less influence on the average (dividing by an ever-bigger t) — so this becomes **extremely smooth but extremely slow to react**, the most extreme "long window" end of the trade-off spectrum. Rarely useful for demand forecasting specifically (a real business almost always cares more about recent behavior than an average of its entire history), but you'll see it in contexts like tracking a running average defect rate or a cumulative average score.

### 3d. Exponentially Weighted Moving Average (EWMA) — the bridge to your next topic

Worth knowing this name exists: **EWMA is mathematically the same thing as Simple Exponential Smoothing** (already covered in the previous session) — every past value gets a weight that decays exponentially the further back it is, generated by a single smoothing constant instead of a hand-picked list. Some textbooks file it under "the moving average family" (since it's still fundamentally a weighted average of past values) rather than as its own category — don't let the different name confuse you if you see it called this elsewhere. Full detail is in the Exponential Smoothing notes, not repeated here.

---

## Block 4 — Quick-recall cheat sheet

| Method | What's different | Reacts to real change | Handles trend? | Handles seasonality? | Needs future data? |
|---|---|---|---|---|---|
| **SMA** | Equal weight, fixed window n | Slow (lags n periods on a shift) | ❌ No — always lags | ❌ No | No |
| **WMA** | Custom weights, recent weighted more | Faster (if weighted correctly) | ❌ No — still lags, just less | ❌ No | No |
| **Double MA** | MA-of-MA, explicit trend term | Faster, trend-corrected | ✅ Yes | ❌ No | No |
| **Centered MA** | Averages before *and* after the point | N/A — not a forecast method | — | Used to help *measure* it | ✅ Yes (that's why it can't forecast live) |
| **Cumulative MA** | Window grows forever from period 1 | Extremely slow | ❌ No | ❌ No | No |
| **EWMA (= Simple Exp. Smoothing)** | Exponentially decaying weights, one constant (α) | Tunable via α | ❌ No (see Holt's for that) | ❌ No | No |

---

## Block 5 — Excel reference

| Task | Function / approach |
|---|---|
| Simple Moving Average | `=AVERAGE(range)`, copied down; or Data Analysis add-in → **Moving Average** tool |
| Flexible-length SMA (change window without rewriting formulas) | `=AVERAGE(OFFSET($A2,0,0,$C$1,1))` — $C$1 holds the window length n |
| Weighted Moving Average | No built-in function — use `=SUMPRODUCT(weights_range, values_range)/SUM(weights_range)` |
| Cumulative Moving Average | `=AVERAGE($A$2:A2)`, copied down (the mixed reference `$A$2` locks the start, `A2` grows as you copy) |
| Centered Moving Average | Build the SMA column first, then average each pair of consecutive SMA values (or use a shifted-average formula) — usually done manually in a helper column since Excel has no direct tool for it |

---

## Block 6 — Common pitfalls (mirror these against your own understanding)

1. **"A 3-month MA reacts to a level shift within 3 months, fully."** ❌ Wrong — it reacts *gradually*, taking the full n periods to completely absorb the change (see the Aug→Oct table in Block 1).
2. **"Weighting recent data more always makes WMA better than SMA."** ❌ Wrong — only if you weight it in the *correct direction*. Reversed weights make WMA actively worse than SMA (Block 2's trap).
3. **"I can use a moving average to forecast several months ahead by copying the formula further down."** ❌ Wrong — SMA/WMA are one-step-ahead methods; stretched further, they start averaging forecasts instead of actuals and degrade quickly.
4. **"Centered Moving Average is just another way to forecast."** ❌ Wrong — it needs data from *after* the point you're calculating, so it can't be used live. It's an analysis tool, most often used to extract seasonal indices from history.
5. **"A longer window is always 'more accurate.'"** ❌ Wrong — it's smoother, not more accurate. Smoother helps when your data is mostly noise; it hurts when real trend/level changes are happening, because it lags them longer.

---

**Next up:** Triple Exponential Smoothing (Holt-Winters) — picking up exactly where we left off, now with this full moving-average picture locked in as the foundation underneath it.

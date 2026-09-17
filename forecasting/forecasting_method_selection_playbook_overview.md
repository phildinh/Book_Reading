# Forecasting Notes — Method Selection & Blending Playbook (Overview)

*Companion notebook to MA/WMA, Exponential Smoothing, and Diagnostic Tools. This is the synthesis: given a real product's data, how do you actually decide which method to use, and how do you combine methods instead of picking just one? This is the **overview** version — concepts, decision rules, tips. A companion **detailed version** (with full worked examples) follows separately.*

---

## Block 0 — The core mindset shift

Every earlier notebook taught you **one method at a time, in isolation**. Real forecasting work is the opposite: you rarely know in advance which method fits, and you almost never rely on just one. This playbook is about **the decision process itself** — diagnosis → selection → blending → monitoring — not a new formula.

🧠 **The single sentence to remember: pick the method the evidence supports, not the method you like best or the most powerful one available.** Complexity that isn't justified by evidence is a cost (more constants to tune, more history required, more overfitting risk), not a free upgrade.

---

## Block 1 — The 6-step process (your operating procedure)

1. **Check history length.** No history at all → skip straight to judgmental forecasting (analogous product, sales estimate) until real data accumulates.
2. **Run the diagnostics** (from the Diagnostic Tools notebook): ACF at lag-1 and lag-p, regression-on-time p-value for trend, ANOVA across seasonal groups for seasonality.
3. **Let the diagnosis + history length select candidate method(s)** — see the decision matrix below.
4. **Fit 2–3 candidates, hold back recent actuals, compare errors** on data the method never saw (forecast accuracy metrics — next notebook). Don't just trust one method by assumption.
5. **Apply the judgmental overlay** — the statistical output is a *baseline*, never the final answer. Humans know about the future (new contracts, promotions, shutdowns) that no historical model can see.
6. **Re-diagnose periodically.** More history can reveal a pattern that wasn't detectable before (e.g., crossing the 2-full-seasons threshold needed for Holt-Winters).

🧠 **Tip:** steps 1–3 answer "which method(s) are even eligible." Step 4 answers "which eligible one actually performs best on my data." Never skip straight from step 1 to step 5 — evidence, then human judgment, in that order.

---

## Block 2 — The decision matrix

| Diagnosis result | History available | Method |
|---|---|---|
| No real trend, no real seasonality | Any | SMA or SES |
| Real trend, no real seasonality | ≥2 periods | Holt's method |
| Real trend + real seasonality | ≥2 full seasons | Holt-Winters |
| Trend + seasonality suspected | <2 full seasons | Can't run Holt-Winters yet — use Holt's + a rough manual seasonal adjustment, or wait for more data |
| Strong, causally sensible external driver exists | Any | Regression — alone or blended with a time-series method |

🧠 **Tip — the "eligibility gate" to always check first:** Holt-Winters is only *eligible* once you clear the 2-full-seasons history requirement (the initialization gotcha from the Exponential Smoothing notebook). Don't let "the data looks seasonal" override "I don't have enough history to prove or safely use it yet."

---

## Block 3 — The four blending strategies (the important part)

| Blend type | What it combines | Why it works |
|---|---|---|
| **Statistical + Judgmental** (S&OP consensus forecast) | Model baseline + human knowledge of the future | The model only ever sees the past; a person can add information the data structurally can't contain (a new contract, a shutdown) |
| **Ensemble** | Two or more statistical forecasts, averaged | Different methods make different kinds of errors; averaging partially cancels them — often beats the single "best" model |
| **Time-series + Regression** | Internal pattern (trend/season) + an external driver | Captures both "what demand does on its own" and "what an outside factor is doing to it" |
| **Segmented** | Different methods for different products, deliberately | Not every SKU deserves the same forecasting effort — match model complexity to business stakes |

🧠 **Tip — the counter-intuitive one to remember:** you don't need to find "the one best method." **Averaging two mediocre methods often beats picking the single best one**, because their errors tend to point in different directions and partly cancel out. This is a documented finding in forecasting practice, not just a shortcut for when you're unsure.

🧠 **Tip — proportional effort:** a high-value, high-volume SKU justifies Holt-Winters + regression + tight monitoring. A low-value, stable SKU is fine with plain SMA — matching effort to stakes is itself part of doing this well, not a shortcut you're taking.

---

## Block 4 — The full genealogy, one more time, now as a single map

```
SMA → WMA → SES → Holt's → Holt-Winters
   (equal weight → arbitrary weights → auto-decay → +trend → +seasonality)

        ↓ diagnosed and selected by ↓

  ACF (Tool 1)  +  Regression/ANOVA hypothesis tests (Tool 2)

        ↓ never trusted alone — always finished with ↓

  Judgmental overlay  +  possibly ensemble/regression blending
        ↓
  Final consensus forecast
```

---

## Block 5 — Common pitfalls

1. **"The most powerful method (Holt-Winters) is always the safest default."** ❌ Unjustified complexity risks overfitting and needs history you may not have yet.
2. **"I found the best method — no need to consider others."** ❌ Blending often beats "the best single method" outright.
3. **"The statistical forecast is the final number."** ❌ It's a baseline. The judgmental overlay is a required step, not an optional nice-to-have.
4. **"All my SKUs deserve the same forecasting rigor."** ❌ Effort should scale with business stakes, not be applied uniformly.

---

**Next:** the detailed companion version — same structure, with full worked examples applying this framework to realistic Supagas scenarios. After that: Forecast Accuracy & Uncertainty (MAPE/MAD/RMSE), the tool that powers Step 4 above.

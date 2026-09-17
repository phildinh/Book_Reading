# Forecasting Notes — Method Selection & Blending Playbook (Detailed, with Worked Examples)

*Companion to the Overview playbook. Same framework, now applied end-to-end to three realistic Supagas scenarios, with full worked numbers — this is the "how do I actually do this on Monday morning" version.*

---

## Quick recap of the framework (full detail is in the Overview notebook)

1. Check history length → 2. Run diagnostics (ACF, regression trend test, ANOVA) → 3. Select candidate method(s) from the decision matrix → 4. Compare candidates on held-out error → 5. Apply judgmental overlay → 6. Re-diagnose periodically.

🧠 **Tip to hold onto through every example below:** the framework never changes — only the *inputs* (how much history, what the diagnostics say, how much the product matters) change what comes out of it.

---

## Scenario A — High-volume seasonal cylinder line (4 years monthly, strong signals)

**The situation:** a core cylinder product. Diagnosis already run (this is the same quarterly dataset from the Diagnostic Tools notebook, standing in for "4 years of clear signal"): ACF lag-4 = 1.00 (seasonality confirmed, though flagged as small-sample), ACF lag-1 = 0.17 (not significant), ANOVA F = 11.53 > critical 6.59 (seasonality confirmed a second, independent way).

**Step 3 — method selection:** Trend confirmed + seasonality confirmed + ≥2 full seasons of history → **Holt-Winters is eligible and indicated.**

**Step 4 — the baseline forecast (already built in the Exponential Smoothing notebook):**

| Year 3 | Holt-Winters forecast |
|---|---|
| Q1 | 235.4 |
| Q2 | 298.0 |
| Q3 (peak) | 363.3 |
| Q4 | 308.1 |

**Step 4, continued — build a second candidate to compare/blend against.** Using the same L₈=282.9, T₈=6.96 but **without the seasonal multiplier** (i.e., what Holt's method alone would say):

| Year 3 | Holt's-only forecast (L₈ + m×T₈) |
|---|---|
| Q1 | 289.9 |
| Q2 | 296.8 |
| Q3 | 303.8 |
| Q4 | 310.7 |

🧠 **Tip:** notice Holt's-only forecast is nearly flat (289.9 → 310.7) — it has no idea Q3 should spike. This is the direct, visible cost of ignoring seasonality on data that has it.

**Ensemble blend — average the two candidates:**

| Year 3 | Holt-Winters | Holt's-only | **Ensemble average** |
|---|---|---|---|
| Q1 | 235.4 | 289.9 | **262.7** |
| Q2 | 298.0 | 296.8 | **297.4** |
| Q3 | 363.3 | 303.8 | **333.6** |
| Q4 | 308.1 | 310.7 | **309.4** |

**Why you might actually want this blend here, not just the pure Holt-Winters number:** recall the ACF lag-4 correlation of 1.00 was flagged as coming from only 4 data pairs — a fragile basis for trusting the seasonal index completely. Blending in the flatter Holt's-only view **hedges against the possibility that the seasonal swing is somewhat overstated** from too little history. As more years accumulate and the seasonal index gets confirmed on more data, you'd lean the blend weight back toward pure Holt-Winters.

**Step 5 — judgmental overlay, made concrete:** suppose Sales confirms a major client is expanding their order volume by 15%, starting in Q3. Apply it to the statistical number:

**Final Q3 consensus forecast = 363.3 × 1.15 ≈ 417.8** (using the Holt-Winters number here, since this is a *known, confirmed* future event, not a hedge against data uncertainty — judgmental adjustments should sit on top of your best statistical estimate, not a hedged blend).

🧠 **Tip — don't confuse the two kinds of adjustment:** ensemble blending hedges against **statistical uncertainty in the model itself**. The judgmental overlay incorporates **real information about the future the model can't see**. Both are legitimate, but they answer different questions — don't apply a client-expansion adjustment to a hedged/averaged number "just to be extra safe" without thinking through which uncertainty you're actually addressing.

---

## Scenario B — Brand-new product, 3 months of history, similar to an existing line

**The situation:** a new cylinder size, only 3 months of actuals — nowhere near enough for any of the statistical methods (all need real history; Holt-Winters specifically needs 2+ full seasons). But it's chemically/functionally similar to an existing, well-established product line whose seasonal shape you already know (reuse the earlier indices: S₁=0.80, S₂=1.00, S₃=1.20, S₄=1.00).

**Step 1 says stop and use judgmental forecasting — but "judgmental" doesn't have to mean "pure guess."** This is where **analog/proxy forecasting** earns its place: borrow the *shape* of a similar, established product's seasonal pattern, and scale it using the new product's own early actuals.

**Worked example:** the new product's first quarter (Q1) actual sales = 50 units. The established line's Q1 seasonal index is 0.80 (Q1 always runs 20% below its own normal level). Use that to back out an *implied normal (de-seasonalized) level* for the new product:

**Implied level = 50 ÷ 0.80 = 62.5**

Now project the rest of Year 1 using the *established product's seasonal shape*, applied to this new implied level:

| Quarter | Borrowed seasonal index | Forecast (62.5 × index) |
|---|---|---|
| Q2 | 1.00 | 62.5 |
| Q3 (peak) | 1.20 | 75.0 |
| Q4 | 1.00 | 62.5 |

🧠 **Tip — this technique has a real name and a real limitation:** "analog forecasting" is standard practice for new products, but it only works if the analog product is **genuinely comparable** — same customer base, same use case, same seasonal drivers. Borrowing the shape from a product that only *looks* similar on the surface (same material, different customer segment) can produce a confident-looking forecast that's confidently wrong. The judgment call of "is this really a good analog" matters more than the arithmetic here.

**Step 6, applied:** once this new product accumulates 2+ full years of its own actuals, re-run the diagnostics (Tool 1/Tool 2) on its *own* data and graduate to its own Holt-Winters model — don't keep leaning on the borrowed shape forever.

---

## Scenario C — Stable, low-volume spare-parts SKU, 6 years of history, nothing significant

**The situation:** 6 years of history (plenty), but ACF shows nothing significant at any lag, and the regression-on-time p-value is 0.71 (nowhere near significant).

**Step 3 — method selection:** No real trend, no real seasonality → **SMA or SES is not just "acceptable," it's correctly indicated by the evidence.** Reaching for Holt-Winters here would mean fitting three components (and three tuning constants) to a pattern that's genuinely just noise around a flat level — pure overfitting risk for zero benefit.

**Worked example — 6-month SMA on recent actuals:** 40, 38, 42, 41, 39, 40.

**SMA = (40+38+42+41+39+40)/6 = 240/6 = 40** → forecast next month = **40**.

**Step applying "segmented blend" (proportional effort) explicitly:** this SKU is explicitly low-value and low-volume. **The correct decision here is to *not* invest further effort** — no ensemble, no regression driver search, no judgmental fine-tuning beyond a sanity check. The skill being demonstrated isn't running more sophisticated math; **it's correctly recognizing when sophistication isn't warranted.**

🧠 **Tip — this is the scenario people most often get wrong, in the opposite direction from Scenario A:** the temptation is to apply the same rigor to every SKU "to be thorough." That's not thoroughness — it's misallocated effort. Reserve deep diagnostic + blending work for the SKUs where forecast accuracy actually moves the business needle (high volume, high value, high supply risk).

---

## Consolidated tips & tricks across all three scenarios

- 🧠 **The decision matrix output is a starting candidate, not a final answer** — Scenario A shows even a "correctly selected" Holt-Winters forecast can be worth blending against a simpler candidate when the underlying evidence (small-sample ACF) has a known weakness.
- 🧠 **No history ≠ no forecast** — Scenario B shows judgmental forecasting can still be evidence-informed (borrowing a real seasonal shape) rather than a pure guess.
- 🧠 **Passing every diagnostic test with "nothing significant" is itself a valid, useful result** — Scenario C shows that confirming "no real pattern" correctly steers you toward the *simpler* method, saving effort that would otherwise be wasted.
- 🧠 **Ensemble blending hedges model/data uncertainty. Judgmental overlay incorporates real future information. Know which one you're doing, and don't substitute one for the other.**
- 🧠 **Effort should scale with the product's business stakes, not be applied uniformly across your whole portfolio** — the same forecasting rigor that's correct for Scenario A would be wasted on Scenario C, and insufficient (without the analog technique) for Scenario B.

---

## Common pitfalls (expanded with these examples in mind)

1. **"Since I have 6 years of data (Scenario C), I should use the most sophisticated method available."** ❌ More history doesn't create trend/seasonality that isn't there — the diagnostics said "nothing significant," and that result should be trusted, not overridden by having "enough data to try something fancier."
2. **"A new product with no history can't be forecast at all."** ❌ Analog forecasting (Scenario B) gives you a reasonable, evidence-informed starting point — just don't treat it as equally reliable as a model built on the product's own confirmed history.
3. **"If I've already blended in a hedge (ensemble), I don't also need the judgmental overlay."** ❌ These solve different problems (model uncertainty vs. real future information) and are typically both needed, in sequence — hedge first, then adjust for known future events on top.
4. **"The seasonal index from Holt-Winters is definitely correct once the F-test/ACF confirms seasonality exists."** ❌ Confirming seasonality is *real* (Scenario A's ANOVA/ACF) is a different question from confirming the *exact size* of the seasonal index is reliable — the ensemble blend exists precisely to hedge that second, separate uncertainty.

---

**Next on the roadmap:** Forecast Accuracy & Uncertainty (MAPE/MAD/RMSE) — the tool that turns "which candidate performed better" (Step 4 throughout this playbook) into an actual measured comparison instead of a judgment call.

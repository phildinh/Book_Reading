# Section 3 — Hypothesis Testing: Notebook Reference

*Built from the Hypothesis Testing slides, corrected and expanded through Q&A. This section is split into two arcs: **Arc A (below)** covers the shared logic every single test in the deck reuses. **Arc B** (added as we cover it) walks through each of the 12 specific tests: what it's for, its assumptions, its formula, and a worked example.*

---

# ARC A — FOUNDATIONS (the logic every test shares)

## Block 1 — H₀, H₁, Significance Level (α), and the P-value

**The business problem:** every day at work you'll face some version of "did something actually change, or am I looking at normal random noise?" Hypothesis testing is the formal, repeatable procedure for answering that without fooling yourself.

**Why H₀ must be the "boring" default, mechanically — not just philosophically:** you can only calculate a probability (a p-value) from a hypothesis that names *one exact number*. H₀ always states something precise (μ = 150, p = 0.5, μ₁ = μ₂) — that precision is what lets you build one specific, known probability distribution to test your data against. H₁ is usually a range ("there's a difference"), which has no single distribution to compute from. So it's not just caution — you literally *cannot* compute a p-value from H₁.

**The mechanical trick for declaring H₀ correctly, every time: H₀ always keeps the "=" sign** (or ≤/≥, which still includes equality). H₁ always gets whatever's left — ≠, <, or >. Try writing your suspicion as a formal statement with a number and a comparison symbol; whichever side naturally contains "=" is H₀. If your suspicion itself contains ≠, <, or >, that confirms it's H₁.

**The second check, in plain words: whatever you're trying to find evidence FOR is always H₁.** The "nothing new happened" claim — regardless of which fact is mentioned first in the story — is always H₀.

**One-tailed nuance:** when your suspicion has a direction, H₀ gets the *entire opposite range*, not just the single "=" value. Testing for an increase → H₀: μ ≤ μ₀, H₁: μ > μ₀. Testing for a decrease → H₀: μ ≥ μ₀, H₁: μ < μ₀. Testing for "different, don't know which way" → H₀: μ = μ₀, H₁: μ ≠ μ₀ (two-tailed).

**P-value** = P(seeing this data, or more extreme, IF H₀ is true). **Decision rule: p-value < α → reject H₀. p-value ≥ α → fail to reject H₀.** Memory hook: **"Small p, problem for H₀."**

**We never "accept" H₀ — courtroom analogy:** "fail to reject" is like a "not guilty" verdict — it doesn't prove innocence, it means the evidence wasn't strong enough to convict. H₀ could still be false; your sample might have just been too small or noisy to catch it (this is exactly what Type II error / power, Blocks 2–3, are about).

**Critical precision point — statistics never proves anything, it only quantifies risk:**
- Reject H₀ → *"Sufficient evidence to support H1, with a known α chance this conclusion is a false alarm."* Not "H1 is proven true."
- Fail to reject H₀ → *"Not enough evidence to support H1 in this data."* Not "H1 is proven false" — the effect could be real but undetected (a Type II error risk).
- Proof: if "reject H₀" meant certain truth, Type I error (rejecting a true H₀) couldn't exist. It does exist — so rejection is always a probabilistic call, never a guarantee.

---

## Block 2 — Type I and Type II Errors

No test eliminates risk entirely — you only choose which kind of mistake you're more willing to risk.

**Memory hook: Type I = False Alarm. Type II = Missed Alarm.**

| | Reality: H₀ true | Reality: H₀ false (H₁ true) |
|---|---|---|
| **You reject H₀** | **Type I Error (α)** — false alarm | Correct decision (this rate = Power) |
| **You fail to reject H₀** | Correct decision | **Type II Error (β)** — missed alarm |

**Fire-alarm example (from the deck):** H₀ = "no fire." Alarm rings, no fire → Type I. Alarm silent, real fire → Type II.

**The trade-off:** lowering α (fewer false alarms) increases β (more missed real problems) — you can't reduce both by moving the same dial. The only lever that improves both at once is a **bigger sample size**.

**Which error matters more is a business judgment, not a stats fact** — it depends on the relative cost of a false alarm vs. a missed detection in that specific situation (e.g., a false vendor accusation costs a relationship; a missed real slowdown costs ongoing delays). This is *why* α gets chosen deliberately per context, not defaulted to 0.05 blindly.

---

## Block 3 — Power of the Test

**Power = 1 − β.** Extending the same fire-alarm story: **β = your miss rate. Power = your catch rate.** They always add to 100% of "the times a real fire actually exists."

**Two separate imagined worlds are needed to compute this** (unlike α/p-value, which only need "if H₀ is true"):
- **World 1 (H₀ true):** your data clusters around μ₀. α = how often this world's data still crosses your cutoff purely by chance.
- **World 2 (H₁ true, at a *specific* assumed value you must choose — the data can't hand you this):** your data clusters around some true alternative value μ₁. Power = how much of *this* world's data lands in the "reject" zone. β = how much still lands in the "fail to reject" zone, purely from unlucky sampling.

**What increases power:**
- **Bigger sample size** → narrower spread on both worlds' curves → less overlap → real effects are less likely to be hidden in noise.
- **Bigger effect size** (true value further from μ₀) → less overlap between the two worlds → easier to tell apart.
- **Higher α** → also raises power, but at the cost of more Type I error — same dial, opposite trade-off (not a free improvement, unlike sample size).

**Target power (deck standard): 0.80** — an 80% chance of detecting a real effect, if one exists.

**Practical use — the one thing to always remember:** a "fail to reject H₀" result from a **low-power** test is weak evidence of "nothing's happening" — it might just mean the test wasn't sensitive enough. Trust a null result more when the test's power was high.

*(Formal β calculation — recentering the sampling distribution at a chosen μ₁ and finding how much of it falls on the "wrong" side of the original cutoff — is deeper mechanics, parked for now. Revisit when a real sample-size-planning problem calls for it; the concept above is sufficient for interpreting test results day to day.)*

---

## Block 4 — Statistical vs. Practical Significance

Two separate questions, easy to conflate:
- **Statistical significance** — "is this difference real, or just chance?" (p-value vs. α)
- **Practical significance** — "is this difference big enough to be worth the cost/effort of acting on it?" (effect size + business judgment, no formula)

**The trap:** with a large enough sample, even a trivial, meaningless difference can become statistically significant (p-value shrinks just because n is huge). *Deck example: a 0.1% sales improvement may be statistically real but not worth the cost of change.* Always ask both questions before committing resources — proven real is not the same as worth doing.

---

## Block 5 — Sample Size Calculation

**The business problem:** every CI/test needs data you have to plan for in advance — too little and your margin of error is useless; too much wastes time and money.

**Formula (for a mean):**

**n = (Z_α/2 × σ / E)²**

- **E** = margin of error (how much precision you want)
- **Z_α/2** = Z-value for your confidence level (1.96 for 95%, 2.576 for 99%)
- Round the result **UP** always — a fractional sample doesn't meet your target.

**Two independent dials, both push n up:**
- **Smaller E (tighter precision)** → bigger n. E is *squared* in the denominator — halving E roughly *quadruples* n.
- **Higher confidence level** → bigger Z_α/2 → bigger n.

**Worked example (deck):** margin of error = 1 hour, 95% confidence, σ = 4 hours → n = (1.96×4/1)² = 61.46 → **round up to 62**.

**Proportions version:** n = (Z_α/2² × p(1−p)) / E² — same shape, using p(1−p) in place of σ². *Worked example: p=0.5, E=0.05, 95% confidence → n = 385.*

**Framing to keep:** this isn't "get as much data as possible" — it's "find the smallest sample that meets my specific precision and confidence targets" before spending the resources to collect it.

---

## Block 6 — Point and Interval Estimates: Confidence Interval vs. Prediction Interval

**Why this matters directly for forecasting:** a point estimate ("demand will be 500 units") hides how much to trust it. Forecasting is fundamentally an interval-estimate problem, and picking the *right kind* of interval is the actual skill.

**The distinguishing test: is the question about the AVERAGE/overall behavior of the whole group (CI), or about ONE specific individual/future case (PI)?** Language cue: "average," "overall," "true rate," "across all X" → CI. "This one," "the next," "a single," "will it be" → PI.

| | Confidence Interval (CI) | Prediction Interval (PI) |
|---|---|---|
| Formula | x̄ ± Z_α/2 × (σ/√n) | x̄ ± Z_α/2 × σ × √(1 + 1/n) |
| Answers | "Where's the TRUE AVERAGE?" | "Where will ONE NEW value land?" |
| As n → ∞ | Shrinks toward 0 width (estimation uncertainty is reducible) | Never shrinks below a floor set by σ (individual variability is irreducible — no amount of averaging removes it) |
| Example use | "True average PO line items across all orders" | "Line items on the next single order" / "next month's demand" |

**Worked example (deck):** mean=70, s=10, n=25, 95% confidence → CI = [66.08, 73.92]; PI = [50.01, 89.99] (much wider, same data).

**The most common misinterpretation — do not make this mistake:** a 95% CI does **NOT** mean "95% of individual values fall in this range." It means "95% confident the TRUE AVERAGE falls in this range." With enough data, a CI can become very narrow even when individual values are scattered widely — the CI only reflects uncertainty about the average's location, not the spread of individuals. Only a PI describes where individual values land.

**Forecasting is (almost always) a PI problem, not a CI problem** — "next month's demand" is one specific future value, not the long-run average. Using a CI here understates real risk and would size safety stock dangerously too tight.

**How this is actually used in an S&OP setting:** the statistical model's **point estimate** becomes the baseline forecast number; the **PI's width** feeds directly into safety stock sizing (wide PI → bigger buffer needed); the **final "consensus forecast"** blends the statistical baseline with judgmental input from Sales/Marketing (known deals, promotions, market shifts the historical data can't see) — statistics gives a defensible starting point and an honest sense of uncertainty, not the final word.

---

## Block 7 — The General Hypothesis Testing Process & Test Taxonomy

**Every specific test in Arc B is the same 5-step skeleton, wearing a different formula:**

1. **State H₀ and H₁**
2. **Choose α**
3. **Calculate the test statistic** — a standardized value: (observed − what H₀ predicted) / typical random variation. This is a generalized Z-score; when true σ is unknown (almost always), it becomes a **t-statistic** instead — same idea, fatter-tailed distribution to cover the extra uncertainty.
4. **Find the critical value** — the cutoff on that distribution set by α (same concept as the cutoff used in the β/power picture).
5. **Decision** — test statistic vs. critical value, or equivalently, p-value vs. α.

**The master map — three questions choose the test:**
1. One sample, two samples, or 3+?
2. Testing a mean, a proportion, or a variance?
3. Is the population variance known, or only the sample's?

| | Testing a... | Test |
|---|---|---|
| **One sample** | Mean (σ known, or n≥30) | One-Sample Z |
| | Mean (σ unknown, small n) | One-Sample t |
| | Proportion | One-Proportion |
| | Variance | One-Variance (Chi-square) |
| **Two samples** | Means (σ known/large n) | Two-Sample Z |
| | Means (σ unknown) | Two-Sample t (pooled or Welch's) |
| | Means, related/dependent data | Paired t |
| | Proportions | Two-Proportions |
| | Variances | Two-Variances (F) |
| **3+ samples** | Means | ANOVA (F) |

*(Two more Chi-square categories appear later in the deck: Goodness-of-Fit and Contingency Tables/Independence — for testing patterns in categorical data rather than a specific numeric parameter.)*

---

## Reference only — Excel functions for Section 3, Arc A

- Critical values: `NORM.S.INV(probability)` (Z), `T.INV(probability, df)` (t), `CHISQ.INV(probability, df)`, `F.INV(probability, df1, df2)`
- Probabilities/CDFs: `NORM.S.DIST(z, TRUE)`, `T.DIST(t, df, TRUE)`, `CHISQ.DIST(x, df, TRUE)`, `F.DIST(x, df1, df2, TRUE)`
- Direct hypothesis-test helpers: `Z.TEST`, `T.TEST`, `CHISQ.TEST`, `F.TEST` (built-in shortcuts that return p-values directly)
- Confidence interval: `CONFIDENCE.NORM(alpha, std_dev, size)` (known σ), `CONFIDENCE.T(alpha, std_dev, size)` (unknown σ)

---

# ARC B — Coming Next

The 12 specific tests, each covered as: what it's for → assumptions → formula → worked example. Not yet covered — to be added here as we go:

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

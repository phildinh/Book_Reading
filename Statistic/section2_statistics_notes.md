# Section 2 — Probability: Notebook Reference

*Built from the Probability slides, corrected and expanded through Q&A. Foundation-first — every concept ties back to "why does historical data let me say anything about the future at all?"*

---

## Block 1 — Population vs. Sample Notation, and the Central Limit Theorem

**Notation:** the deck uses separate symbols for population vs. sample so you never confuse "the true, usually-unknowable value" with "my estimate of it": **μ** (population mean) vs. **x̄** (sample mean), **σ** vs. **s**, **N** vs. **n**. In practice, almost all forecasting work calculates *sample* statistics — you never have "all future demand" to measure directly.

**Central Limit Theorem (CLT):** if you take sums or averages of enough independent random values, that sum/average approaches a **normal (bell-shaped) distribution — regardless of the shape of the original data.**

*Demonstrated with dice:* a single die's outcomes are perfectly flat (uniform) — every face equally likely. The sum of just 2 dice already forms a triangle. The sum of 3 dice already looks close to a smooth bell curve. The reason: hitting an *extreme* sum (like all dice showing 6) requires every single die to line up the same way, and that probability shrinks fast as you add dice `(1/6)ⁿ`. Hitting a *middle* sum has combinatorially many more ways to get there, and that count grows even faster as dice are added — so extremes get rarer, middles get more crowded, and the shape smooths into a bell curve.

**Why it matters:** daily demand data can be as messy, skewed, or multimodal as reality wants (recall the lumpy retail/bulk mix from Section 1). But the *average* of that demand over many days — a sample mean — becomes increasingly normal as you include more days, even though the raw daily numbers never were. This is **why** normal-distribution tools (z-scores, confidence intervals) remain usable on real, messy business data at all.

**Direct consequence:** a confidence interval built from the average of 60 days of demand is more trustworthy than one built from 3 days — more data in the average means closer to normal, means more trustworthy. This is the same intuition (small sample = extra caution needed) that reappears formally in Block 7 as the reason the t-distribution exists.

---

## Block 2 — Core Probability Language

**Sample space** = every possible outcome. **Event** = an outcome (or group of outcomes) you care about. **Complement (A′)** = everything *not* in A, so P(A) + P(A′) = 1.

**Mutually exclusive** = events that cannot both happen in one trial (non-overlapping in a Venn diagram) — e.g., a SKU is either in-stock or in stockout on a given day, never both. **Independent** = knowing one event happened doesn't change the probability of the other — e.g., a delivery truck's mechanical breakdown and a customer's decision to place an order. These are answers to *different questions*, and any two real mutually exclusive events are automatically **dependent** (never independent), because knowing one happened tells you the other definitely didn't.

The case that matters most in demand planning is **dependent, but not mutually exclusive** — both events can happen together, and knowing one changes your estimate of the other. *Example: "demand was high last month" and "demand is high this month" can obviously co-occur, and they're dependent — last month being strong genuinely shifts your estimate for this month.*

**Why this is the whole reason forecasting works:** forecasting only makes sense because consecutive time periods are *dependent*, not independent. If this month's demand were truly independent of every prior month — like separate dice rolls — there'd be no pattern to learn from, and history would be useless for predicting the future. Every technique on the forecasting roadmap (moving average, exponential smoothing, everything after it) is really a method for **measuring and exploiting the dependence between time periods.**

**Union (A∪C, "OR")** = everything in A or C or both — like a SQL `UNION` / full join. **Intersection (A∩C, "AND")** = only what's in both at once, in a *single* item — like a SQL `INNER JOIN`. *Example: Order #123 being both a repeat-customer order and a weekend order is one order satisfying both conditions at once — not a comparison between two separate orders.*

**Counting trap:** if A and C aren't mutually exclusive, you can't just add their counts to get the union — you'd double-count the overlap. *Example: 40 repeat-customer orders, 25 weekend orders, 10 that are both → union = 40 + 25 − 10 = 55, not 65.* That subtraction is exactly the addition rule formalized in Block 3.

---

## Block 3 — Multiplication Rule, Addition Rule, Conditional Probability

**Addition rule ("OR"):** P(A∪B) = P(A) + P(B) − P(A∩B). The subtraction exists purely to avoid double-counting the overlap. If A and B are mutually exclusive, P(A∩B) = 0, so it simplifies to P(A) + P(B) — no correction needed because there's no overlap to double-count.

**Multiplication rule ("AND"):** always multiply — the only question is *which* second number to use:
- **Independent events:** P(A∩B) = P(A) × P(B). *Example: rolling a die and flipping a coin — 1/6 × 1/2 = 1/12.*
- **Dependent events:** P(A∩B) = P(A) × P(B|A). *Example: drawing 2 yellow candies in a row without replacement — 1/5 × 1/9 = 1/45; the second draw's probability changed because the first draw removed a candy from the pool.*

**Conditional probability:** P(A|B) = P(A∩B) / P(B) — "given B already happened, what fraction of that also has A?" This is the *same relationship* as the dependent multiplication rule above, just rearranged algebraically — not a separate idea to memorize twice.

---

## Block 4 — Permutations vs. Combinations

**Why these formulas exist at all:** for small cases you can just list every possibility by hand (like 2 dice). Real problems — choosing 3 products out of 50 to feature, or counting possible 4-digit PINs — have too many arrangements to list manually. Permutations and combinations are systematic shortcuts for counting "how many ways can this happen," which then feeds directly into probability (favorable outcomes ÷ total outcomes).

**The one question that decides everything: does order matter?**
- **Order matters → Permutation.** P(n,r) = n! / (n−r)!. *Example: choosing 1st, 2nd, 3rd place from 8 competitors — being 1st vs. 2nd is a genuinely different outcome.*
- **Order doesn't matter → Combination.** C(n,r) = n! / [r! × (n−r)!]. *Example: choosing a 3-person committee from 8 people — there's no ranking, just group membership.*

**The relationship, made concrete:** any one group of 3 people (say Alice, Bob, Carol) can be arranged in 3! = 6 different orders (Alice-Bob-Carol, Alice-Carol-Bob, Bob-Alice-Carol, Bob-Carol-Alice, Carol-Alice-Bob, Carol-Bob-Alice). All 6 orderings are different *permutations*, but they collapse into exactly **1 combination** — same three people, order thrown away. So: **C(n,r) = P(n,r) / r!** — every combination gets "inflated" by r! when order starts to matter, and dividing by r! removes that inflation.

*Worked example: C(8,4) = P(8,4) / 4! = 1,680 / 24 = 70 — each of those 70 committees corresponds to 4! = 24 different orderings.*

**Litmus test for real problems (used again in Block 5):** ask "does swapping which item comes first create a genuinely different, distinguishable real-world outcome?" If the items just share a status (like "defective" — a piece doesn't have a rank as "1st defective" or "2nd defective," it's just broken or not) → combination. If they're a true sequence of distinguishable events (1st/2nd/3rd place) → permutation.

**With repetition allowed (less common, but in the deck):**
- Permutation with repetition: n^r. *Example: a 4-digit PIN, digits 0–9, repeats allowed → 10⁴ = 10,000 possible PINs.*
- Combination with repetition: C(n+r−1, r). *Example: 3 scoops of ice cream from 5 flavors, doubles allowed → C(7,3) = 35.*

Excel: `PERMUT(n,r)`, `PERMUTATIONA(n,r)` (with repetition), `COMBIN(n,r)`, `COMBINA(n,r)` (with repetition).

---

## Block 5 — Binomial and Poisson Distributions

Both distributions are really just Block 3 (multiplication rule) and Block 4 (combinations) fused into one formula — not new ideas to memorize from scratch.

**The business problem that motivates the Binomial formula:** a manufacturer has a 12% defect rate. A buyer tests 20 random pieces and accepts the batch only if 2 or fewer are defective. You need P(exactly 2 defective out of 20).

**Building it from what you already know:**
1. **One specific arrangement** (say, pieces #1 and #2 defective, the rest fine) is pure multiplication rule: P = p² × (1−p)¹⁸ = 0.12² × 0.88¹⁸.
2. **But you don't care *which* 2 pieces are defective** — just that exactly 2 out of 20 are. The number of different pairs that could be "the defective ones" is a combinations question: C(20,2) = 190.
3. Each of those 190 arrangements has the *same* probability, and they're mutually exclusive outcomes (only one specific pair is actually defective in a real batch), so you add them all up — equivalent to multiplying.

**Binomial formula: P(X=x) = C(n,x) × pˣ × (1−p)ⁿ⁻ˣ**

*Worked result:* P(X=0) ≈ 7.76%, P(X=1) ≈ 21.15%, P(X=2) ≈ 27.40% → P(accept the batch) = P(0)+P(1)+P(2) ≈ **56.3%**. A 12% defect rate with only a 2-defective tolerance on 20 pieces is a genuinely tight bar — the batch fails almost half the time.

**Requirements for Binomial:** a fixed number of trials (n), each trial binary (success/failure), constant probability (p), trials independent of each other. Mean = n×p, Variance = n×p×(1−p).

**Poisson distribution** — for counting events over a continuous interval (time or space) when you know the **average rate (λ)**, with **no natural "number of trials"** to define. P(X=k) = (λᵏ × e⁻λ) / k!. Uniquely, mean = variance = λ.

**The test for which to use:** can you name a fixed, finite, countable set of trials? → Binomial. Is it really "events happening over time/space at some average rate," with no natural trial count? → Poisson. Forcing a Poisson-shaped problem into Binomial would mean inventing an arbitrary n and shrinking p to compensate — Poisson skips that entirely.

**Honest relevance check for Supagas work:**
- Neither is a core forecasting engine — moving average, exponential smoothing, and regression are the time-series workhorses, and they don't use Binomial/Poisson math directly.
- **Binomial** shows up for yes/no outcome questions, not demand levels: *"what's the probability more than 2 of 20 cylinder deliveries are late this month?"* (an SLA question), or quality acceptance sampling — exactly the worked example above.
- **Poisson** is more directly useful, but for a specific slice: modeling **order counts for slow-moving or intermittent-demand SKUs**, where a smooth "average per day" doesn't really apply. This connects straight back to the lumpy-demand segmentation problem from Section 1 — Poisson-based approaches are part of how professionals handle that specific class of item, distinct from smooth continuous-demand items that moving average/exponential smoothing handle well.

Both are foundation pieces (item 1 on the roadmap) and occasionally the right situational tool — not the daily forecasting engine, which comes later in the roadmap.

---

## Block 6 — Normal Distribution, Z-scores, and the Empirical Rule

This is the destination Block 1's Central Limit Theorem was pointing toward — the shape itself, formalized so you can actually calculate probabilities from it.

**Normal distribution properties:**
- **Symmetric** — perfectly mirrored on both sides of the center.
- **Mean = Median = Mode**, all sitting at the exact center of the curve. (This doubles as a diagnostic: if a real dataset's mean, median, and mode all land close together, that's evidence the data might be roughly normal — the same test introduced in Section 1, Block 3.)
- **Total area under the curve = 1** (100%).

**Empirical rule (68-95-99.7)** — for a normal distribution, the percentage of data within a given number of standard deviations of the mean:
- **68%** within **1σ** of the mean (μ ± 1σ)
- **95%** within **2σ** (μ ± 2σ)
- **99.7%** within **3σ** (μ ± 3σ)

*Example: adult male heights, mean = 70 in, std dev = 3 in → 68% of men are 67–73 in, 95% are 64–76 in, 99.7% are 61–79 in. Almost nobody falls outside ±3σ — that's the "practically never happens" zone.*

**The gap the empirical rule leaves:** it only gives round numbers at exactly 1, 2, or 3 standard deviations. For anything in between — e.g., "taller than 74 inches" when mean = 70, std dev = 3 — you need the **Z-score**.

**Z-score formula: Z = (X − μ) / σ**

*Example: Z = (74 − 70) / 3 = 1.33 → 74 inches is 1.33 standard deviations above the mean.*

**Reading the sign (falls straight out of the formula, no separate rule to memorize):**
- Negative Z = value is **below** the mean (X − μ is negative).
- Positive Z = value is **above** the mean.
- Z = 0 = value is exactly at the mean.

**What a Z-score is actually for:** it converts *any* value, from *any* normal distribution (any mean, any std dev), onto one universal "how many standard deviations from the mean" scale. That means one shared reference table (or one Excel function) works for every possible normal distribution, instead of needing a separate table for every mean/std-dev combination. This is the actual mechanism underneath confidence intervals and hypothesis testing.

*Full example: Z = (65 − 70)/3 = −1.67. By symmetry, P(Z < −1.67) = P(Z > +1.67) = 0.0475 → **4.75% of men are shorter than 65 inches.***

Excel: `NORM.DIST(x, mean, std_dev, TRUE)` for direct probability from raw values; `NORM.S.DIST(z, TRUE)` once already standardized to Z. Watch parentheses — the order of operations breaks the Z formula silently if dropped.

---

## Block 7 — t-distribution, Chi-square, F-distribution

These three aren't used directly very often on their own — their real job is powering the actual hypothesis tests in the next deck (t-tests, chi-square tests, ANOVA). The goal here is knowing *what each is for*, so test selection makes sense later instead of feeling arbitrary.

**t-distribution — builds directly on Block 1.** The Z-score formula assumes you know the *true population* standard deviation, σ. In real life — e.g., estimating average delivery lead time from 15 recent orders — you almost never know true σ; you only have `s`, the standard deviation calculated from your own sample. Using a small sample's `s` in place of the true σ introduces extra uncertainty the plain Z formula doesn't account for (the same "small sample = less trustworthy" intuition from Block 1's dice). The t-distribution has fatter tails than the normal curve specifically to absorb that extra uncertainty.

**The organizing question isn't "big or small sample → pick t or F."** It's **"what kind of question am I asking?"**

| Distribution | Question it answers | Sample-size role |
|---|---|---|
| **t** vs **Z** | Estimating/testing a **mean** | t when σ unknown (small OR large sample); Z only if true σ is known, or n is large enough that s ≈ σ |
| **Chi-square** | Categorical data (independence, goodness-of-fit), or testing a claim about a single **variance** | Not sample-size driven — about the *type* of question |
| **F** | Comparing **variances or means across multiple groups** (the engine behind ANOVA) | Not sample-size driven — about comparing groups |

**Forecasting/Supagas links:**
- t → confidence interval around a forecasted average demand.
- Chi-square → "is stockout reason independent of product category?" or "has demand variance changed?"
- F → "do different warehouses have significantly different demand variability?" (ANOVA territory)

Full mechanics (degrees of freedom, exact test setup) belong to the Hypothesis Testing deck — this block only needed to plant the map.

---

## Section 2 Genealogy — How Every Block Chains to the Next

1. **Population vs. sample + CLT (Block 1)** sets the whole section's foundation: you only ever calculate sample statistics, but CLT is *why* you're allowed to trust that a sample average behaves normally — even when the raw data doesn't.

2. **Core probability language (Block 2)** gives you the vocabulary (events, mutually exclusive, independent, union, intersection) needed before anything can be combined — and reveals the real reason forecasting works: consecutive time periods are *dependent*, not independent.

3. **Addition/multiplication rules + conditional probability (Block 3)** are the formal tools for *combining* the events named in Block 2 — literally just formalizing the double-counting fix you'd naturally reach for.

4. **Permutations vs. combinations (Block 4)** is a counting toolkit that answers "how many ways can this happen" for cases too big to list by hand — which is exactly the ingredient Block 5 needed.

5. **Binomial and Poisson (Block 5)** are Block 3's multiplication rule and Block 4's combinations *fused into one formula*, applied to two different real-world shapes of problem: fixed trials (Binomial) vs. rate-over-time/space (Poisson).

6. **Normal distribution, Z-scores, empirical rule (Block 6)** is the formalized destination Block 1's CLT was always pointing toward — turning "sums/averages approach normal" into an actual, usable probability tool via one universal Z scale.

7. **t, Chi-square, F (Block 7)** are the machinery that takes Block 6's normal-distribution logic and adapts it for the messier, more realistic case where the true population standard deviation is unknown (t), the question is about categories or variance (Chi-square), or you're comparing multiple groups at once (F) — setting up the Hypothesis Testing deck directly.

**The one-sentence version:** Section 1 was "how do I honestly describe what I've observed." Section 2 is "how do I reason about uncertainty and combine probabilities" — and its endpoint (the normal distribution, reached via the Central Limit Theorem) is the exact mathematical justification for why confidence intervals and hypothesis tests are allowed to exist at all, which is where the next deck picks up.

---

## Reference only — Excel functions for Section 2 (not yet covered in depth)

- Counting: `PERMUT(n,r)`, `PERMUTATIONA(n,r)`, `COMBIN(n,r)`, `COMBINA(n,r)`
- Binomial: `BINOM.DIST(x, n, p, cumulative)`
- Poisson: `POISSON.DIST(x, mean, cumulative)`
- Normal: `NORM.DIST(x, mean, std_dev, TRUE)`, `NORM.S.DIST(z, TRUE)`, `NORM.S.INV(probability)` (reverse lookup: probability → Z)
- t / Chi-square / F (previewed here, used properly in Hypothesis Testing): `T.DIST`, `CHISQ.DIST`, `F.DIST` and their inverse (`.INV`) counterparts

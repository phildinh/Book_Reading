# Chapter 6 — Order Management: Real-World Complexity

## What This Chapter Covers

Chapter 6 is where dimensional modelling gets messy — because real business data is messy. Orders have multiple dates. Transactions have dozens of low-cardinality flags. Facts exist at different grains on the same form. This chapter gives you the patterns to handle all of it cleanly.

---

## Role-Playing Dimensions

### The Problem

Most fact tables have more than one date. An order has: order date, requested ship date, actual ship date, invoice date. An accumulating snapshot might have six. Each is a different business concept, but all are dates.

**SQL cannot join the same physical table twice with two different FKs.** If you write:
```sql
JOIN date_dim d ON d.date_key = f.order_date_key
JOIN date_dim d ON d.date_key = f.ship_date_key  -- same alias = error
```
SQL requires both dates to be identical, which defeats the purpose.

### The Solution — SQL Views with Unique Column Names

Maintain one physical date dimension table. Create a named view per role, renaming every column:

```sql
CREATE VIEW order_date AS
SELECT 
    date_key      AS order_date_key,
    day_of_week   AS order_day_of_week,
    month         AS order_month,
    year          AS order_year
FROM date_dim;

CREATE VIEW ship_date AS
SELECT 
    date_key      AS ship_date_key,
    day_of_week   AS ship_day_of_week,
    month         AS ship_month,
    year          AS ship_year
FROM date_dim;
```

Now the query uses two logically distinct dimensions:
```sql
JOIN order_date od ON od.order_date_key = f.order_date_key
JOIN ship_date sd  ON sd.ship_date_key  = f.ship_date_key
```

**Why column renaming matters:** without unique column names, when a user drags "month" from both roles into a Power BI report, there is no way to tell which month is the order month and which is the ship month.

**In Power BI:** create separate inactive relationships for each date FK column. Activate with `USERELATIONSHIP()` inside the relevant DAX measures.

---

## Junk Dimensions

### The Problem

Every transactional source system generates low-cardinality flags and indicators alongside the real data. Payment type (Cash/Credit). Order type (Inbound/Outbound). Commission flag (Yes/No). Online flag (Yes/No).

**Options:**
- Leave them on the fact row as raw codes → cryptic, unreadable in Power BI
- Leave them as full text on the fact row → bloats every row with descriptive strings
- Create a separate dimension for each flag → 8 flags = 8 extra FKs, 8 extra tables, 8 extra joins

### The Solution — One Junk Dimension

Combine all flags into one dimension. Each unique COMBINATION of flag values becomes one row:

```
junk_key | payment_type | order_type | commission_flag | online_flag
---------|--------------|------------|-----------------|------------
1        | Cash         | Inbound    | Commissionable  | Online
2        | Cash         | Inbound    | Non-Comm        | Online
3        | Cash         | Outbound   | Commissionable  | Online
...
```

**Result:** fact table has ONE extra FK column (`junk_key`) instead of four. One join to get all four attributes.

**ETL logic:** when loading a fact row, look up whether this combination already exists in the junk dimension. If yes, use that key. If no, insert a new row and get the new key.

**When junk dimensions stop working:** if five attributes each have 100 possible values → 100⁵ = 10 billion combinations. At that point, separate dimensions are better. The guideline: junk dimensions work when the total combination count is tractable (under a few thousand rows).

---

## Header/Line Anti-Patterns

Real transactional data almost always has a two-level structure: one header per transaction, multiple line items per header. This creates two common design mistakes.

### Anti-Pattern A — Treating the Header as a Dimension

```
PO Header Dimension (one row per PO)  ←  PO Line Fact (one row per line item)
  - PO Number (PK)                         - PO Header Key (FK)
  - PO Date                                - Product Key (FK)
  - Vendor Name                            - Quantity
  - Contract Terms                         - Extended Amount
  - Freight Charge
```

**What goes wrong:**
- The header dimension grows at nearly the same rate as the fact table (5 lines per PO → 1 header row per 5 fact rows). Normal dimensions should be orders of magnitude smaller than fact tables.
- To filter lines by vendor or date, you must traverse the large header dimension — slow and confusing.
- Header-level facts (freight charge) are trapped at the wrong grain and can't be sliced by product.

### Anti-Pattern B — Treating the Header as Its Own Fact Table

```
PO Header Fact  ←─(join on PO number)─→  PO Line Fact
```

**What goes wrong:** joining two fact tables directly. Cartesian product produces wrong numbers. This is a fact-to-fact join — the silent killer from Chapter 2.

### The Correct Approach — Push Everything to the Line

Every header-level dimension becomes a direct FK on the line fact table. Header-level facts get allocated proportionally to lines. The PO number becomes a degenerate dimension.

```
PO Line Fact (correct):
  date_key        (FK → Date dim)      ← pushed down from header
  vendor_key      (FK → Vendor dim)    ← pushed down from header
  contract_key    (FK → Contract dim)  ← pushed down from header
  product_key     (FK → Product dim)
  po_number       (DD)                 ← degenerate, no dim table
  line_number     (DD)                 ← degenerate, no dim table
  quantity
  extended_amount
  allocated_freight                    ← header freight ÷ extended amount ×  line amount
```

**Freight allocation example:**
- Order total = $1,970. Header freight = $30.
- Line 1 (60.5% of total) → allocated freight = $18.15
- Line 2 (39.5% of total) → allocated freight = $11.85

Allocated freight is now at line grain, fully additive, sliceable by product.

**The degenerate PO number is still useful:** GROUP BY po_number to see all lines on one order, count lines per order, trace back to the source system.

---

## Facts at Different Granularity

When a form has both header-level facts and line-level facts, the correct approach is allocation — push the header-level fact to line level via a business rule.

**If allocation is impossible (no agreed rule):** create a separate header-grain fact table. Never mix granularities in a single fact table.

> **Rule:** Do NOT repeat the unallocated header fact on every line (overcounting risk). Do NOT store it only on the first or last line (disappears when those lines are filtered out). Allocate or separate.

---

## Accumulating Snapshot with Lag Facts

### Lag Facts — Pre-Computed Pipeline Metrics

Store the number of days between each milestone pair directly on the accumulating snapshot fact row. Calculate when the milestone completes. Store permanently.

**Why:** "Average days from application to enrolment by course" — with stored lag:
```sql
SELECT course_name, AVG(application_to_enrolment_lag)
FROM pipeline_fact f
JOIN course_dim c ON c.course_key = f.course_key
WHERE application_to_enrolment_lag IS NOT NULL
GROUP BY course_name
```

Without stored lag: two date dimension joins, DATEDIFF, CASE to exclude TBD rows — fragile, repeated in every query, easy to get wrong.

**Lag examples for an order fulfilment pipeline:**
- `order_to_manufacturing_lag` (days)
- `manufacturing_to_inventory_lag` (days)
- `inventory_to_shipment_lag` (days)
- `order_to_shipment_lag` (days) ← end-to-end efficiency measure

---

## The Audit Dimension

### The Problem

Every ETL load involves processing decisions: some records had missing fields that were defaulted, some dollar amounts were outside the expected range, some records came from a newer version of the transformation script. None of this is visible in the fact table — every row looks identical regardless of data quality.

### The Solution — Audit Dimension

A small dimension capturing ETL processing metadata. Added to the fact table as a FK.

```
audit_dim:
  audit_key (PK)
  quality_indicator     ('Clean', 'Warning', 'Anomaly')
  fee_defaulted         ('Yes', 'No')
  source_system_matched ('Yes', 'No')
  pipeline_version      ('v1.0', 'v2.0', ...)
  load_batch_id
```

**How it works:** ETL determines which audit row applies to each fact row during loading, stamps the `audit_key`, done. Never changes after that.

**What it enables in Power BI:**
- Business users add `quality_indicator` as a slicer → filter to 'Clean' rows only → high-confidence numbers
- Data team runs `GROUP BY quality_indicator, pipeline_version` → instant pipeline health report
- When someone questions a number → filter the audit dimension → see exactly how that row was produced

**Why not just log files?** Log files live outside the dimensional model. Business users can't join to them, Power BI can't filter on them. The audit dimension brings data quality metadata inside the model, making it queryable like any other dimension.

---

## Multiple Currencies and Units of Measure

When facts need to be expressed in both local currency and a standard corporate currency, carry both on the same fact row:

```
fact_table:
  extended_gross_local_amount
  extended_gross_standard_amount  ← converted at load time using rate for that day
  local_currency_key (FK → Currency dim)
```

**Why not just store a conversion rate and let users multiply?** Users will apply the rate incorrectly. The DW should pre-compute the converted values and present them directly.

For multiple units of measure (shipping cases vs retail units vs consumer units): store base quantity in one unit plus conversion factors on the same row. Expose derived quantities through SQL views.

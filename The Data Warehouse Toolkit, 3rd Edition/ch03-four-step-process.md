# Chapter 3 — Retail Sales: The Four-Step Design Process

## Why This Chapter Matters

Chapter 3 introduces the four-step design process — the engine of all dimensional design. Every dimensional model you ever build starts here, in this exact order, with no shortcuts. The retail grocery case study is just the teaching vehicle. The patterns apply everywhere.

---

## The Four-Step Process

### Step 1 — Select the Business Process

Ask: **what operational activity are we measuring?**

A business process is a specific operational activity — not a department, not a report, not a KPI.

| Wrong | Right |
|---|---|
| The Enrolment Department | Student enrolment (the activity of confirming a student into a course) |
| The Sales Report | POS transaction (the activity of a product being scanned at the register) |
| Marketing Analytics | Lead creation (the activity of a new prospect entering HubSpot) |

Each business process maps to one source system and one primary fact table.

### Step 2 — Declare the Grain

Ask: **what does exactly one row represent?**

This is the most important decision in the entire design. Lock this before touching dimensions or facts. Everything else depends on it.

**Correct grain declarations:**
- One product scanned on one POS transaction
- One student enrolled in one course in one intake
- One order line item on one purchase order

**How to test your grain declaration:**
- Can every dimension you want to add take exactly one value per row? If no — wrong grain.
- Does every fact column make sense at this level? If no — wrong grain.

**Always choose atomic grain.** Summaries can always be derived from atomic data. You can never recreate individual events from a pre-aggregated row.

### Step 3 — Identify the Dimensions

Ask: **who, what, where, when, how, why?**

Each dimension must:
- Take exactly ONE value per fact row at the declared grain
- Be non-numeric and descriptive (text, categories, dates)
- Contain attributes useful for filtering, grouping, and report labelling

If a dimension candidate takes multiple values per fact row → multivalued dimension → bridge table (see Chapter 8).

### Step 4 — Identify the Facts

Ask: **what is the process measuring?**

Facts must be:
- True to the declared grain
- Numeric and measurable
- As additive as possible

**Three types of facts:**
- Additive — SUM across every dimension (sales dollars, units sold)
- Semi-additive — SUM across some dimensions but NOT time (account balance, headcount)
- Non-additive — never SUM (unit price, ratio, percentage)

---

## Dimension Design Rules

### Build the Date Dimension First

The date dimension is the only dimension built before any data arrives. Pre-populate 10–20 years (~7,300 rows — trivially small).

**Standard columns:** date, day of week, day number in month, calendar month/quarter/year, fiscal month/quarter/year, holiday indicator, weekday indicator.

**Why not use SQL date functions?** SQL cannot tell you if a date is a public holiday, a fiscal quarter boundary, or a custom business event. That logic belongs in the dimension table, not in application code that every developer reinvents differently.

### Flags and Indicators — Always Full Text, Never Codes

| Bad | Good |
|---|---|
| `is_scholarship = 'Y'` | `scholarship_indicator = 'Scholarship Student'` |
| `delivery_mode = 'OL'` | `delivery_mode = 'Online'` |
| `payment_type = 1` | `payment_type = 'Upfront Payment'` |

Reason: these values appear directly in Power BI filter dropdowns and report labels. Business users should never see codes or abbreviations they need to decode.

### Null Handling

| Where | Rule | Reason |
|---|---|---|
| Dimension text attributes | Replace with `'Unknown'` or `'Not Applicable'` | Nulls disappear from filter lists and break GROUP BY comparisons |
| Fact numeric columns | Leave as null | SUM/COUNT/AVG handle null correctly; replacing with zero corrupts averages |
| Fact FK columns | NEVER null — point to a default row | Referential integrity; null FKs cause rows to disappear from joins |

### Denormalise Deliberately

In 3NF: Product → Brand table → Category table (three tables, two joins).
In dimensional: Brand name and Category name both sit in the Product dimension row.

**Why this is correct:**
- Dimension tables are geometrically smaller than fact tables — the space cost is negligible
- One-table lookup instead of three-table join
- Hierarchies in one place — filter by category means one WHERE clause on one table
- BI tools work natively with flat dimensions

---

## Surrogate Keys

Never use the source system's natural key as your dimension's primary key.

**Three reasons:**
1. Operational keys can be recycled over time — your history breaks
2. Integrating multiple sources requires a single universal key above all source systems
3. SCD Type 2 requires multiple rows for the same natural key — impossible without surrogate keys

The DW generates its own sequential integer key. It owns it completely.

---

## Degenerate Dimensions

A meaningful operational identifier that has no descriptive attributes — no reason for its own dimension table.

**Examples:** invoice number, order number, enrolment ID, receipt number.

**Where it lives:** directly on the fact table as a column, with no corresponding dimension table.

**What it enables:**
- Grouping all line items on a single transaction (`GROUP BY order_number`)
- Traceability back to the source system
- Sometimes part of the fact table's primary key

---

## Factless Fact Tables

A fact table with no numeric measurement columns — or a single artificial count of 1.

### Use 1 — Activity Events

Records that a set of dimensional entities came together at a point in time, with no natural measurement.

**Example:** Student attends a class session. Either they showed up or they didn't. No number to record.

```
attendance_fact:
  date_key (FK)
  student_key (FK)
  course_key (FK)
  instructor_key (FK)
  facility_key (FK)
  attendance_count = 1  ← artificial, always 1
```

COUNT rows to analyse attendance patterns.

### Use 2 — Coverage + What Didn't Happen

The more powerful use. Requires two tables working together:

**Coverage table** — every event that COULD have happened (all enrolled students × all scheduled sessions).
**Activity table** — every event that DID happen (students who actually showed up).

**Set difference = what didn't happen** (students who enrolled but never attended).

```sql
-- Students enrolled in a course but never attended
SELECT s.student_name, c.course_name
FROM coverage_fact cf
JOIN student_dim s ON s.student_key = cf.student_key
JOIN course_dim c ON c.course_key = cf.course_key
WHERE NOT EXISTS (
    SELECT 1 FROM attendance_fact af
    WHERE af.student_key = cf.student_key
    AND af.course_key = cf.course_key
)
```

This query is impossible using the activity table alone. If a student never attended, they have zero rows in the activity table — they are simply absent. You can only identify absence by comparing against what should have happened.

---

## Graceful Extensibility

When the fact table is designed at atomic grain with conformed dimensions, adding new dimensions later is straightforward:
- Create the new dimension table
- Add a new FK column to the fact table
- Populate historical rows with a default/unknown surrogate key
- All existing queries continue working unchanged

This is the practical payoff of getting grain right upfront.

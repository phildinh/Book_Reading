# Chapter 2 — Kimball Dimensional Modelling Techniques Overview

## How to Use This Chapter

Chapter 2 is a reference catalogue — a master list of every pattern Kimball has formalised, with pointers to where each is explained in depth. Think of it like a design pattern library. When you hit a design problem, come back here — there is almost certainly an established pattern for it.

---

## The Three Fact Table Types

This is the most important thing in Chapter 2.

### Transaction Fact Table
- One row per business event at a point in time
- Sparse — a row only exists if the event occurred
- Insert only — never updated after loading
- Best for answering: **"What happened and when?"**
- Date dimension: one transaction date FK

### Periodic Snapshot Fact Table
- One row per entity per fixed time period (daily, weekly, monthly)
- Dense — a row exists even if nothing changed (zero IS meaningful)
- Insert only — new rows stacked on top for each period
- Best for answering: **"What is the balance/level right now?"**
- Date dimension: one snapshot date FK

### Accumulating Snapshot Fact Table
- One row per pipeline/workflow instance (one application, one order, one claim)
- Updated repeatedly as milestones complete — the **only** fact table you ever intentionally update
- Multiple date FKs — one per milestone
- Best for answering: **"Where is this in the pipeline? How long did each stage take?"**

> **Critical distinction:** Transaction and Periodic Snapshot are INSERT only. Accumulating Snapshot requires a MERGE (upsert) in your ETL — look up existing row, update it.

---

## Fact Additivity

| Type | Definition | Examples | Aggregation |
|---|---|---|---|
| Additive | SUM across every dimension | Sales dollars, units sold, enrolment fee | SUM() always valid |
| Semi-additive | SUM across some dimensions but NOT time | Stock on hand, account balance, headcount | AVG() across time periods |
| Non-additive | Never SUM | Unit price, ratio, percentage, conversion rate | Calculate in BI layer |

> **Semi-additive trap:** Adding Monday's stock level to Tuesday's stock level produces a meaningless number. The correct time aggregation is AVERAGE. In Power BI DAX: use `AVERAGEX(ALL('Date'), [measure])`, not `SUM`.

---

## Dimension Table Techniques

### Surrogate Keys
Never use the source system's natural key as your dimension's primary key.

**Why:**
- Operational keys can be recycled or reused across time
- Integrating multiple sources means different systems have different keys for the same entity
- SCD Type 2 requires multiple rows per natural key — only possible with surrogate keys

The DW generates its own anonymous integer key (1, 2, 3...) and owns it entirely.

### Degenerate Dimensions
A meaningful identifier that has no attributes of its own — no dimension table behind it.

- Lives directly on the fact table as a column
- Useful for traceability back to source systems and for grouping
- Examples: invoice number, order number, enrolment ID, lead ID
- Rule: use for operational transaction control numbers only. Do not use for bulky text fields.

### Role-Playing Dimensions
One physical dimension table serving multiple logical roles in the same fact table.

**Most common example:** Date dimension used as order_date, ship_date, invoice_date — all on the same fact row.

**Problem:** SQL cannot join the same physical table twice with two different FKs. The engine requires both dates to be identical.

**Solution:** Create one SQL VIEW per role, each with unique column names:
```sql
CREATE VIEW application_date AS
SELECT date_key AS application_date_key,
       month AS application_month,
       year AS application_year
FROM date_dim;

CREATE VIEW offer_date AS
SELECT date_key AS offer_date_key,
       month AS offer_month,
       year AS offer_year
FROM date_dim;
```
Each view is semantically independent. One physical table, multiple logical dimensions.

### Junk Dimensions
A grouping of low-cardinality flags and indicators packed into one dimension.

**Problem:** A fact table has 8 binary flags (Y/N payment type, order type, commission flag, etc.). Options:
- Leave them on the fact row — cryptic codes, bloats every row
- Create a separate dimension for each flag — 8 extra FKs, 8 extra tables, 8 extra joins

**Solution:** Combine all flags into one junk dimension. Each unique COMBINATION of flag values = one row. One FK on the fact table.

```
Junk Dim Row 1: Online | Scholarship | Upfront | Domestic
Junk Dim Row 2: Online | Scholarship | Payment Plan | Domestic
Junk Dim Row 3: On Campus | No Scholarship | Upfront | International
...
```

ETL builds rows on-demand when new combinations appear. Works well when total combination count is tractable (under a few thousand rows).

### Snowflaking — Why to Avoid It
Normalising a dimension by splitting it into multiple linked tables. Looks "cleaner" to someone with a 3NF background.

**Why it's wrong for analytical models:**
- Adds joins for every query
- Confuses BI tools and Power BI's relationship engine
- Saves negligible disk space (dimensions are tiny compared to fact tables)
- Destroys bitmap index effectiveness on low-cardinality columns

**The one exception:** outrigger dimensions — a secondary date reference or a genuine separately-managed reference table. Use rarely and deliberately.

---

## Slowly Changing Dimensions (SCDs) — Overview

| Type | Mechanism | Preserves History | Use When |
|---|---|---|---|
| 0 | Never change | N/A | Truly immutable (original sign-up date) |
| 1 | Overwrite old value | No | Correction, history doesn't matter |
| 2 | Add new row | Yes — full | Must know what was true when the fact occurred |
| 3 | Add new column | Partial — one prior | Need current AND one previous value simultaneously |
| 4 | Mini-dimension | Yes — split off | Large dimension with rapidly changing attributes |
| 6 | Type 1 on Type 2 rows | Yes + current | Need history AND current value on every row |
| 7 | Dual keys in fact table | Yes + current | Same as Type 6 but via two FK columns |

**Types 1, 2, and 6 cover 95% of real work.** Types 3, 4, 5, 7 exist for specific edge cases.

### SCD Type 2 — Required Admin Columns
Every Type 2 dimension needs three extra columns:
- `row_effective_date` — first date this row's attributes were valid
- `row_expiration_date` — set to `9999-12-31` for the current row
- `current_row_indicator` — `'Current'` or `'Expired'`

---

## The Bus Architecture and Conformed Dimensions

### Bus Matrix
A grid where rows = business processes and columns = dimensions. An X in a cell means that dimension participates in that process.

**Purpose:**
- Architecture planning — shows the full enterprise data model at a glance
- Integration mechanism — shared dimensions across rows enable drill-across queries
- Project planning — each row is one implementation sprint
- Stakeholder communication — shows business and IT the master plan

### Conformed Dimensions
A dimension is conformed when it means exactly the same thing with every fact table it joins to — same column names, same attribute definitions, same domain values.

**Two flavours:**
- **Identical:** the exact same physical table (or synchronised replica) shared across multiple fact tables
- **Shrunken rollup:** a strict subset of attributes from the atomic dimension, used when a fact table operates at a higher grain (e.g. a Month dimension is a shrunken subset of the Date dimension)

### Drill-Across
Combining data from two separate fact tables in one report by:
1. Querying each fact table separately
2. Aggregating within each
3. Merging the results on shared conformed dimension attributes

> **Never join two fact tables directly.** The cartesian product inflates numbers silently. Always drill across — two queries, one merge.

---

## Common Anti-Patterns

**Centipede fact table** — too many FK columns because every level of a hierarchy gets its own dimension instead of being flattened into one dimension. Bloats the fact table.

**Fact-to-fact table joins** — joining two fact tables directly in SQL. Produces wrong answers without errors. Always use drill-across instead.

**Abstract generic dimensions** — one "Person" dimension for employees, customers, and vendor contacts. The attribute sets diverge, cardinality grows, queries get confusing. Keep them separate.

**Fact normalisation** — converting multiple numeric columns into rows with a "measurement type" dimension. Multiplies row count by the number of fact types. Makes arithmetic between facts extremely difficult in SQL. Avoid unless facts are extremely sparse and heterogeneous.

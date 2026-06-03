# Chapter 4 — Inventory: The Bus Architecture & Conformed Dimensions

## The Value Chain

Every organisation has a sequence of connected business processes — Kimball calls this the **value chain**. Each step produces its own measurements at its own grain with its own timing. Each step becomes its own fact table.

But every step shares something: **dimensions**. The product, the date, the location — these appear across multiple steps. This is the core insight of Chapter 4.

---

## The Three Fact Table Types — Side by Side

| | Transaction | Periodic Snapshot | Accumulating Snapshot |
|---|---|---|---|
| **Grain** | One event at a moment in time | State at end of a fixed period | One pipeline instance |
| **Density** | Sparse | Dense (row always exists) | Sparse or dense |
| **Updates** | Never | Never | Yes — row updated as milestones complete |
| **Date FKs** | One transaction date | One snapshot date | Multiple — one per milestone |
| **Best for** | "What happened?" | "What is the level/balance?" | "Where is this in the pipeline?" |
| **ETL pattern** | INSERT | INSERT | MERGE (upsert) |

> **The only rule that matters on updates:** Transaction and Periodic Snapshot are INSERT-ONLY forever. Accumulating Snapshot is the ONLY fact table you ever intentionally update.

---

## Semi-Additive Facts in Periodic Snapshots

The inventory case study introduces the most common semi-additive fact: **inventory quantity on hand**.

- You CAN sum across products (total inventory across all SKUs = valid)
- You CANNOT sum across time periods (adding Monday's stock to Tuesday's stock = meaningless)

**Correct time aggregation:** AVERAGE. Take the average inventory level across the time period, not the sum.

**In Power BI DAX:**
```dax
Average Monthly Balance = 
AVERAGEX(
    ALL('Date'[Month]),
    [Inventory Balance]
)
```
Never use `SUM` for semi-additive facts.

---

## The Enterprise Data Warehouse Bus Architecture

The bus architecture answers: *how do you build an enterprise DW incrementally, team by team, process by process, without ending up with disconnected silos?*

**Answer:** define the shared dimensions first. Build fact tables independently. Connect them through the shared dimensions.

### The Bus Matrix

A grid where:
- **Rows** = business processes (one per operational source system or activity)
- **Columns** = conformed dimensions (shared across processes)
- **X in a cell** = this process uses this dimension

**How to read it:**
- Scan a row → one fact table, all its dimensions
- Scan a column → one conformed dimension, all the fact tables that share it

**What it is used for:**
- Architecture planning — the enterprise DW blueprint
- Sprint planning — each row is one implementation project
- Data governance — each column is an invitation to the conformity meeting
- Stakeholder communication — shows the full picture without technical detail

### Common Bus Matrix Mistakes

| Mistake | Problem |
|---|---|
| Rows = departments | Departments have multiple processes; you lose granularity |
| Rows = reports | One process supports many reports; you build redundant fact tables |
| Columns too broad | A "Person" column covering employees, customers, and vendors — zero overlap |
| Columns for each hierarchy level | Use one Date column, not separate Day/Month/Year columns |

---

## Conformed Dimensions — The Integration Mechanism

### What "Conformed" Actually Means

Conformed means the same dimension means exactly the same thing with every fact table it joins to:
- Same column names
- Same attribute definitions  
- Same domain values (not "Sydney" in one and "SYD" in another)

**Two flavours:**

**Identical conformity** — the exact same physical table (or synchronised replica). Same keys, same rows, same everything. Used when both fact tables operate at the same grain.

**Shrunken rollup conformity** — a strict subset of attributes from the atomic dimension. Used when a fact table operates at a higher grain. Example: a Month dimension is a shrunken subset of the Date dimension. The shared attributes (month name, year) must have identical column names and values in both.

### Why Conformed Dimensions Enable Drill-Across

If the Product dimension is defined once and shared across the sales fact table and the inventory fact table, you can combine data from both in one report — because they share the same product keys, same product names, same category definitions.

This is called **drill-across**. Without conformed dimensions, each fact table has its own version of "product" and they can never be joined cleanly.

### Conformed Facts

If the same fact (revenue, headcount, cost) appears in multiple fact tables, its definition and units must be identical. If they are not identical, give them different names — labelling incompatible facts the same is one of the most dangerous DW mistakes.

---

## Drill-Across — How It Works

### The Correct Approach

1. Query fact table A, group by the shared conformed dimension attribute → Result set A
2. Query fact table B, group by the same shared conformed dimension attribute → Result set B
3. Full outer join Result A to Result B on the shared dimension attribute

**In Power BI:** this happens automatically when your model has both fact tables connected to the same conformed dimension. You never write the SQL manually.

### Why Direct Joins Break

If Fact Table A has 4 rows for Product X and Fact Table B has 2 rows for Product X, joining them directly produces 4 × 2 = 8 matched rows. Every metric from both tables is inflated. The result looks plausible but is silently wrong.

> **Rule:** Aggregation must happen within each fact table BEFORE the two results ever touch each other. Join first then aggregate = wrong. Aggregate first then join = correct.

---

## The Accumulating Snapshot — Milestone Tracking

### Structure

One row per pipeline instance. Multiple date FK columns, one per milestone:

```
pipeline_fact:
  received_date_key    (FK → Date dim)
  inspected_date_key   (FK → Date dim)  ← starts as 0 (TBD)
  completed_date_key   (FK → Date dim)  ← starts as 0 (TBD)
  product_key          (FK → Product dim)
  quantity_received    (fact)
  receipt_to_inspected_lag  (fact, in days)  ← calculated at update time
  receipt_to_completed_lag  (fact, in days)  ← calculated at update time
```

### Lag Facts

Pre-calculate and store the number of days between milestone pairs directly on the fact row. ETL calculates when the milestone completes. Never recalculate at query time.

**Why:** "Average days from application to enrolment by course" — with stored lag: `AVG(lag_column)`. Without stored lag: two date dimension joins, DATEDIFF calculation, CASE to exclude TBD placeholders. Fragile and complex.

### Default Dates for Unknown Milestones

Unresolved milestone dates must point to a special "Unknown / To Be Determined" surrogate row in the date dimension. Never null. Null FKs violate referential integrity and cause rows to disappear from joins.

---

## Data Governance and Conformed Dimensions

Defining conformed dimensions requires organisational consensus — agreement on attribute names, definitions, and domain values. This is the hardest part of enterprise DW work, and it is fundamentally a people problem, not a technical one.

**IT cannot drive this alone.** Business subject matter experts must lead the governance of shared dimensions. They have the authority to make decisions that stick.

**Conformed dimensions enable agile development:**
- Build a dimension once → reuse across every new fact table
- Time-to-market for new business process data sources shrinks as the dimension library grows
- Eventually, new ETL development focuses almost entirely on loading new fact tables, because the dimension tables are already on the shelf

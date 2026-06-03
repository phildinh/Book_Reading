# Chapter 1 — Data Warehousing, Business Intelligence & Dimensional Modelling Primer

## The Core Problem

Every business runs two completely different data systems simultaneously.

**Operational systems** — write-heavy, transaction-by-transaction, optimised for speed. Record one event at a time. Overwrite rather than accumulate history. Examples: ERP, CRM, LMS.

**Analytical systems** — read-heavy, query millions of rows at once, optimised for aggregation and pattern-finding. Nobody answers "what was our conversion rate last semester?" one record at a time.

> **The fundamental conflict:** the data structure that is great for writing transactions is terrible for analytical queries.

A normalised 3NF operational database makes your query engine do ten-way joins just to answer a simple business question. The optimiser chokes, the query is slow, and business users cannot understand the schema.

**Kimball's answer:** build a second, completely separate system — the DW/BI system — designed from scratch for reading and analysis.

---

## 3NF vs Dimensional Modelling

| | 3NF (Operational) | Dimensional (Analytical) |
|---|---|---|
| Purpose | Protect the data | Expose the data |
| Design goal | Eliminate redundancy | Enable fast, intuitive queries |
| Structure | Many normalised tables | Few wide denormalised tables |
| Joins needed | 10+ to answer simple questions | 1–2 per query |
| Updates | Designed for writes | Designed for reads |
| Who uses it | Application code | Business users and BI tools |

**One sentence:** 3NF models how data is stored. Dimensional models how people think.

---

## The Two Table Types

### Fact Table
- One row per business event
- Contains numeric measurements (the numbers you sum, average, count)
- Contains foreign keys pointing outward to dimension tables
- Insert only — rows are never changed after loading
- Sparse by design — a row only exists if the event occurred

### Dimension Table
- One row per "thing" (one product, one customer, one date)
- Wide and flat — many descriptive columns
- Contains text attributes for filtering and grouping
- Intentionally denormalised — hierarchies are flattened into one table
- What Power BI filter dropdowns and report labels come from

---

## Grain — The Most Important Concept in the Book

The grain declares: **what does exactly one row represent?**

- Declare the grain before touching dimensions or facts
- Every single row in the fact table must represent the same thing
- The most common design error in real projects is skipping or rushing the grain

**Examples of correctly declared grains:**
- One product scanned on one POS transaction
- One student enrolled in one course in one intake
- One order line item on one purchase order

**What breaks when grain is wrong:**
- Mixed granularity rows produce aggregations that are wrong in ways that are invisible
- You cannot add dimensions that don't make sense at the declared grain
- Queries return numbers that look correct but are silently wrong

---

## The Star Schema

A fact table at the centre surrounded by dimension tables. Every dimension connects via a foreign key → primary key relationship.

```
        [Date Dim]
            |
[Product Dim] — [Fact Table] — [Store Dim]
            |
       [Promotion Dim]
```

**Why not snowflake?** Normalising dimensions (splitting Brand into its own table, Category into its own table) adds joins, confuses BI tools, and saves negligible disk space since dimensions are tiny compared to fact tables. Keep hierarchies flat inside one dimension table.

---

## The Kimball Architecture — Four Components

```
[Source Systems] → [ETL System] → [Presentation Area] → [BI Applications]
                    (back room)      (front room)
```

**Source systems** — operational apps. Write-optimised. Not queryable for analytics. You have no control over their schema.

**ETL system (back room)** — extracts from source, cleanses, conforms, loads into the dimensional model. Business users never touch this. Off limits.

**Presentation area (front room)** — dimensional star schemas that analysts and BI tools query. Must be:
- Dimensional (star schema structure)
- Atomic (most granular level available — no pre-aggregated only)
- Business-process organised (one subject area per business activity)
- Conformed (shared dimensions across all subject areas)

**BI applications** — Power BI, ad hoc SQL, dashboards. Everything the business sees.

---

## Medallion Architecture Mapping

| Medallion Layer | Kimball Equivalent | What it does |
|---|---|---|
| Bronze | Raw staging | Exact copy of source system data. Untouched. Never queried by users. |
| Silver | ETL back room output | Cleansed and conformed. NOT 3NF — just clean. Data quality fixed. |
| Gold | Presentation area | Dimensional star schemas. What Power BI queries. |

> **Common misconception:** Silver is NOT 3NF. Silver's job is cleansing and conforming, not normalising. 3NF in Silver means you're building the Inmon hybrid architecture — double the transformation work for no analytical benefit.

---

## Three Alternative Architectures

**Independent data marts** — each department builds its own mart from the same sources. Fast to build. The problem: Marketing's "revenue" and Finance's "revenue" are defined differently. Numbers never reconcile. Kimball strongly opposes this.

**Inmon CIF (hub-and-spoke)** — normalised 3NF enterprise DW at the centre, departmental data marts off the side. Users cannot query the 3NF centre efficiently. Marts tend to be pre-aggregated and lose atomic detail.

**Hybrid** — normalised EDW used only as a staging/integration layer (not user-facing), with a full Kimball presentation area on top. Acceptable. Common in organisations with significant existing investment in a normalised repository.

---

## Five Myths — Know These for Design Reviews

1. **Dimensional models are only for summary data.** False — Kimball explicitly demands atomic grain. Summaries are supplements.
2. **They are departmental, not enterprise.** False — conformed dimensions make them the most integrative approach.
3. **They are not scalable.** False — star schemas are what every cloud data warehouse is optimised for.
4. **They can't handle ad hoc queries.** False — no built-in query bias means every dimension is an equal entry point.
5. **They can't be integrated.** False — this is exactly what conformed dimensions solve.

# ch10 — Physical Data Models (PDM)

## What problem does this solve and why does it exist

The LDM is designed for correctness. The PDM is designed for performance, storage efficiency, and the constraints of real hardware and software. You cannot skip the LDM and go straight to the PDM — you need to know what is correct before you decide where to make trade-offs.

The disk/RAM/CPU lens: the LDM is about what goes on disk correctly. The PDM is about how to structure what's on disk so the CPU reads the minimum amount of RAM to answer a query.

> The PDM is the logical data model compromised for specific software or hardware.

---

## CDM → LDM → PDM in one table

| Level | What it answers | Trade-offs |
|-------|----------------|-----------|
| CDM | What concepts exist and how do they relate? | None — scope only |
| LDM | What data does each concept need? (3NF) | None — business rules only |
| PDM | How do we store it efficiently for real queries? | Correctness traded for performance |

---

## Denormalization

> Denormalization is the process of selectively violating normalization rules and reintroducing redundancy into the model to reduce data retrieval time.

**Why it exists (disk/RAM/CPU framing):**

A fully normalised 3NF schema requires multiple joins to answer a question. Each join means the CPU reads additional pages from disk into RAM. For analytical queries that scan millions of rows, this is expensive. Denormalization pre-joins data so a single table scan answers the question — fewer disk reads, less RAM pressure, faster CPU execution.

**The trade-off:**
- Normalised → less storage, slower reads (more joins), easier updates
- Denormalised → more storage, faster reads (fewer joins), harder updates (same data in multiple places)

Choose denormalization when: read volume >> write volume (analytics, reporting). Keep normalization when: write consistency matters (OLTP, source systems).

---

## Two denormalization techniques

### Rolldown (most common)

The parent entity disappears. Its columns and relationships move into the child entity.

```
Before (normalised):
Category ──|──<── Offering_Category_Assignment ──>──|── Offering

After rolldown:
Offering_Assignment (contains category columns folded in)

SQL before:
SELECT o.offering_name, c.category_name
FROM offering o
JOIN offering_category_assignment a ON o.offering_id = a.offering_id
JOIN category c ON a.category_id = c.category_id;

SQL after rolldown:
SELECT offering_name, category_name_1, category_name_2, category_name_3
FROM offering_assignment;
```

**When to rolldown:**
- The parent is always queried alongside the child
- You want to reduce join complexity for developers
- Storage cost is acceptable

**When NOT to rolldown:**
- The parent has many children (data would repeat massively)
- The parent needs its own access path

### Rollup (repeating groups — reverses 1NF)

The same column group is repeated N times in the same table. This is the denormalization equivalent of 1NF violation — intentionally reintroduced for query simplicity.

```
Before (normalised):
StudentDiagnosis: student_id, diagnosis_code, diagnosis_order

After rollup:
Student: student_id, ..., diagnosis_1, diagnosis_2, diagnosis_3

SQL after rollup (simpler for fixed-set queries):
SELECT student_id, diagnosis_1, diagnosis_2, diagnosis_3
FROM student;
```

**When to rollup:**
- The count of repeats is small and bounded (never >3)
- The parent entity is queried far more than the child

**Danger of rollup:** If a student gets a 4th diagnosis, you have no place to put it. The column count is fixed. Only rollup when you are certain the number never exceeds N.

---

## Subtype resolution at the physical level

A logical subtype (supertype + subtypes) cannot exist directly in an RDBMS. It must be resolved into one of three physical patterns:

### Identity — keep tables separate (1:1 FK)

```sql
CREATE TABLE title (
    title_id    INT PRIMARY KEY,
    title_name  VARCHAR(200) NOT NULL,
    ...
);

CREATE TABLE electronic_title (
    title_id        INT PRIMARY KEY REFERENCES title(title_id),
    download_size   DECIMAL(10,2),  -- subtype-specific
    format_code     CHAR(10)
);

CREATE TABLE print_title (
    title_id        INT PRIMARY KEY REFERENCES title(title_id),
    weight_kg       DECIMAL(6,3),   -- subtype-specific
    page_count      INT
);
```

**Use identity when:** Subtypes have many unique attributes; subtype-specific rules need enforcement; read patterns differ per subtype.

### Rolldown — collapse supertype into each subtype

```sql
CREATE TABLE electronic_title (
    title_id    INT PRIMARY KEY,
    title_name  VARCHAR(200) NOT NULL,  -- from supertype
    download_size DECIMAL(10,2)
);

CREATE TABLE print_title (
    title_id    INT PRIMARY KEY,
    title_name  VARCHAR(200) NOT NULL,  -- duplicated from supertype
    weight_kg   DECIMAL(6,3)
);
```

**Use rolldown when:** Subtypes are mostly queried independently; concrete subtype names are more readable than the supertype; new subtypes are unlikely.

**Cost:** Shared supertype columns are duplicated across subtype tables.

### Rollup — collapse all subtypes into the supertype

```sql
CREATE TABLE title (
    title_id         INT PRIMARY KEY,
    title_type_code  CHAR(10) NOT NULL CHECK (title_type_code IN ('ELECTRONIC','PRINT')),
    title_name       VARCHAR(200) NOT NULL,
    download_size    DECIMAL(10,2) NULL,  -- only for ELECTRONIC, NULL for PRINT
    weight_kg        DECIMAL(6,3) NULL,   -- only for PRINT, NULL for ELECTRONIC
    page_count       INT NULL
);
```

**Use rollup when:** Subtypes share most attributes; new subtypes are expected; queries typically span both subtypes.

**Cost:** Nullable columns that are invalid for some subtypes. Subtype-specific business rules must be enforced in application code, not the database.

---

## Star schema — dimensional denormalization

> A star schema is when each dimension hierarchy is flattened into a single dimension table. The fact table is in the centre; each dimension relates to it at the lowest grain.

**Why it's not called denormalization:** You cannot denormalize something that was never normalised. Star schemas are built from dimensional LDMs, not relational LDMs. The equivalent action is called **flattening** or **collapsing**.

**Before (snowflake — dimension hierarchies kept normalised):**
```
fact_enrolment → dim_campus → dim_state → dim_country
fact_enrolment → dim_semester → dim_academic_year
```

**After (star schema — each hierarchy flattened into one table):**
```
fact_enrolment → dim_campus  (campus + state + country columns all in one table)
fact_enrolment → dim_date    (semester + year columns all in one table)
```

**SQL comparison:**

```sql
-- Snowflake (normalised dimensions) — 4 joins
SELECT
    ac.year_name,
    ds.state_name,
    SUM(fe.enrolment_count)
FROM fact_enrolment fe
JOIN dim_semester s ON fe.semester_sk = s.semester_sk
JOIN dim_academic_year ac ON s.academic_year_sk = ac.academic_year_sk
JOIN dim_campus c ON fe.campus_sk = c.campus_sk
JOIN dim_state ds ON c.state_sk = ds.state_sk
GROUP BY ac.year_name, ds.state_name;

-- Star schema (flattened dimensions) — 2 joins, same result
SELECT
    d.academic_year_name,
    c.state_name,
    SUM(fe.enrolment_count)
FROM fact_enrolment fe
JOIN dim_date d ON fe.date_sk = d.date_sk
JOIN dim_campus c ON fe.campus_sk = c.campus_sk
GROUP BY d.academic_year_name, c.state_name;
```

**The star schema is standard for Power BI and DirectLake in Microsoft Fabric.** Flattened dimensions reduce the number of relationships the VertiPaq engine must navigate, improving DAX query performance.

---

## Views

> A view is a virtual table. It is a SQL query stored in the database. The data is not stored separately — the query runs every time the view is accessed.

**Use views when:**
- You want to present a denormalised or simplified version of a normalised schema
- You need to restrict what columns a consumer can see (security)
- The query logic is complex and used frequently

**Do not use views when:**
- The result set is large and queried repeatedly — materialise it instead
- In Fabric Direct Lake mode — views break DirectLake, use Delta tables directly

```sql
-- Silver layer view: 3NF joins presented as a flat record
CREATE VIEW silver.v_student_enrolment AS
SELECT
    s.student_id,
    s.student_first_name,
    s.student_last_name,
    e.enrolment_date,
    e.enrolment_status_code,
    c.course_code,
    c.course_name,
    sem.semester_name,
    sem.academic_year
FROM silver.student s
JOIN silver.enrolment e ON s.student_id = e.student_id
JOIN silver.course c ON e.course_id = c.course_id
JOIN silver.semester sem ON e.semester_id = sem.semester_id;
```

---

## Indexing

> An index is a value and a pointer to instances of that value on disk. It reduces retrieval time by letting the database find rows without scanning the entire table.

**Index types from the key taxonomy:**
- Primary key → unique clustered index (anchor for physical storage order)
- Alternate key → unique non-clustered index (enforces uniqueness, enables fast lookup)
- Inversion entry → non-unique non-clustered index (fast lookup for frequently filtered columns)

**Decision rule for adding an index:**
```
Add an index when:
1. The column appears frequently in WHERE, JOIN ON, or ORDER BY
2. The column has high cardinality (many distinct values)
3. The table is read far more than written

Do NOT index:
1. Low-cardinality columns (e.g. status_code with 3 values — a full scan is faster)
2. Columns that are updated frequently (index maintenance overhead on writes)
3. Small tables (full scans are fast enough)
```

**Composite index column ordering:** Put the most-selective column (highest cardinality) first. If queries filter by `campus_code` and `semester_code`, and there are 5 campuses and 20 semesters, put `semester_code` first.

```sql
-- Index supporting: WHERE campus_code = ? AND semester_code = ?
CREATE INDEX ix_enrolment_semester_campus
ON fact_enrolment (semester_code, campus_code)
INCLUDE (enrolment_count);  -- covering index — avoids key lookup
```

---

## Partitioning

> Partitioning splits a table into two or more physical segments to improve query performance and manageability.

**Two types:**

| Type | What is split | Use case |
|------|-------------|---------|
| **Horizontal** | Rows split by a range or list | Ten years of orders → one partition per year |
| **Vertical** | Columns split between tables | Volatile columns (updated daily) separated from stable columns |

**Disk/RAM/CPU framing:** Partitioning means a query for "2024 data" only reads the 2024 partition off disk into RAM, not all ten years. The CPU processes 1/10 of the data. This is called **partition pruning**.

```sql
-- SQL Server / Fabric: horizontal partition by year
CREATE PARTITION FUNCTION pf_enrolment_year (INT)
AS RANGE RIGHT FOR VALUES (2020, 2021, 2022, 2023, 2024, 2025);

CREATE PARTITION SCHEME ps_enrolment_year
AS PARTITION pf_enrolment_year ALL TO ([PRIMARY]);

CREATE TABLE fact_enrolment (
    enrolment_sk    INT,
    academic_year   INT,
    ...
) ON ps_enrolment_year(academic_year);
```

**In Fabric Lakehouse (Delta):** Partitioning is defined at the Delta table level using `PARTITIONED BY`:

```sql
-- Fabric Lakehouse: partition fact table by load_date
CREATE TABLE gold.fact_enrolment
USING DELTA
PARTITIONED BY (load_date)
AS SELECT ...;
```

**Partition column selection rules:**
- High-cardinality but query-bounded (year, month, region)
- The column most commonly used in WHERE filters
- Not too fine-grained (daily partitions on a 10-year table = 3,650 files — too many)

---

## Common mistakes

**Mistake 1 — Denormalising the LDM (silver layer).**
The silver layer must stay 3NF. Denormalization happens in the gold layer only. If you denormalise silver, downstream consumers cannot reconstruct the original rules.

**Mistake 2 — Snowflake in Power BI.**
Power BI with a snowflake schema forces the engine to join across multiple dimension tables. Always flatten to star schema in your gold layer before connecting Power BI.

**Mistake 3 — Indexing low-cardinality columns.**
Indexing `enrolment_status_code` (3 values: Active/Withdrawn/Completed) is wasteful. The query optimiser will choose a full scan anyway. Index only high-cardinality columns.

**Mistake 4 — Rollup with no type discriminator.**
If you rollup subtypes, always add a `_type_code` column. Without it, queries cannot filter to a specific subtype.

**Mistake 5 — Too many partitions.**
Partitioning by day on a table with 5 years of data creates 1,825 partitions. File management overhead exceeds the query benefit. Partition by month or year instead.

**Mistake 6 — Using a view in DirectLake mode.**
Microsoft Fabric's DirectLake reads Delta tables directly from OneLake. SQL views are not Delta tables — they force the model into DirectQuery or Import mode. Build your gold layer as Delta tables, not views.

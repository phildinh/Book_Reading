# ch11, ch12, ch14 — Data Dictionary, Lineage, and Naming Standards

These three chapters from Section IV contain tool-specific material about ER/Studio. Below is the tool-agnostic conceptual content that applies directly to your stack.

---

## Ch11 — Data Dictionary: Domains and Reference Values

### The problem

When five engineers define "date" differently — some as `DATE`, some as `DATETIME`, some as `VARCHAR(10)` — every JOIN on a date column becomes a cast operation and a potential error. A data dictionary enforces consistency at source.

### Domains in practice (tool-agnostic)

A domain is a reusable attribute template. Define it once, apply it everywhere. If it changes, update one definition.

**In dbt:** Domains live in `sources.yml` or `models/schema.yml` as column-level test definitions that you reuse via macros or generic tests.

```yaml
# macros/generic_tests.sql — reusable domain test
{% test is_valid_enrolment_status(model, column_name) %}
  SELECT *
  FROM {{ model }}
  WHERE {{ column_name }} NOT IN ('Active', 'Withdrawn', 'Completed', 'Deferred')
    AND {{ column_name }} IS NOT NULL
{% endtest %}

# schema.yml — apply the domain test to any column
models:
  - name: fact_enrolment
    columns:
      - name: enrolment_status_code
        tests:
          - is_valid_enrolment_status
  - name: dim_student
    columns:
      - name: current_enrolment_status_code
        tests:
          - is_valid_enrolment_status
```

**In SQL Server / Fabric:** User-defined types let you create a named type and apply it to any column.

```sql
-- Define the domain once
CREATE TYPE dbo.EnrolmentStatus FROM VARCHAR(20) NOT NULL;

-- Apply it to any column
CREATE TABLE silver.enrolment (
    enrolment_status_code  dbo.EnrolmentStatus,
    ...
);
```

### Reference values (lookup tables)

Reference values are the allowed list for a domain. In practice, these are lookup/code tables.

```sql
-- Create the reference value table
CREATE TABLE silver.ref_enrolment_status (
    status_code   VARCHAR(20) PRIMARY KEY,
    status_name   VARCHAR(100) NOT NULL,
    is_active     BIT NOT NULL DEFAULT 1
);

INSERT INTO silver.ref_enrolment_status VALUES
    ('Active',    'Currently enrolled', 1),
    ('Withdrawn', 'Student withdrew',   0),
    ('Completed', 'Completed the unit', 0),
    ('Deferred',  'Deferred to next semester', 0);

-- Enforce referential integrity
ALTER TABLE silver.enrolment
ADD CONSTRAINT fk_enrolment_status
FOREIGN KEY (enrolment_status_code)
REFERENCES silver.ref_enrolment_status(status_code);
```

**Decision rule:** Use a lookup table (not a CHECK constraint) when:
- The list of valid values may grow over time
- The values have display names or descriptions
- You need to filter by `is_active` to manage deprecated codes

Use a CHECK constraint when: the list is fixed forever (e.g. gender biological binary, direction NSEW).

---

## Ch12 — Data Lineage: Source → Transform → Target

### The problem

When a metric is wrong, the question is always "where did this come from?" Without lineage documentation, you spend hours reverse-engineering transformation logic from code. With lineage, you trace the problem in minutes.

### The three-component model

Hoberman's lineage model has three components:

| Component | What it is | In your stack |
|-----------|-----------|--------------|
| **Source** | Where the data comes from | HubSpot API, Dynamics 365, Paradigm DB |
| **Rule** | The transformation applied | Rename, filter, join, derive |
| **Target** | Where the data lands | silver.student, gold.dim_student |

### Lineage in dbt

dbt generates lineage automatically from the `ref()` and `source()` functions. Every `{{ ref('model_name') }}` creates an edge in the DAG.

```sql
-- silver/stg_enrolment.sql
-- Source: paradigm.enrolment (raw)
-- Target: silver.stg_enrolment (cleaned)
-- Rules: standardise status codes, cast dates, deduplicate

WITH raw AS (
    SELECT * FROM {{ source('paradigm', 'enrolment') }}
),
standardised AS (
    SELECT
        enrolment_id,
        student_id,
        course_code,
        CAST(enrolment_date AS DATE) AS enrolment_date,
        UPPER(TRIM(status)) AS enrolment_status_code,  -- rule: standardise
        _loaded_at
    FROM raw
    WHERE enrolment_id IS NOT NULL               -- rule: remove nulls
      AND status NOT IN ('TEST','DUMMY')         -- rule: filter test records
),
deduped AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY enrolment_id
            ORDER BY _loaded_at DESC
        ) AS rn
    FROM standardised
)
SELECT * EXCLUDE rn FROM deduped WHERE rn = 1;   -- rule: keep latest
```

The dbt DAG gives you: `paradigm.enrolment` → `stg_enrolment` → `silver.enrolment` → `gold.fact_enrolment`.

### Column-level lineage

Document source-to-target column mappings in your schema.yml `description` field for audit trail:

```yaml
models:
  - name: stg_enrolment
    description: "Standardised enrolment records from Paradigm SIS"
    columns:
      - name: enrolment_status_code
        description: |
          Source: paradigm.enrolment.status
          Rule: UPPER(TRIM(status)) — normalised to uppercase, whitespace removed
          Valid values: Active, Withdrawn, Completed, Deferred
```

---

## Ch14 — Naming Standards: Class Words and Conventions

### The problem

`sales`, `Sales_Amount`, `SalesAmt`, `SALES_AMOUNT`, `amt_sales` — four engineers, four ways to name the same column. Cross-model JOINs fail silently; new engineers take days to understand the schema; dbt tests need `column_name` to match exactly.

### Class words — the mandatory suffix convention

A class word is the last term in an attribute name. It makes the data type, purpose, and domain obvious without reading documentation.

```
student_last_name        → VARCHAR, human-readable label
enrolment_date           → DATE, no time component
tuition_amount           → DECIMAL, currency
campus_code              → VARCHAR short, lookup value
unit_count               → INT, count
attendance_pct           → DECIMAL 0-100, percentage
is_international_flag    → BIT, boolean
enrolment_notes_text     → VARCHAR(MAX), free text
student_sk               → INT, surrogate key
student_id               → VARCHAR, natural business key
```

**Enforce these class words in your team's convention:**

| Suffix | Data type | Nullable rule |
|--------|-----------|--------------|
| `_sk` | INT / BIGINT | NOT NULL (surrogate keys never null) |
| `_id` | VARCHAR or INT | NOT NULL (natural keys never null) |
| `_code` | VARCHAR(10-20) | NOT NULL with FK or CHECK |
| `_name` | VARCHAR(100-255) | NOT NULL for display names |
| `_date` | DATE | Depends on optionality |
| `_datetime` | DATETIME2 | Depends on optionality |
| `_amount` | DECIMAL(15,2) | NOT NULL, DEFAULT 0 for financials |
| `_count` | INT | NOT NULL, DEFAULT 0 |
| `_pct` | DECIMAL(5,2) | nullable when not yet calculated |
| `_flag` / `is_` | BIT | NOT NULL, DEFAULT 0 |
| `_text` | NVARCHAR(MAX) | nullable |
| `_key` | VARCHAR | alternate/natural key |

### Naming convention rules

**Rule 1 — snake_case for everything in SQL/dbt.**
```
Good: student_first_name, enrolment_status_code, fact_enrolment
Bad:  StudentFirstName, EnrolmentStatusCode, FactEnrolment
```

**Rule 2 — Prefix tables by layer and type.**
```
bronze.raw_paradigm_enrolment      # raw, source-named
silver.stg_enrolment               # staged/cleaned
silver.enrolment                   # normalised source-of-truth
gold.dim_student                   # dimension
gold.fact_enrolment                # fact
gold.dim_date                      # standard date dimension
ref_enrolment_status               # reference/lookup table
```

**Rule 3 — Use full words, not abbreviations.**
```
Good: student_first_name, enrolment_date, campus_code
Bad:  stdt_frst_nm, enrl_dt, cmpus_cd
```

Exceptions: `sk` (surrogate key), `id` (identifier), `pct` (percentage), `amt` is acceptable but `amount` is preferred.

**Rule 4 — Boolean columns use `is_` prefix.**
```
Good: is_international, is_active, is_scholarship_recipient
Bad:  international_flag, active_yn, scholarship
```

**Rule 5 — Fact tables include grain in the name or documentation.**
```
fact_enrolment          → one row per student-unit-semester
fact_assessment_attempt → one row per student-unit-assessment-attempt
fact_daily_headcount    → one row per campus-date
```

**Rule 6 — Dimension tables prefix with `dim_`.**
```
dim_student, dim_course, dim_campus, dim_date, dim_staff
```

**Rule 7 — Avoid names that change meaning over time.**
```
Bad:  current_student (what does "current" mean when the table has history?)
Good: student_enrolment_status_code (explicit), with dim_student.is_current_record (SCD flag)
```

### Applying naming standards as a code review checklist

```
□ Every column ends in a class word
□ No abbreviations except approved list (sk, id, pct, is_)
□ Table prefixes match layer (raw_, stg_, dim_, fact_, ref_)
□ snake_case throughout — no camelCase, no PascalCase
□ Boolean columns use is_ prefix
□ Surrogate keys end in _sk, natural keys in _id or _key
□ No columns named with reserved words (date, name, value, type, key)
```

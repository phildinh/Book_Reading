# ch06 — Attributes, Keys, and Domains

## What problem does this solve and why does it exist

Once you know what entities exist, you need to know what facts about them to record, how to uniquely identify each instance, and what values are valid. Without this discipline you get: columns with ambiguous names, tables with no clear primary key, foreign keys that point to the wrong thing, and data quality issues that only surface months after go-live.

---

## Attributes

> An attribute is an elementary piece of information that **identifies**, **describes**, or **measures** instances of an entity.

These three purposes are a forcing function:

| Purpose | What it does | Examples |
|---------|-------------|---------|
| **Identifies** | Points to a specific instance | `student_id`, `enrolment_id`, `course_code` |
| **Describes** | Gives a property of the instance | `student_last_name`, `course_title`, `campus_name` |
| **Measures** | Quantifies something about the instance | `final_grade_score`, `credit_points`, `attendance_pct` |

If you cannot decide which category an attribute falls into, it is either two attributes combined (multi-valued — needs splitting) or it is derived (calculated from other attributes — should not be stored raw in the logical model).

---

## Key taxonomy — full hierarchy

```
Candidate Key
│   "Any attribute (or combination) that uniquely identifies an instance"
│   Properties: unique, mandatory, non-volatile, minimal
│
├── Primary Key     ← the chosen candidate key
│   Used in FKs, clustered index anchor, surrogate if needed
│
└── Alternate Key   ← other candidate keys; enforce as unique constraint
    e.g. student_number even when student_id (surrogate) is PK

Surrogate Key
│   System-generated integer, no business meaning
│   No embedded intelligence (not a date code, not a department prefix)
│   Used to avoid unstable natural keys as the PK
│   Always pair with an Alternate Key on the natural key

Foreign Key
│   Points from a child entity to a parent entity's primary key
│   Automatically created when you define a relationship
│   Propagates the parent's PK into the child

Inversion Entry (IE)
    Non-unique index hint — attributes queried frequently but not unique
    e.g. enrolment_status, created_date, campus_code
```

### Surrogate key decision rule

Use a surrogate key when:
- The natural key is composite (3+ attributes)
- The natural key can change (violates non-volatile requirement)
- The natural key contains sensitive data (SSN, passport number)
- You are integrating multiple source systems with conflicting natural keys

When you use a surrogate, always define an alternate key on the natural key to preserve uniqueness enforcement.

```sql
-- Good pattern: surrogate PK + alternate key on natural key
CREATE TABLE dim_student (
    student_sk      INT IDENTITY(1,1) PRIMARY KEY,   -- surrogate
    student_id      VARCHAR(20) NOT NULL,             -- natural key from source
    student_email   VARCHAR(255) NOT NULL,            -- another candidate key
    ...
    CONSTRAINT uq_student_id    UNIQUE (student_id),
    CONSTRAINT uq_student_email UNIQUE (student_email)
);
```

### Bad approach vs good approach

**Bad:** Using a composite natural key as your PK when it will be used as a FK everywhere.

```sql
-- Bad: FK in fact table is now three columns
CREATE TABLE fact_enrolment (
    student_first_name VARCHAR(100),
    student_last_name  VARCHAR(100),
    student_dob        DATE,          -- three-column FK
    ...
);
```

**Good:** Surrogate key reduces every FK to a single integer.

```sql
-- Good: FK is one column, joins are fast
CREATE TABLE fact_enrolment (
    student_sk    INT,               -- single-column FK to dim_student
    ...
    FOREIGN KEY (student_sk) REFERENCES dim_student(student_sk)
);
```

---

## Domains

> A domain is the complete set of valid values an attribute can hold. It is a reusable contract applied to one or more attributes.

Three domain types, each more specific than the last:

```
Format domain    →  the data type and length
                    e.g. VARCHAR(10), DECIMAL(15,2), DATE

List domain      →  a finite set of allowed values (refines Format)
                    e.g. enrolment_status IN ('Active','Withdrawn','Completed','Deferred')

Range domain     →  min and max bounds (refines Format)
                    e.g. grade_score BETWEEN 0 AND 100
```

### Why domains matter

**Data quality enforcement.** A domain defined as `DECIMAL(15,2)` for anything ending in `_amount` prevents string values from entering monetary columns.

**Consistency across attributes.** If `enrolment_date`, `assessment_date`, and `graduation_date` all use the same `Date` domain (no time component, not nullable), they will behave consistently in aggregations and comparisons.

**Reuse without re-definition.** Define `Amount` once (15-digit decimal, 2 decimal places, NOT NULL, ≥ 0). Apply it to `tuition_amount`, `scholarship_amount`, `refund_amount`. Change the domain definition once; all attributes update.

### Mapping domains to your stack

| Hoberman concept | dbt equivalent | Fabric/SQL Server equivalent |
|-----------------|----------------|------------------------------|
| Format domain | column `data_type` in schema.yml | `CREATE TABLE` data type |
| List domain | `accepted_values` test | `CHECK` constraint |
| Range domain | `dbt_utils.expression_is_true` | `CHECK` constraint |
| Domain reuse | `sources.yml` column definitions | User-defined type or domain |

```yaml
# dbt schema.yml — domains as tests
models:
  - name: fact_enrolment
    columns:
      - name: enrolment_status
        tests:
          - accepted_values:              # List domain
              values: ['Active', 'Withdrawn', 'Completed', 'Deferred']
      - name: final_grade_score
        tests:
          - dbt_utils.expression_is_true: # Range domain
              expression: "final_grade_score BETWEEN 0 AND 100"
```

---

## Class words — the naming convention that enforces domains

Hoberman defines a **class word** as the last term in an attribute name. Class words signal domain type and make names unambiguous.

| Class word | Meaning | Examples |
|-----------|---------|---------|
| `_id` / `_sk` | Identifier / surrogate key | `student_id`, `course_sk` |
| `_code` | Short coded value from a controlled list | `campus_code`, `status_code` |
| `_name` | Human-readable label | `student_last_name`, `course_name` |
| `_date` | Calendar date, no time | `enrolment_date`, `graduation_date` |
| `_datetime` | Date + time | `created_datetime`, `submitted_datetime` |
| `_amount` | Currency value, decimal | `tuition_amount`, `refund_amount` |
| `_count` | Integer count | `credit_point_count`, `unit_count` |
| `_pct` | Percentage (0-100 or 0-1, be consistent) | `attendance_pct`, `completion_pct` |
| `_flag` / `_indicator` | Boolean | `is_international`, `has_scholarship` |
| `_text` | Free-form long text | `notes_text`, `feedback_text` |
| `_key` | Natural business key | `student_number_key` |

**Rule:** Every attribute name must end in a class word. If it doesn't, the domain is unclear.

```
Bad:  student_phone     → what format? mobile? work? nullable?
Good: student_mobile_number_text  → free text, may include country code

Bad:  approved         → boolean? date? by whom?
Good: is_approved_flag          → boolean
      approved_date             → date the approval happened
      approved_by_staff_id      → who approved it
```

---

## Common mistakes

**Mistake 1 — Multi-valued attributes.**
`student_name` contains both first and last name. Violates 1NF. Split into `student_first_name` and `student_last_name`.

**Mistake 2 — Derived attributes in the logical model.**
`age` is derived from `date_of_birth`. Do not store it. Calculate it at query time (or materialise it in a gold-layer view for performance).

**Mistake 3 — Surrogate key without alternate key.**
You add `student_sk` as the PK but forget to add a unique constraint on `student_id`. Two rows for the same student can now exist with different surrogate keys. Data integrity violation.

**Mistake 4 — No class word.**
Column named `status` — is it an enrolment status? A payment status? What values are valid? Always use `enrolment_status_code` or similar.

**Mistake 5 — Redefining the same domain multiple times.**
`tuition_amount DECIMAL(10,2)` in one table, `scholarship_amount NUMERIC(15,4)` in another, `refund_amount FLOAT` in a third. Three different definitions for "money". Pick one domain and apply it consistently.

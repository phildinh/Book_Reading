# jmc-applied — Data Modeling for JMC Academy

**Scenario:** Higher education provider in Australia. Sources: HubSpot (CRM/marketing), Dynamics 365 (admissions/CRM), Paradigm (student information system / enrolments), Canvas (LMS / learning activity). Stack: Microsoft Fabric Lakehouse, dbt, Power BI. Goal: track the full student lifecycle from lead to graduation.

**This file applies every major concept from Hoberman to real tables, columns, decisions, and queries.**

---

## 1. Conceptual Data Model (Ch08)

### Step 1 — Five strategic questions answered

1. **What does this do?** Consolidate student lifecycle data from four source systems into a single analytics platform. Deliver Power BI reports for Admissions, Academic Quality, Student Services, and Finance.
2. **As-is or to-be?** Both. As-is for existing enrolment and academic data (3 years). To-be for lead-to-enrolment pipeline and completion tracking.
3. **Analytics a requirement?** Yes — dimensional model required for the gold layer. Relational model required for the silver layer.
4. **Audience?** Gold layer: VP Admissions, VP Academic, Finance Director (Power BI). Silver layer: Data engineers, dbt, downstream transformations.
5. **Flexibility or simplicity?** Simplicity. JMC is a single institution. Use specific terms (Student, Course, Enrolment) not generic (Party, Event).

### Conceptual entities — 6-category classification

```
Who:    Student, Staff, Agent (education agent), Employer (for industry placements)
What:   Course, Unit, Qualification, LMS Module, Assessment
When:   Semester, Academic Year, Intake Cohort
Where:  Campus, Delivery Mode (Online/On-campus/Blended), State/Territory
Why:    Application, Offer, Enrolment, Assessment Attempt, Withdrawal, Deferral, Completion
How:    Offer Letter, Enrolment Confirmation, Academic Transcript, Invoice, CoE (Certificate of Enrolment)
```

### Dimensional CDM — Grain Matrix

```
Business questions collected:

Q1: How many applications and offers by course, semester, campus, agent? [Admissions]
Q2: What is assessment pass rate by unit, campus, instructor, semester? [Academic Quality]
Q3: How many students enrolled, withdrew, deferred, completed by course and year? [Student Services]
Q4: What is average time from application to enrolment decision by intake? [Marketing]
Q5: What is tuition revenue by course, semester, campus, visa type? [Finance]

Grain Matrix:

                              app_count  offer_count  enrol_count  withdraw_count  pass_rate  days_to_decision  revenue_amount
                              (Q1)       (Q1)         (Q3)         (Q3)            (Q2)       (Q4)              (Q5)
Course                           ✓          ✓             ✓            ✓                           
Unit                                                                                  ✓            
Campus                           ✓          ✓             ✓            ✓             ✓                             ✓
Semester                         ✓          ✓             ✓            ✓             ✓             ✓               ✓
Academic Year                    ✓                        ✓            ✓                           ✓               ✓
Agent                            ✓          ✓                                                                       
Instructor                                                                            ✓            
Delivery Mode                    ✓          ✓             ✓                          ✓                             ✓
Visa Type / Student Type                                  ✓                                                         ✓
Intake Cohort                    ✓          ✓                                                       ✓              

Observation: Q1 and Q3 can share a single fact table if they use the same grain
(student-course-semester). Q2 needs a different grain (student-unit-assessment).
```

---

## 2. Medallion Architecture Mapping

```
Source Systems         Bronze              Silver (3NF)         Gold (Star)
──────────────         ──────────          ────────────         ───────────
HubSpot           →   raw_hubspot_*    →   stg_lead         →   dim_student
Dynamics 365      →   raw_dynamics_*   →   stg_application  →   dim_course
Paradigm SIS      →   raw_paradigm_*   →   enrolment        →   dim_campus
Canvas LMS        →   raw_canvas_*     →   assessment       →   dim_date
                                           student          →   fact_enrolment
                                           course           →   fact_application
                                           campus           →   fact_assessment
                                           semester         →   fact_revenue
```

---

## 3. Silver Layer — 3NF Schema

### Entity list and definitions (from the CDM)

| Entity | Category | Definition |
|--------|----------|-----------|
| `student` | Who | A person who has submitted at least one application to JMC. Includes applicants, enrolled, withdrawn, graduated, and deferred. |
| `course` | What | A structured program of study leading to a qualification. Identified by `course_code`. |
| `unit` | What | An individual subject within a course. Can appear in multiple courses. |
| `campus` | Where | A physical location or virtual delivery mode at which JMC delivers education. |
| `semester` | When | A defined study period within an academic year. |
| `application` | Why | The event of a student applying for enrolment in a course at JMC. |
| `offer` | Why | JMC's response to an application, granting or declining admission. |
| `enrolment` | Why | The event of a student formally enrolling in a unit within a course in a semester. |
| `assessment_attempt` | Why | A single student attempt at an assessment task within a unit. |
| `withdrawal` | Why | The event of a student ceasing enrolment in a unit or course. |

### Silver layer DDL

```sql
-- ─────────────────────────────────────────────────────────────
-- REFERENCE TABLES
-- ─────────────────────────────────────────────────────────────

CREATE TABLE silver.ref_enrolment_status (
    status_code   VARCHAR(20)  NOT NULL PRIMARY KEY,
    status_name   VARCHAR(100) NOT NULL,
    is_terminal   BIT          NOT NULL DEFAULT 0,  -- no further state changes expected
    is_active     BIT          NOT NULL DEFAULT 1
);

INSERT INTO silver.ref_enrolment_status VALUES
    ('Active',    'Currently enrolled',        0, 1),
    ('Withdrawn', 'Withdrawn by student',      1, 1),
    ('Deferred',  'Deferred to future term',   0, 1),
    ('Completed', 'Successfully completed',    1, 1),
    ('Excluded',  'Excluded by institution',   1, 1),
    ('LOA',       'Leave of absence',          0, 1);

CREATE TABLE silver.ref_delivery_mode (
    mode_code   CHAR(10)     NOT NULL PRIMARY KEY,
    mode_name   VARCHAR(50)  NOT NULL
);

INSERT INTO silver.ref_delivery_mode VALUES
    ('ONCAMPUS', 'On-campus'),
    ('ONLINE',   'Online'),
    ('BLENDED',  'Blended delivery');

CREATE TABLE silver.ref_visa_type (
    visa_code   VARCHAR(10)  NOT NULL PRIMARY KEY,
    visa_name   VARCHAR(100) NOT NULL,
    is_domestic BIT          NOT NULL DEFAULT 0
);

-- ─────────────────────────────────────────────────────────────
-- CORE ENTITIES — 3NF
-- ─────────────────────────────────────────────────────────────

CREATE TABLE silver.student (
    student_id              VARCHAR(20)  NOT NULL,  -- natural key from Paradigm SIS
    hubspot_contact_id      VARCHAR(50)  NULL,       -- FK to HubSpot (may not exist yet)
    dynamics_contact_id     VARCHAR(50)  NULL,       -- FK to Dynamics 365
    student_first_name      VARCHAR(100) NOT NULL,
    student_last_name       VARCHAR(100) NOT NULL,
    student_email           VARCHAR(255) NOT NULL,
    student_dob             DATE         NULL,
    visa_code               VARCHAR(10)  NOT NULL REFERENCES silver.ref_visa_type(visa_code),
    is_international_flag   BIT          NOT NULL DEFAULT 0,
    created_datetime        DATETIME2    NOT NULL,
    updated_datetime        DATETIME2    NOT NULL,
    _source_system          VARCHAR(20)  NOT NULL,
    _loaded_at              DATETIME2    NOT NULL,
    CONSTRAINT pk_student            PRIMARY KEY (student_id),
    CONSTRAINT uq_student_email      UNIQUE (student_email),
    CONSTRAINT uq_hubspot_contact    UNIQUE (hubspot_contact_id),
    CONSTRAINT uq_dynamics_contact   UNIQUE (dynamics_contact_id)
);

CREATE TABLE silver.course (
    course_id               INT          NOT NULL IDENTITY(1,1),
    course_code             VARCHAR(20)  NOT NULL,  -- natural key e.g. BSBA701
    course_name             VARCHAR(200) NOT NULL,
    qualification_level     VARCHAR(50)  NOT NULL,  -- Certificate III, Diploma, Bachelor, Master
    credit_point_count      INT          NOT NULL,
    duration_semester_count INT          NOT NULL,
    is_active_flag          BIT          NOT NULL DEFAULT 1,
    CONSTRAINT pk_course             PRIMARY KEY (course_id),
    CONSTRAINT uq_course_code        UNIQUE (course_code)
);

CREATE TABLE silver.unit (
    unit_id                 INT          NOT NULL IDENTITY(1,1),
    unit_code               VARCHAR(20)  NOT NULL,  -- e.g. BSBITA401
    unit_name               VARCHAR(200) NOT NULL,
    credit_point_count      INT          NOT NULL DEFAULT 0,
    is_active_flag          BIT          NOT NULL DEFAULT 1,
    CONSTRAINT pk_unit               PRIMARY KEY (unit_id),
    CONSTRAINT uq_unit_code          UNIQUE (unit_code)
);

-- Associative entity: resolves course-unit M:M
CREATE TABLE silver.course_unit (
    course_id               INT          NOT NULL REFERENCES silver.course(course_id),
    unit_id                 INT          NOT NULL REFERENCES silver.unit(unit_id),
    unit_sequence_number    INT          NOT NULL,  -- order within the course
    is_core_flag            BIT          NOT NULL DEFAULT 1,
    CONSTRAINT pk_course_unit        PRIMARY KEY (course_id, unit_id)
);

CREATE TABLE silver.campus (
    campus_id               INT          NOT NULL IDENTITY(1,1),
    campus_code             VARCHAR(10)  NOT NULL,
    campus_name             VARCHAR(100) NOT NULL,
    state_code              CHAR(3)      NOT NULL,  -- NSW, VIC, QLD, etc.
    is_active_flag          BIT          NOT NULL DEFAULT 1,
    CONSTRAINT pk_campus             PRIMARY KEY (campus_id),
    CONSTRAINT uq_campus_code        UNIQUE (campus_code)
);

CREATE TABLE silver.semester (
    semester_id             INT          NOT NULL IDENTITY(1,1),
    semester_code           VARCHAR(20)  NOT NULL,  -- e.g. 2024-S1
    semester_name           VARCHAR(100) NOT NULL,
    academic_year           INT          NOT NULL,
    semester_number         INT          NOT NULL,  -- 1 or 2
    start_date              DATE         NOT NULL,
    end_date                DATE         NOT NULL,
    CONSTRAINT pk_semester           PRIMARY KEY (semester_id),
    CONSTRAINT uq_semester_code      UNIQUE (semester_code)
);

-- ─────────────────────────────────────────────────────────────
-- EVENT ENTITIES (Why)
-- ─────────────────────────────────────────────────────────────

CREATE TABLE silver.application (
    application_id          VARCHAR(50)  NOT NULL,  -- natural key from HubSpot/Dynamics
    student_id              VARCHAR(20)  NOT NULL REFERENCES silver.student(student_id),
    course_id               INT          NOT NULL REFERENCES silver.course(course_id),
    campus_id               INT          NULL REFERENCES silver.campus(campus_id),
    semester_id             INT          NOT NULL REFERENCES silver.semester(semester_id),
    mode_code               CHAR(10)     NOT NULL REFERENCES silver.ref_delivery_mode(mode_code),
    agent_id                VARCHAR(50)  NULL,       -- FK to agent table (not shown)
    application_date        DATE         NOT NULL,
    application_status_code VARCHAR(20)  NOT NULL,
    source_system           VARCHAR(20)  NOT NULL,   -- 'HUBSPOT' or 'DYNAMICS'
    _loaded_at              DATETIME2    NOT NULL,
    CONSTRAINT pk_application        PRIMARY KEY (application_id)
);

CREATE TABLE silver.offer (
    offer_id                VARCHAR(50)  NOT NULL,
    application_id          VARCHAR(50)  NOT NULL REFERENCES silver.application(application_id),
    offer_date              DATE         NOT NULL,
    offer_status_code       VARCHAR(20)  NOT NULL,   -- Conditional, Unconditional, Declined
    offer_expiry_date       DATE         NOT NULL,
    accepted_date           DATE         NULL,        -- null until student accepts
    _loaded_at              DATETIME2    NOT NULL,
    CONSTRAINT pk_offer              PRIMARY KEY (offer_id)
);

-- Grain: one row per student-unit-semester
-- This is the atomic fact — the lowest grain available
CREATE TABLE silver.enrolment (
    enrolment_id            VARCHAR(50)  NOT NULL,   -- natural key from Paradigm
    student_id              VARCHAR(20)  NOT NULL REFERENCES silver.student(student_id),
    unit_id                 INT          NOT NULL REFERENCES silver.unit(unit_id),
    course_id               INT          NOT NULL REFERENCES silver.course(course_id),
    campus_id               INT          NOT NULL REFERENCES silver.campus(campus_id),
    semester_id             INT          NOT NULL REFERENCES silver.semester(semester_id),
    mode_code               CHAR(10)     NOT NULL REFERENCES silver.ref_delivery_mode(mode_code),
    enrolment_date          DATE         NOT NULL,
    enrolment_status_code   VARCHAR(20)  NOT NULL REFERENCES silver.ref_enrolment_status(status_code),
    withdrawal_date         DATE         NULL,
    completion_date         DATE         NULL,
    tuition_amount          DECIMAL(15,2) NOT NULL DEFAULT 0,
    _source_system          VARCHAR(20)  NOT NULL,
    _loaded_at              DATETIME2    NOT NULL,
    CONSTRAINT pk_enrolment          PRIMARY KEY (enrolment_id),
    CONSTRAINT uq_enrolment_grain    UNIQUE (student_id, unit_id, semester_id)
);

-- Grain: one row per student-unit-assessment-attempt
CREATE TABLE silver.assessment_attempt (
    attempt_id              VARCHAR(50)  NOT NULL,
    enrolment_id            VARCHAR(50)  NOT NULL REFERENCES silver.enrolment(enrolment_id),
    assessment_task_code    VARCHAR(50)  NOT NULL,
    attempt_number          INT          NOT NULL DEFAULT 1,
    submission_datetime     DATETIME2    NULL,
    grade_score             DECIMAL(5,2) NULL,
    grade_code              VARCHAR(10)  NULL,       -- HD, D, C, P, F, NC
    is_pass_flag            BIT          NULL,       -- null until marked
    _loaded_at              DATETIME2    NOT NULL,
    CONSTRAINT pk_assessment_attempt PRIMARY KEY (attempt_id)
);
```

---

## 4. Gold Layer — Star Schema DDL

### Dimension tables (flattened from silver layer hierarchies)

```sql
-- ─────────────────────────────────────────────────────────────
-- dim_student — SCD Type 2 (history-tracked)
-- ─────────────────────────────────────────────────────────────

CREATE TABLE gold.dim_student (
    student_sk              INT          NOT NULL IDENTITY(1,1),  -- surrogate key
    student_id              VARCHAR(20)  NOT NULL,                -- natural key (alternate key)
    student_first_name      VARCHAR(100) NOT NULL,
    student_last_name       VARCHAR(100) NOT NULL,
    student_email           VARCHAR(255) NOT NULL,
    visa_code               VARCHAR(10)  NOT NULL,
    visa_name               VARCHAR(100) NOT NULL,               -- denormalised from ref_visa_type
    is_international_flag   BIT          NOT NULL,
    student_type_name       VARCHAR(50)  NOT NULL,               -- 'Domestic' or 'International'
    -- SCD Type 2 admin columns
    effective_start_date    DATE         NOT NULL,
    effective_end_date      DATE         NOT NULL DEFAULT '9999-12-31',
    is_current_record       BIT          NOT NULL DEFAULT 1,
    CONSTRAINT pk_dim_student        PRIMARY KEY (student_sk),
    CONSTRAINT uq_dim_student_nk     UNIQUE (student_id, effective_start_date)
);

-- ─────────────────────────────────────────────────────────────
-- dim_course — SCD Type 1 (overwrite current)
-- ─────────────────────────────────────────────────────────────

CREATE TABLE gold.dim_course (
    course_sk               INT          NOT NULL IDENTITY(1,1),
    course_id               INT          NOT NULL,
    course_code             VARCHAR(20)  NOT NULL,
    course_name             VARCHAR(200) NOT NULL,
    qualification_level     VARCHAR(50)  NOT NULL,
    credit_point_count      INT          NOT NULL,
    duration_semester_count INT          NOT NULL,
    is_active_flag          BIT          NOT NULL,
    CONSTRAINT pk_dim_course         PRIMARY KEY (course_sk),
    CONSTRAINT uq_dim_course_nk      UNIQUE (course_id)
);

-- ─────────────────────────────────────────────────────────────
-- dim_unit
-- ─────────────────────────────────────────────────────────────

CREATE TABLE gold.dim_unit (
    unit_sk                 INT          NOT NULL IDENTITY(1,1),
    unit_id                 INT          NOT NULL,
    unit_code               VARCHAR(20)  NOT NULL,
    unit_name               VARCHAR(200) NOT NULL,
    credit_point_count      INT          NOT NULL,
    CONSTRAINT pk_dim_unit           PRIMARY KEY (unit_sk),
    CONSTRAINT uq_dim_unit_nk        UNIQUE (unit_id)
);

-- ─────────────────────────────────────────────────────────────
-- dim_campus — hierarchy FLATTENED (state + campus in one table)
-- ─────────────────────────────────────────────────────────────

CREATE TABLE gold.dim_campus (
    campus_sk               INT          NOT NULL IDENTITY(1,1),
    campus_id               INT          NOT NULL,
    campus_code             VARCHAR(10)  NOT NULL,
    campus_name             VARCHAR(100) NOT NULL,
    state_code              CHAR(3)      NOT NULL,
    state_name              VARCHAR(50)  NOT NULL,              -- denormalised — star schema
    country_name            VARCHAR(50)  NOT NULL DEFAULT 'Australia',
    delivery_mode_code      CHAR(10)     NOT NULL,
    delivery_mode_name      VARCHAR(50)  NOT NULL,
    CONSTRAINT pk_dim_campus         PRIMARY KEY (campus_sk),
    CONSTRAINT uq_dim_campus_nk      UNIQUE (campus_id)
);

-- ─────────────────────────────────────────────────────────────
-- dim_date — hierarchy FLATTENED (date + semester + year in one table)
-- ─────────────────────────────────────────────────────────────

CREATE TABLE gold.dim_date (
    date_sk                 INT          NOT NULL,              -- YYYYMMDD as integer
    full_date               DATE         NOT NULL,
    day_of_week_number      INT          NOT NULL,
    day_name                VARCHAR(10)  NOT NULL,
    day_of_month            INT          NOT NULL,
    month_number            INT          NOT NULL,
    month_name              VARCHAR(10)  NOT NULL,
    calendar_quarter        INT          NOT NULL,
    calendar_year           INT          NOT NULL,
    -- academic calendar (denormalised from silver.semester)
    semester_code           VARCHAR(20)  NULL,
    semester_name           VARCHAR(100) NULL,
    academic_year           INT          NULL,
    semester_number         INT          NULL,
    is_semester_start_flag  BIT          NOT NULL DEFAULT 0,
    is_semester_end_flag    BIT          NOT NULL DEFAULT 0,
    CONSTRAINT pk_dim_date           PRIMARY KEY (date_sk)
);
```

### Fact tables

```sql
-- ─────────────────────────────────────────────────────────────
-- fact_enrolment
-- Grain: one row per student-unit-semester
-- ─────────────────────────────────────────────────────────────

CREATE TABLE gold.fact_enrolment (
    enrolment_sk            INT          NOT NULL IDENTITY(1,1),
    -- Foreign keys (dimensions)
    student_sk              INT          NOT NULL REFERENCES gold.dim_student(student_sk),
    unit_sk                 INT          NOT NULL REFERENCES gold.dim_unit(unit_sk),
    course_sk               INT          NOT NULL REFERENCES gold.dim_course(course_sk),
    campus_sk               INT          NOT NULL REFERENCES gold.dim_campus(campus_sk),
    enrolment_date_sk       INT          NOT NULL REFERENCES gold.dim_date(date_sk),
    -- Degenerate dimensions (no separate dim table needed — single attribute)
    enrolment_status_code   VARCHAR(20)  NOT NULL,
    enrolment_id            VARCHAR(50)  NOT NULL,             -- degenerate: natural key for traceability
    -- Measures
    tuition_amount          DECIMAL(15,2) NOT NULL DEFAULT 0,
    credit_point_count      INT           NOT NULL DEFAULT 0,
    is_withdrawal_flag      BIT           NOT NULL DEFAULT 0,
    is_completion_flag      BIT           NOT NULL DEFAULT 0,
    -- Audit
    _loaded_at              DATETIME2     NOT NULL,
    CONSTRAINT pk_fact_enrolment     PRIMARY KEY (enrolment_sk),
    CONSTRAINT uq_fact_enrolment_grain UNIQUE (student_sk, unit_sk, enrolment_date_sk)
);

-- ─────────────────────────────────────────────────────────────
-- fact_application
-- Grain: one row per application
-- ─────────────────────────────────────────────────────────────

CREATE TABLE gold.fact_application (
    application_sk          INT          NOT NULL IDENTITY(1,1),
    student_sk              INT          NOT NULL REFERENCES gold.dim_student(student_sk),
    course_sk               INT          NOT NULL REFERENCES gold.dim_course(course_sk),
    campus_sk               INT          NULL REFERENCES gold.dim_campus(campus_sk),
    application_date_sk     INT          NOT NULL REFERENCES gold.dim_date(date_sk),
    offer_date_sk           INT          NULL REFERENCES gold.dim_date(date_sk),
    acceptance_date_sk      INT          NULL REFERENCES gold.dim_date(date_sk),
    -- Degenerate
    application_id          VARCHAR(50)  NOT NULL,
    application_status_code VARCHAR(20)  NOT NULL,
    source_system           VARCHAR(20)  NOT NULL,
    -- Measures
    days_to_offer_count     INT          NULL,   -- offer_date - application_date
    days_to_decision_count  INT          NULL,   -- acceptance_date - application_date
    is_offer_made_flag      BIT          NOT NULL DEFAULT 0,
    is_offer_accepted_flag  BIT          NOT NULL DEFAULT 0,
    _loaded_at              DATETIME2    NOT NULL,
    CONSTRAINT pk_fact_application   PRIMARY KEY (application_sk),
    CONSTRAINT uq_fact_application_nk UNIQUE (application_id)
);

-- ─────────────────────────────────────────────────────────────
-- fact_assessment_attempt
-- Grain: one row per student-unit-assessment-attempt
-- ─────────────────────────────────────────────────────────────

CREATE TABLE gold.fact_assessment_attempt (
    attempt_sk              INT          NOT NULL IDENTITY(1,1),
    student_sk              INT          NOT NULL REFERENCES gold.dim_student(student_sk),
    unit_sk                 INT          NOT NULL REFERENCES gold.dim_unit(unit_sk),
    course_sk               INT          NOT NULL REFERENCES gold.dim_course(course_sk),
    campus_sk               INT          NOT NULL REFERENCES gold.dim_campus(campus_sk),
    submission_date_sk      INT          NULL REFERENCES gold.dim_date(date_sk),
    -- Degenerate
    attempt_id              VARCHAR(50)  NOT NULL,
    assessment_task_code    VARCHAR(50)  NOT NULL,
    attempt_number          INT          NOT NULL,
    grade_code              VARCHAR(10)  NULL,
    -- Measures
    grade_score             DECIMAL(5,2) NULL,
    is_pass_flag            BIT          NULL,
    is_first_attempt_flag   BIT          NOT NULL DEFAULT 0,
    _loaded_at              DATETIME2    NOT NULL,
    CONSTRAINT pk_fact_assessment    PRIMARY KEY (attempt_sk),
    CONSTRAINT uq_fact_assessment_nk UNIQUE (attempt_id)
);
```

---

## 5. Key Modeling Decisions Explained

### Decision 1 — Why `enrolment_status_code` is a degenerate dimension in `fact_enrolment`

The status code has only 6 values. A separate `dim_enrolment_status` table would add a join for almost no benefit. Degenerate dimensions (single-attribute, no hierarchy, low cardinality) belong in the fact table directly.

However, the reference table still exists in silver (`ref_enrolment_status`) for data quality enforcement.

### Decision 2 — Why `dim_campus` includes `delivery_mode`

In JMC's model, a student's delivery mode (Online/On-campus/Blended) is determined per campus-enrolment combination. Since every enrolment joins to a campus, the delivery mode rolls up naturally into `dim_campus`. This avoids a separate join to a delivery mode dimension.

### Decision 3 — Why `dim_date` includes semester columns

The time hierarchy at JMC is: Date → Semester → Academic Year. Flattening semester into `dim_date` means reports that show enrolment by semester use the same date dimension as reports that show enrolment by calendar month. One JOIN, not two.

### Decision 4 — Why SCD Type 2 is only on `dim_student`

`dim_student` tracks changes to student attributes (visa type, email, name changes). These matter because a student may have been domestic when they enrolled in 2022 and international by 2024. Fact rows from 2022 should reflect the 2022 student record.

`dim_course`, `dim_unit`, `dim_campus` use SCD Type 1 (overwrite). Course name changes are rare and not historically significant for reporting.

### Decision 5 — Why `fact_enrolment` is at unit grain, not course grain

A student can withdraw from one unit while remaining enrolled in others. A student can fail one unit and repeat it. Analysing pass rates, retention, and completions requires unit-level grain. Course-level aggregations can always be derived by summing unit-level rows.

---

## 6. dbt Models

### Staging (silver layer)

```sql
-- models/silver/stg_enrolment.sql
WITH raw AS (
    SELECT * FROM {{ source('paradigm', 'enrolment') }}
),
cleaned AS (
    SELECT
        TRIM(enrolment_id)                          AS enrolment_id,
        TRIM(student_id)                            AS student_id,
        TRIM(unit_code)                             AS unit_code,
        TRIM(course_code)                           AS course_code,
        TRIM(campus_code)                           AS campus_code,
        TRIM(semester_code)                         AS semester_code,
        CAST(enrolment_date AS DATE)                AS enrolment_date,
        UPPER(TRIM(status))                         AS enrolment_status_code,
        TRY_CAST(withdrawal_date AS DATE)           AS withdrawal_date,
        TRY_CAST(completion_date AS DATE)           AS completion_date,
        ISNULL(TRY_CAST(tuition_amount AS DECIMAL(15,2)), 0) AS tuition_amount,
        GETUTCDATE()                                AS _loaded_at
    FROM raw
    WHERE enrolment_id IS NOT NULL
      AND student_id IS NOT NULL
),
deduped AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY enrolment_id
            ORDER BY _loaded_at DESC
        ) AS rn
    FROM cleaned
)
SELECT * EXCLUDE rn FROM deduped WHERE rn = 1
```

### Gold fact table (dbt)

```sql
-- models/gold/fact_enrolment.sql
WITH enrolment AS (
    SELECT * FROM {{ ref('stg_enrolment') }}
),
dim_student AS (
    SELECT student_sk, student_id
    FROM {{ ref('dim_student') }}
    WHERE is_current_record = 1
),
dim_unit AS (
    SELECT unit_sk, unit_code FROM {{ ref('dim_unit') }}
),
dim_course AS (
    SELECT course_sk, course_code FROM {{ ref('dim_course') }}
),
dim_campus AS (
    SELECT campus_sk, campus_code FROM {{ ref('dim_campus') }}
),
dim_date AS (
    SELECT date_sk, full_date FROM {{ ref('dim_date') }}
)
SELECT
    {{ dbt_utils.generate_surrogate_key(['e.enrolment_id']) }} AS enrolment_sk,
    s.student_sk,
    u.unit_sk,
    c.course_sk,
    ca.campus_sk,
    d.date_sk                                    AS enrolment_date_sk,
    e.enrolment_status_code,
    e.enrolment_id,
    e.tuition_amount,
    -- derive credit points from unit dimension
    u.credit_point_count,
    CASE WHEN e.enrolment_status_code = 'Withdrawn' THEN 1 ELSE 0 END AS is_withdrawal_flag,
    CASE WHEN e.enrolment_status_code = 'Completed' THEN 1 ELSE 0 END AS is_completion_flag,
    GETUTCDATE()                                 AS _loaded_at
FROM enrolment e
JOIN dim_student s  ON e.student_id   = s.student_id
JOIN dim_unit u     ON e.unit_code    = u.unit_code
JOIN dim_course c   ON e.course_code  = c.course_code
JOIN dim_campus ca  ON e.campus_code  = ca.campus_code
JOIN dim_date d     ON e.enrolment_date = d.full_date
```

---

## 7. Key Power BI / Semantic Model Queries

### Enrolment count by course and semester

```sql
SELECT
    dc.course_name,
    dd.semester_name,
    COUNT(*)                                    AS total_enrolments,
    SUM(fe.is_withdrawal_flag)                  AS withdrawals,
    SUM(fe.is_completion_flag)                  AS completions,
    SUM(fe.tuition_amount)                      AS total_tuition_amount
FROM gold.fact_enrolment fe
JOIN gold.dim_course dc     ON fe.course_sk = dc.course_sk
JOIN gold.dim_date dd       ON fe.enrolment_date_sk = dd.date_sk
WHERE dd.academic_year = 2024
GROUP BY dc.course_name, dd.semester_name
ORDER BY dc.course_name, dd.semester_name;
```

### Assessment pass rate by unit and campus

```sql
SELECT
    du.unit_code,
    du.unit_name,
    dca.campus_name,
    COUNT(*)                                    AS total_attempts,
    SUM(CAST(fa.is_pass_flag AS INT))           AS pass_count,
    CAST(SUM(CAST(fa.is_pass_flag AS INT)) AS FLOAT)
        / NULLIF(COUNT(*), 0) * 100             AS pass_rate_pct
FROM gold.fact_assessment_attempt fa
JOIN gold.dim_unit du       ON fa.unit_sk = du.unit_sk
JOIN gold.dim_campus dca    ON fa.campus_sk = dca.campus_sk
JOIN gold.dim_date dd       ON fa.submission_date_sk = dd.date_sk
WHERE fa.is_pass_flag IS NOT NULL              -- exclude unmarked attempts
  AND dd.academic_year = 2024
GROUP BY du.unit_code, du.unit_name, dca.campus_name
ORDER BY pass_rate_pct ASC;                   -- surface lowest pass rates first
```

### Average time from application to enrolment decision

```sql
SELECT
    dc.course_name,
    dd_app.semester_name                        AS application_semester,
    AVG(CAST(fa.days_to_decision_count AS FLOAT)) AS avg_days_to_decision,
    COUNT(*)                                    AS application_count,
    SUM(fa.is_offer_accepted_flag)              AS accepted_count
FROM gold.fact_application fa
JOIN gold.dim_course dc     ON fa.course_sk = dc.course_sk
JOIN gold.dim_date dd_app   ON fa.application_date_sk = dd_app.date_sk
WHERE fa.days_to_decision_count IS NOT NULL
  AND dd_app.academic_year >= 2022
GROUP BY dc.course_name, dd_app.semester_name
ORDER BY avg_days_to_decision DESC;
```

---

## 8. dbt Schema Tests (Domain Enforcement)

```yaml
models:
  - name: fact_enrolment
    description: "One row per student-unit-semester enrolment. Grain: enrolment_id."
    columns:
      - name: enrolment_sk
        tests: [not_null, unique]
      - name: student_sk
        tests: [not_null, relationships: {to: ref('dim_student'), field: student_sk}]
      - name: unit_sk
        tests: [not_null, relationships: {to: ref('dim_unit'), field: unit_sk}]
      - name: enrolment_status_code
        tests:
          - not_null
          - accepted_values:
              values: ['Active', 'Withdrawn', 'Deferred', 'Completed', 'Excluded', 'LOA']
      - name: tuition_amount
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: "tuition_amount >= 0"
      - name: is_withdrawal_flag
        tests:
          - not_null
          - accepted_values:
              values: [0, 1]
              quote: false

  - name: dim_student
    columns:
      - name: student_sk
        tests: [not_null, unique]
      - name: student_id
        tests:
          - not_null
          - unique:
              where: "is_current_record = 1"   -- unique among current records only
```

---

## 9. Naming conventions applied

All objects in this design follow the class word convention from ch14:

- `_sk` — surrogate keys (`student_sk`, `course_sk`)
- `_id` — natural business keys (`student_id`, `course_id`)
- `_code` — short controlled-list values (`enrolment_status_code`, `campus_code`)
- `_name` — display labels (`course_name`, `campus_name`)
- `_date` — DATE columns (`enrolment_date`, `graduation_date`)
- `_datetime` — DATETIME2 columns (`_loaded_at`, `created_datetime`)
- `_amount` — DECIMAL(15,2) financial values (`tuition_amount`, `refund_amount`)
- `_count` — INT counts (`credit_point_count`, `unit_count`)
- `_pct` — percentage values (`pass_rate_pct`, `attendance_pct`)
- `is_` prefix — BIT booleans (`is_international_flag`, `is_current_record`)

Prefixes by layer:
- `raw_` — bronze (source mirror)
- `stg_` — silver staging (cleaned, not yet joined)
- `silver.` — silver schema (3NF)
- `gold.` — gold schema (star schema)
- `ref_` — reference/lookup tables (any layer)

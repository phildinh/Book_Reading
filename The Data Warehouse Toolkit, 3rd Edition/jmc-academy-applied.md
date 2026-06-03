# JMC Academy — Applied Data Model

> Everything from Chapters 1–13 applied to one real project.  
> Stack: HubSpot · Dynamics 365 · Paradigm · Canvas · Microsoft Fabric · Power BI

---

## The Problem We Are Solving

JMC runs four operational systems: HubSpot (leads and marketing), Dynamics 365 (CRM and applications), Paradigm (student enrolments and academic records), and Canvas (learning management). Each is write-optimised and built for its own purpose. None of them can answer the CCO's question:

> *"Show me lead-to-enrolment conversion by course and campus, with revenue, for this intake versus last intake."*

That question spans all four systems. There is no single place where it can be answered. We are building that place — a dimensional model in Microsoft Fabric's Gold layer, queried by Power BI via Direct Lake.

---

## Architecture

```
HubSpot ──────┐
Dynamics 365 ─┤── [Bronze: raw data] ──► [Silver: cleansed & conformed] ──► [Gold: star schemas] ──► Power BI
Paradigm ─────┤
Canvas ────────┘
```

| Layer | Kimball Equivalent | What We Do |
|---|---|---|
| Bronze | Raw staging | Exact copy from each source. Untouched. Never queried. |
| Silver | ETL back room | Resolve 'JMCA' vs 'JMC Academy'. Match student IDs across systems. Standardise dates. Fix data quality. NOT 3NF. |
| Gold | Presentation area | Star schemas. Fact tables + dimension tables. What Power BI queries. |

---

## The JMC Bus Matrix

| Business Process | Source | Fact Type | Date | Student | Course | Campus | Intake | Instructor | Lead Source | Status |
|---|---|---|---|---|---|---|---|---|---|---|
| Lead capture | HubSpot | Transaction | ✓ | ✓ | ✓ | ✓ | | | ✓ | ✓ |
| Application pipeline | Dynamics | Accum. snapshot | ✓×6 | ✓ | ✓ | ✓ | ✓ | | | ✓ |
| Enrolment confirmed | Paradigm | Transaction | ✓ | ✓ | ✓ | ✓ | ✓ | | | |
| Student headcount | Paradigm | Periodic snapshot | Term | | ✓ | ✓ | ✓ | | | ✓ |
| Withdrawal/deferral | Paradigm | Transaction | ✓ | ✓ | ✓ | ✓ | ✓ | | | ✓ |
| Course registration | Canvas | Factless (activity) | Term | ✓ | ✓ | ✓ | ✓ | ✓ | | |
| Class attendance | Canvas | Factless (activity) | ✓ | ✓ | ✓ | | ✓ | ✓ | | |
| Attendance coverage | Canvas | Factless (coverage) | ✓ | ✓ | ✓ | | ✓ | ✓ | | |
| Assessment submission | Canvas | Transaction | ✓ | ✓ | ✓ | | ✓ | ✓ | | ✓ |
| Grade recorded | Canvas | Transaction | ✓ | ✓ | ✓ | | ✓ | ✓ | | ✓ |

Scan a column = one conformed dimension shared by all marked processes. Build it once. Reuse everywhere.

---

## The Gold Layer — Fact Tables

### 1. lead_fact (Transaction)

**Grain:** one row per lead event per student per course.

```
lead_fact:
  lead_event_date_key     FK → date_dim
  student_key             FK → student_dim
  course_key              FK → course_dim
  campus_key              FK → campus_dim
  lead_source_key         FK → lead_source_dim
  application_status_key  FK → status_dim
  lead_id                 Degenerate dim (HubSpot lead ID)
  lead_event_count = 1    Additive
```

---

### 2. application_pipeline_fact (Accumulating Snapshot)

**Grain:** one row per student application — inserted once, updated as milestones complete.

```
application_pipeline_fact:
  -- Six date FKs (six role-playing views of date_dim)
  initial_inquiry_date_key        FK → date_dim (role: Inquiry Date)
  campus_visit_date_key           FK → date_dim (role: Campus Visit Date)
  application_submitted_date_key  FK → date_dim (role: Submission Date)
  application_completed_date_key  FK → date_dim (role: Completion Date)
  decision_date_key               FK → date_dim (role: Decision Date)
  enrol_withdraw_date_key         FK → date_dim (role: Enrol/Withdraw Date)

  student_key             FK → student_dim
  course_key              FK → course_dim
  campus_key              FK → campus_dim
  intake_key              FK → intake_dim
  lead_source_key         FK → lead_source_dim
  application_status_key  FK → status_dim
  application_id          Degenerate dim (source system ID)

  -- Milestone counts (0 or 1 per row — additive)
  inquiry_count
  campus_visit_count
  application_submitted_count
  application_completed_count
  admit_count
  enrol_count
  withdraw_count

  -- Lag facts (calculated at update time, stored permanently)
  inquiry_to_submission_lag       days
  submission_to_decision_lag      days
  decision_to_enrolment_lag       days
  inquiry_to_enrolment_lag        days
```

**Unresolved dates** → surrogate key 0 (TBD row in date_dim). Never null.

**ETL pattern:** MERGE (upsert). Look up existing row by application_id. If found: update milestone columns. If not found: insert new row.

**What milestone counts enable:**
```sql
-- Funnel conversion rates
SELECT 
    c.course_name,
    SUM(enrol_count)                                  AS enrolled,
    SUM(inquiry_count)                                AS inquiries,
    ROUND(SUM(enrol_count) * 100.0 / SUM(inquiry_count), 1) AS conversion_pct
FROM application_pipeline_fact f
JOIN course_dim c ON c.course_key = f.course_key
JOIN intake_dim i ON i.intake_key = f.intake_key
WHERE i.intake_name = 'Semester 1 2024'
GROUP BY c.course_name
ORDER BY conversion_pct DESC;
```

---

### 3. enrolment_fact (Transaction)

**Grain:** one row per student per course per intake at point of confirmed enrolment.

```
enrolment_fact:
  enrolment_date_key      FK → date_dim
  student_key             FK → student_dim
  course_key              FK → course_dim
  campus_key              FK → campus_dim
  intake_key              FK → intake_dim
  enrolment_profile_key   FK → enrolment_profile_dim (junk)
  audit_key               FK → audit_dim
  enrolment_id            Degenerate dim (Paradigm ID)

  enrolment_fee           Additive fact
  scholarship_amount      Additive fact
  net_fee_payable         Additive fact
  credit_points           Additive fact
  study_load              Semi-additive (average across time, not sum)
```

**Enrolment profile junk dimension** — one FK instead of four separate flags:
```
enrolment_profile_dim:
  enrolment_profile_key (PK)
  delivery_mode           ('Online', 'On Campus', 'Blended')
  scholarship_indicator   ('Scholarship Student', 'Non-Scholarship')
  payment_plan            ('Upfront', 'Payment Plan', 'Deferred')
  student_type            ('Domestic', 'International')
```

---

### 4. student_headcount_snapshot (Periodic Snapshot)

**Grain:** one row per course per intake per term-end snapshot date.

```
student_headcount_snapshot:
  snapshot_date_key       FK → date_dim
  course_key              FK → course_dim
  campus_key              FK → campus_dim
  intake_key              FK → intake_dim
  enrolment_status_key    FK → status_dim

  student_count           Semi-additive (can SUM across courses, cannot SUM across snapshots)
  full_time_count         Semi-additive
  part_time_count         Semi-additive
  new_enrolment_count     Additive (change during period — fully additive)
  withdrawal_count        Additive (change during period — fully additive)
```

> **Power BI DAX for headcount over time:**  
> `AVERAGEX(ALL('Date'[Month]), [Student Count])` — NOT SUM

---

### 5. withdrawal_fact (Transaction)

**Grain:** one row per withdrawal or deferral event.

```
withdrawal_fact:
  withdrawal_date_key     FK → date_dim
  student_key             FK → student_dim
  course_key              FK → course_dim
  campus_key              FK → campus_dim
  intake_key              FK → intake_dim
  withdrawal_reason_key   FK → withdrawal_reason_dim
  withdrawal_id           Degenerate dim
  withdrawal_count = 1    Additive
```

---

### 6. course_registration_fact (Factless — Activity)

**Grain:** one row per student per course per term — at point of registration.

```
course_registration_fact:
  term_key                FK → term_dim
  student_key             FK → student_dim
  course_key              FK → course_dim
  campus_key              FK → campus_dim
  intake_key              FK → intake_dim
  instructor_group_key    FK → instructor_group_bridge (if co-taught) OR
  instructor_key          FK → instructor_dim (if single instructor)
  registration_count = 1
```

---

### 7. attendance_fact (Factless — Activity)

**Grain:** one row per student per class session actually attended.

```
attendance_fact:
  session_date_key        FK → date_dim
  student_key             FK → student_dim
  course_key              FK → course_dim
  intake_key              FK → intake_dim
  instructor_key          FK → instructor_dim
  facility_key            FK → facility_dim
  attendance_count = 1
```

---

### 8. attendance_coverage_fact (Factless — Coverage)

**Grain:** one row per enrolled student per scheduled session — regardless of attendance.

Same schema as attendance_fact. This table defines what SHOULD have happened.

**At-risk student query (enrolled but never attended):**
```sql
SELECT s.student_name, c.course_name
FROM attendance_coverage_fact cf
JOIN student_dim s ON s.student_key = cf.student_key
JOIN course_dim c ON c.course_key = cf.course_key
JOIN intake_dim i ON i.intake_key = cf.intake_key
WHERE i.intake_name = 'Semester 1 2024'
AND NOT EXISTS (
    SELECT 1 FROM attendance_fact af
    WHERE af.student_key = cf.student_key
    AND af.course_key = cf.course_key
    AND af.intake_key = cf.intake_key
)
```

---

### 9. assessment_fact (Transaction)

**Grain:** one row per assessment submission per student.

```
assessment_fact:
  submission_date_key     FK → date_dim
  student_key             FK → student_dim
  course_key              FK → course_dim
  intake_key              FK → intake_dim
  instructor_key          FK → instructor_dim
  assessment_type_key     FK → assessment_type_dim
  submission_status_key   FK → status_dim
  assessment_id           Degenerate dim
  submission_count = 1    Additive
```

---

### 10. grade_fact (Transaction)

**Grain:** one row per grade recorded per student per assessment.

```
grade_fact:
  grade_date_key          FK → date_dim
  student_key             FK → student_dim
  course_key              FK → course_dim
  intake_key              FK → intake_dim
  instructor_key          FK → instructor_dim
  grade_status_key        FK → status_dim
  grade_id                Degenerate dim
  grade_score             Non-additive (do not SUM — use AVG)
  grade_points            Additive (GPA points — can accumulate)
```

---

## The Conformed Dimensions

### student_dim

One row per student profile version (SCD Type 2 on key attributes).

```
student_dim:
  student_key (PK surrogate)
  student_natural_id (NK — durable across all Type 2 versions)
  student_name            SCD Type 1
  email                   SCD Type 1
  campus                  SCD Type 2
  declared_course         SCD Type 2
  class_level             SCD Type 2 ('Year 1', 'Year 2', 'Year 3')
  enrolment_status        SCD Type 2 ('Full-time', 'Part-time')
  postcode                SCD Type 2
  state                   SCD Type 2
  country
  gender
  date_of_birth_key       FK → date_dim (outrigger role)
  application_source      ('Agent', 'Direct', 'Social Media', 'Web')
  row_effective_date
  row_expiration_date
  current_row_indicator   ('Current', 'Expired')

  -- engagement_score → lives in engagement_profile_dim (SCD Type 4)
```

**Engagement mini-dimension (SCD Type 4):**
```
engagement_profile_dim:
  engagement_profile_key (PK)
  engagement_band         ('High', 'Medium', 'Low', 'Inactive')
  attendance_rate_band    ('90-100%', '70-89%', '50-69%', '<50%')
  lms_activity_band       ('Active', 'Moderate', 'Inactive')
```

---

### course_dim

```
course_dim:
  course_key (PK)
  course_natural_id (NK)
  course_name             SCD Type 1 (corrections)
  course_code
  faculty_name            Flattened hierarchy — no separate faculty table
  department_name         Flattened hierarchy
  delivery_mode           SCD Type 2 ('Online', 'On Campus', 'Blended')
  aqf_level               ('Certificate III', 'Diploma', 'Bachelor', etc.)
  credit_points
  duration_semesters
  cricos_flag             ('CRICOS Registered', 'Non-CRICOS')
  fee_structure           SCD Type 2 (fee changes must be tracked)
  row_effective_date
  row_expiration_date
  current_row_indicator
```

---

### date_dim

Pre-populated 10 years. One row per calendar day.

```
date_dim:
  date_key (PK — integer YYYYMMDD)
  full_date
  day_of_week             ('Monday', 'Tuesday', ...)
  day_number_in_month
  calendar_month          ('January', ...)
  calendar_month_number
  calendar_quarter        ('Q1', 'Q2', 'Q3', 'Q4')
  calendar_year
  fiscal_month
  fiscal_quarter
  fiscal_year
  weekday_indicator       ('Weekday', 'Weekend')
  public_holiday_flag     ('Public Holiday', 'Non-Holiday')
  -- JMC-specific:
  semester                ('Semester 1 2024', 'Semester 2 2024', ...)
  teaching_week_number    (1–18 within semester, null outside teaching period)
  census_date_flag        ('Census Date', 'Non-Census Date')
  orientation_week_flag   ('Orientation Week', 'Standard Week')
  exam_period_flag        ('Exam Period', 'Standard Week')
```

---

### term_dim

Conforms to date_dim. One row per academic term.

```
term_dim:
  term_key (PK)
  term_name               ('Semester 1 2024')
  academic_year           ('2023-2024')
  term_start_date_key     FK → date_dim
  term_end_date_key       FK → date_dim
  census_date_key         FK → date_dim
  orientation_start_key   FK → date_dim
  exam_period_start_key   FK → date_dim
```

---

### Other Conformed Dimensions

```
campus_dim:         campus_key, campus_name, city, state, country
intake_dim:         intake_key, intake_name ('Semester 1 2024'), intake_start_date_key, intake_end_date_key
lead_source_dim:    lead_source_key, source_name ('Agent', 'Direct', 'Social Media', 'Referral')
instructor_dim:     instructor_key, instructor_name, department, tenure_indicator, employment_type
facility_dim:       facility_key, building_name, room_name, facility_type, capacity, projector_flag
status_dim:         status_key, status_description, status_category (reused across processes)
withdrawal_reason_dim: reason_key, reason_description, reason_category
assessment_type_dim: type_key, type_name ('Assignment', 'Exam', 'Project', 'Practical')
```

---

### audit_dim

Tracks data quality and pipeline metadata for every fact row loaded.

```
audit_dim:
  audit_key (PK)
  quality_indicator       ('Clean', 'Warning', 'Anomaly')
  student_id_matched      ('Yes', 'No')
  fee_defaulted           ('Yes', 'No')
  lead_source_mapped      ('Yes', 'No')
  out_of_bounds_flag      ('Yes', 'No')
  source_system           ('HubSpot', 'Paradigm', 'Canvas', 'Dynamics')
  pipeline_version        ('v1.0', 'v2.0', ...)
  load_batch_id
```

**In Power BI:** CCO adds `quality_indicator` slicer → selects 'Clean' → every number on the dashboard comes only from fully validated rows.

---

## Bridge Table — Co-Taught Courses

If JMC courses are co-taught by multiple instructors:

```
instructor_group_bridge:
  instructor_group_key (FK)
  instructor_key (FK)
  weight              (0.5 for 2 equal instructors, 1.0 for solo)
```

- course_registration_fact carries `instructor_group_key` instead of `instructor_key`
- Weighted query: `SUM(enrolment_fee * b.weight)` → correct financial attribution per instructor
- Impact query: `SUM(enrolment_fee)` without weights → exposure measure (overcounting acknowledged)

---

## Role-Playing Date Dimension

The application_pipeline_fact has six date FK columns, all pointing to date_dim. In Fabric/SQL, create six views:

```sql
CREATE VIEW inquiry_date   AS SELECT date_key AS inquiry_date_key,   month AS inquiry_month,   year AS inquiry_year   FROM date_dim;
CREATE VIEW submission_date AS SELECT date_key AS submission_date_key, month AS submission_month, year AS submission_year FROM date_dim;
CREATE VIEW decision_date  AS SELECT date_key AS decision_date_key,  month AS decision_month,  year AS decision_year  FROM date_dim;
CREATE VIEW enrolment_date AS SELECT date_key AS enrolment_date_key, month AS enrolment_month, year AS enrolment_year FROM date_dim;
-- etc.
```

In Power BI: create one inactive relationship per date FK. Activate with `USERELATIONSHIP()` in DAX measures.

---

## Questions the CCO Can Now Answer

| Question | Tables Used | Pattern |
|---|---|---|
| Lead-to-enrolment conversion by course, this intake vs last | application_pipeline_fact | Milestone count facts |
| Average days from offer to enrolment by course | application_pipeline_fact | Lag fact: `AVG(decision_to_enrolment_lag)` |
| Revenue by course with high-quality data only | enrolment_fact + audit_dim | Filter `quality_indicator = 'Clean'` |
| Which courses are underenrolled vs same period last year | student_headcount_snapshot | Periodic snapshot comparison |
| Students enrolled but never attended a single class | coverage_fact minus attendance_fact | Set difference |
| Attendance rate by course and teaching week | attendance_fact + date_dim | `teaching_week_number` column |
| How many students changed from full-time to part-time, and what was their grade outcome | student_dim (Type 2) + grade_fact | SCD Type 2 history on study_load |
| Revenue attribution per instructor for co-taught courses | enrolment_fact + instructor_group_bridge | Weighted bridge table |
| Application funnel stage counts as at census date | application_pipeline_fact | Filter on `decision_date_key` context |
| Lead source effectiveness — which sources produce enrolled students | lead_fact + application_pipeline_fact + lead_source_dim | Drill-across on student_key and course_key |

---

## The One Rule That Ties Everything Together

Build the Student, Course, Campus, Date, Intake, and Term dimensions once. Define them in Silver. Load them into Gold. Every fact table points to the same rows.

Every question that crosses HubSpot, Paradigm, and Canvas data has a clean answer — because the vocabulary is shared. Without this, you have four separate reports from four systems that never reconcile. With this, you have one version of the truth that the entire business can trust.

This is the Kimball enterprise data warehouse bus architecture applied to JMC Academy.

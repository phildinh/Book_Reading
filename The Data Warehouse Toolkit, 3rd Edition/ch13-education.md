# Chapter 13 — Education: Accumulating Snapshots & Factless Fact Tables

## Why This Chapter

Chapter 13 applies the full Kimball toolkit to higher education. The two primary concepts — accumulating snapshot for pipeline tracking and factless fact tables for events and coverage — are illustrated through university-specific examples that map directly to any higher education institution.

---

## The University Bus Matrix

Higher education has an unusually broad set of business processes. The key ones and their fact table types:

| Business Process | Fact Table Type | Grain |
|---|---|---|
| Applicant tracking | Accumulating snapshot | One row per applicant |
| Financial aid awards | Transaction | One row per award |
| Student enrolment snapshot | Periodic snapshot | One row per student per term |
| Course registration | Factless (activity) | One row per student per course per term |
| Admissions event attendance | Factless (activity) | One row per event per attendee |
| Student attendance | Factless (activity) | One row per student per class session |
| Facility utilisation | Factless (coverage) | One row per facility per time block |
| Research grant pipeline | Accumulating snapshot | One row per grant proposal |
| Assessment submission | Transaction | One row per submission |
| Grade recorded | Transaction | One row per grade |

**Conformed dimensions shared across most rows:** Date/Term, Student/Applicant, Course, Department, Employee (Faculty/Staff), Campus, Facility.

---

## Applicant Pipeline — Accumulating Snapshot

### Design Rationale

The student admissions process is a classic pipeline with a definite beginning (initial enquiry), a definite end (enrolment or withdrawal), and standard intermediate milestones. An accumulating snapshot is the correct model.

### Schema

```
applicant_pipeline_fact:
  initial_inquiry_date_key          (FK → Date dim, role: Inquiry Date)
  campus_visit_date_key             (FK → Date dim, role: Campus Visit Date)
  application_submitted_date_key    (FK → Date dim, role: Submission Date)
  application_completed_date_key    (FK → Date dim, role: Completion Date)
  admission_decision_date_key       (FK → Date dim, role: Decision Date)
  enrol_withdraw_date_key           (FK → Date dim, role: Enrol/Withdraw Date)
  applicant_key                     (FK → Applicant dim)
  intended_course_key               (FK → Course dim)
  campus_key                        (FK → Campus dim)
  intake_key                        (FK → Intake dim)
  application_status_key            (FK → Status dim)
  application_id                    (Degenerate dim — source system ID)

  -- Milestone count facts (0 or 1 per row)
  inquiry_count
  campus_visit_count
  application_submitted_count
  application_completed_count
  admit_count
  enrol_count
  withdraw_count

  -- Lag facts (calculated at update time, stored permanently)
  inquiry_to_submission_lag         (days)
  submission_to_decision_lag        (days)
  decision_to_enrolment_lag         (days)
  inquiry_to_enrolment_lag          (days)
```

### How the Row Evolves

| Stage | Dates filled | Lag stored | Status |
|---|---|---|---|
| Application arrives | inquiry_date | — | Applied |
| Campus visit occurs | + campus_visit_date | — | Visited |
| Application submitted | + submitted_date | inquiry_to_submission_lag | Submitted |
| Decision made | + decision_date | submission_to_decision_lag | Offered |
| Enrolment confirmed | + enrol_date | decision_to_enrolment_lag, inquiry_to_enrolment_lag | Enrolled |

### Unresolved Dates

All date FKs for unresolved milestones point to a special "TBD" row in the Date dimension (surrogate key = 0). Never null — null FKs violate referential integrity.

### Milestone Counts as Facts

The 0/1 milestone count columns are additive:
- `SUM(enrol_count)` = total students enrolled
- `SUM(enrol_count) / SUM(inquiry_count)` = inquiry-to-enrolment conversion rate
- `SUM(enrol_count) / SUM(admit_count)` = offer acceptance rate

### Important Limitation

Accumulating snapshots do not preserve point-in-time pipeline counts. If you need to compare "pipeline as at census date vs pipeline as at today," add a companion periodic snapshot at key cut-off dates, or add effective/expiration timestamps to the accumulating snapshot rows.

---

## Applicant Dimension

The applicant dimension is rich with attributes used to slice the pipeline:

- Geographic: postcode, state, country
- Academic credentials: high school GPA, SAT/ACT scores, AP credits
- Demographic: gender, date of birth, ethnicity
- Application context: application source (agent, direct, social media, web), intended major, full-time/part-time indicator
- High school: school name, school type

These attributes allow admissions teams to ask: "Where are our application drop-offs highest — among domestic students applying via agents, or international students applying direct?"

---

## Factless Fact Tables in Higher Education

### 1 — Admissions Events (Activity)

Tracks attendance at admissions events: open days, information sessions, alumni interviews, campus overnights.

```
admissions_event_fact:
  event_date_key          (FK → Date dim)
  planned_enrol_term_key  (FK → Term dim)
  applicant_key           (FK → Applicant dim)
  applicant_status_key    (FK → Status dim)
  admissions_officer_key  (FK → Employee dim)
  admission_event_key     (FK → Event dim)
  attendance_count = 1
```

**Questions this answers:**
- Which marketing events generate the highest subsequent application rate?
- Which admissions officer conducted the most campus visits with eventual enrolled students?

### 2 — Course Registration (Activity)

One row per student per course per term at point of registration.

```
course_registration_fact:
  term_key          (FK → Term dim)
  student_key       (FK → Student dim)
  course_key        (FK → Course dim)
  instructor_key    (FK → Instructor dim)  ← or instructor_group_key if co-taught
  campus_key        (FK → Campus dim)
  registration_count = 1
```

**Questions this answers:**
- How many students registered for each course this semester?
- Which courses have the most students taking them as out-of-faculty electives?
- How many unique students has each instructor taught over three years?

**Co-taught courses:** if some courses have multiple instructors, replace `instructor_key` with `instructor_group_key` (FK to a bridge table). See Chapter 8.

### 3 — Student Attendance (Activity)

One row per student per class session attended.

```
attendance_fact:
  date_key          (FK → Date dim)
  student_key       (FK → Student dim)
  course_key        (FK → Course dim)
  instructor_key    (FK → Instructor dim)
  facility_key      (FK → Facility dim)
  attendance_count = 1
```

### 4 — Attendance Coverage (Coverage)

Every enrolled student × every scheduled session = one row. This table defines what SHOULD have happened.

```
attendance_coverage_fact:
  date_key          (FK → Date dim)
  student_key       (FK → Student dim)
  course_key        (FK → Course dim)
  instructor_key    (FK → Instructor dim)
  facility_key      (FK → Facility dim)
  coverage_count = 1
```

### Coverage + Activity = At-Risk Students

```sql
-- Students enrolled in a course but never attended ANY session
SELECT s.student_name, c.course_name
FROM attendance_coverage_fact cf
JOIN student_dim s ON s.student_key = cf.student_key
JOIN course_dim c ON c.course_key = cf.course_key
WHERE NOT EXISTS (
    SELECT 1
    FROM attendance_fact af
    WHERE af.student_key = cf.student_key
    AND af.course_key = cf.course_key
)
```

This is impossible from the attendance table alone. If a student never attended, they have zero rows in the activity table — they are simply absent. You can only identify absence by subtracting activity from coverage.

### 5 — Facility Utilisation (Coverage)

One row per facility per standard hourly time block per day per term — regardless of whether the facility is actually being used. The `utilisation_status` dimension carries 'Available' or 'Occupied'.

```
facility_utilisation_fact:
  term_key                (FK → Term dim)
  day_of_week_key         (FK → Day of Week dim)
  time_of_day_hour_key    (FK → Time dim)
  facility_key            (FK → Facility dim)
  owner_department_key    (FK → Department dim)
  assigned_department_key (FK → Department dim, role play)
  utilisation_status_key  (FK → Status dim)
  facility_count = 1
```

**Questions this answers:**
- Which facilities are used less than 50% of available hours?
- Does Friday afternoon utilisation drop significantly?
- Which department owns the most underutilised space?

---

## The Term Dimension

Education operates at term grain, not just calendar grain. Term dimension must CONFORM to the Date dimension:
- Every calendar date maps to exactly one term
- Term attributes (semester name, academic year) appear on the Date dimension as well
- Column labels and values must be identical in both — this is what enables drill-across between daily-grain facts (attendance) and term-grain facts (registration headcount)

```
term_dim:
  term_key (PK)
  term_name               ('Semester 1 2024')
  academic_year           ('2023-2024')
  term_start_date
  term_end_date
  census_date
  orientation_week_start
  exam_period_start
```

---

## Student Dimension — SCD Decisions for Education

The student dimension evolves as a student progresses through their studies. Different attributes warrant different change-handling strategies:

| Attribute | SCD Type | Reason |
|---|---|---|
| Student name, email | 1 | Corrections only |
| Campus | 2 | Must know campus when each assessment, grade, and enrolment occurred |
| Declared course / major | 2 | Course changes affect academic progress analysis |
| Class level (Year 1/2/3) | 2 | Determines curriculum context for each grade |
| Full-time / part-time status | 2 | Affects workload and progression analysis |
| Engagement score | 4 mini-dim | Changes weekly — would bloat student dim with Type 2 rows |

**Type 7 consideration:** if both "what was true when the fact occurred" AND "roll up all history by current student profile" are needed from the same dimension, use Type 7 (dual keys in fact table) or Type 6 (current attribute columns on Type 2 rows).

---

## Research Grant Proposal Pipeline

A second accumulating snapshot in education. Similar to the applicant pipeline but tracking grant proposals through funding milestones:

- Preliminary proposal submitted
- Sponsor review
- Full proposal submitted
- Award decision
- Award received
- Grant expenditure period

Dimensions: Date (×milestone roles), Faculty member, Department, Research topic, Funding source, Grant status.

Facts: proposed amount, awarded amount, preliminary-to-full-proposal lag, full-proposal-to-decision lag.

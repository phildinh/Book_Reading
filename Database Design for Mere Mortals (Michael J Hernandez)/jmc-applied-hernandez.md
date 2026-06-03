# jmc-applied-hernandez — Database Design for Mere Mortals Applied to JMC Academy

**Scenario:** Higher education provider in Australia. Sources: HubSpot (CRM/leads), Dynamics 365 (admissions CRM), Paradigm SIS (enrolments), Canvas LMS (learning/assessments). Target: Microsoft Fabric Lakehouse + dbt + Power BI. Goal: track the full student lifecycle from lead inquiry to graduation.

**This file applies every major concept from Hernandez's methodology to real tables, fields, keys, relationships, business rules, and views — enough detail that another engineer can implement from this file alone.**

---

## Phase 1 — Mission Statement & Objectives

### Mission Statement

> The purpose of the JMC Academy Student Platform database is to maintain the data we need to support student services from first lead inquiry through to graduation, including admissions, academic management, financial tracking, and compliance reporting.

### Mission Objectives (each = one task)

```
1.  Maintain complete lead and applicant information.
2.  Track all applications and their outcomes (offer/decline/withdraw).
3.  Maintain complete student information including visa and contact details.
4.  Maintain complete course and qualification information.
5.  Maintain complete unit information and course-unit associations.
6.  Track enrolments and their current status for each student-unit-semester combination.
7.  Record all assessment attempts and their outcomes.
8.  Maintain campus and delivery mode information.
9.  Record semester and academic year scheduling.
10. Maintain agent information for international student referrals.
11. Track student withdrawals, deferrals, and completions.
12. Produce information supporting admissions reporting by course, semester, agent.
13. Produce information supporting academic quality reporting by unit and campus.
14. Produce information supporting finance reporting by course and campus.
15. Support student services reporting across the full lifecycle.
```

---

## Phase 2 — Analyzed Sources

### Data collection samples gathered

| Source | Key forms/screens reviewed |
|--------|---------------------------|
| HubSpot | Lead capture form, contact record, deal pipeline |
| Dynamics 365 | Application form, offer letter, CoE fields |
| Paradigm SIS | Enrolment form, student record, semester schedule, unit record |
| Canvas | Assignment submission screen, grade entry screen |

### Preliminary Field List (representative subset)

```
From HubSpot:    lead_id, first_name, last_name, email, phone, date_of_birth,
                 course_interest, campus_preference, delivery_mode_preference,
                 lead_source, agent_code, created_date

From Dynamics:   application_id, application_date, application_status,
                 course_code, campus_code, delivery_mode, visa_type,
                 agent_id, offer_date, offer_type, offer_status,
                 offer_expiry_date, accepted_date

From Paradigm:   student_id, student_first_name, student_last_name,
                 student_email, student_dob, visa_code, is_international,
                 enrolment_id, unit_code, course_code, campus_code,
                 semester_code, enrolment_date, enrolment_status,
                 withdrawal_date, completion_date, tuition_amount,
                 course_name, unit_name, unit_credit_points,
                 qualification_level, semester_start_date, semester_end_date,
                 academic_year, campus_name, state_code,
                 agent_id, agent_name, agent_commission_rate

From Canvas:     attempt_id, student_id, assignment_id, assessment_task_code,
                 unit_code, submitted_at, grade_score, grade_code, is_pass,
                 attempt_number

Calculated Field List (removed from Preliminary):
  days_to_offer          = offer_date - application_date
  days_to_decision       = accepted_date - application_date
  pass_rate_pct          = (pass_count / attempt_count) × 100
  tuition_revenue        = SUM(tuition_amount)
  completion_rate_pct    = (completed_count / enrolled_count) × 100
  total_credit_points    = SUM(unit_credit_points per enrolment)
```

---

## Phase 3 — Table Structures

### Final Table List

| Table Name | Type | Description |
|-----------|------|-------------|
| `student` | Data | A person who has submitted at least one application to JMC. Includes applicants, enrolled, withdrawn, deferred, graduated. |
| `course` | Data | A structured program of study leading to a qualification. Identified by course_code (e.g., BSBA701). |
| `unit` | Data | An individual subject that forms part of one or more courses. Identified by unit_code (e.g., BSBITA401). |
| `course_unit` | Linking | Associates units with the courses they belong to, including sequence and whether core or elective. |
| `campus` | Data | A physical location or virtual delivery mode through which JMC delivers education. |
| `semester` | Data | A defined study period within an academic year. |
| `application` | Data | The event of a student formally applying for enrolment in a course at JMC. |
| `offer` | Data | JMC's formal response to an application — granted or declined. |
| `enrolment` | Data | The event of a student enrolling in a specific unit within a course in a semester. |
| `assessment_attempt` | Data | A single student attempt at an assessment task within a unit. |
| `agent` | Data | An education agent who refers international students to JMC. |
| `ref_enrolment_status` | Validation | Controlled list of valid enrolment status codes. |
| `ref_visa_type` | Validation | Controlled list of visa types with domestic/international flag. |
| `ref_delivery_mode` | Validation | Controlled list of delivery modes (On-campus, Online, Blended). |
| `ref_grade_code` | Validation | Controlled list of grade codes (HD, D, C, P, F, NC, etc.). |

### Subject-to-table cross-check via mission objectives

```
Objective 1 (leads/applicants):     → student, application
Objective 2 (application outcomes): → application, offer
Objective 3 (student info):         → student, ref_visa_type
Objective 4 (course info):          → course
Objective 5 (unit/course assoc):    → unit, course_unit
Objective 6 (enrolments):           → enrolment, ref_enrolment_status
Objective 7 (assessments):          → assessment_attempt, ref_grade_code
Objective 8 (campus/delivery):      → campus, ref_delivery_mode
Objective 9 (semester scheduling):  → semester
Objective 10 (agents):              → agent
Objective 11 (withdraw/defer):      → enrolment, ref_enrolment_status
Objectives 12-15 (reporting):       → addressed by views
```

### Table structures — field lists with ideal field compliance

**student** (primary: student_id — natural key from Paradigm, stable, unique, not null)
```
student_id              VARCHAR(20)   PK    Not null, no edits after creation
hubspot_contact_id      VARCHAR(50)   AK    Nullable (not all students have HubSpot record)
dynamics_contact_id     VARCHAR(50)   AK    Nullable
student_first_name      VARCHAR(100)        Not null
student_last_name       VARCHAR(100)        Not null
student_email           VARCHAR(255)  AK    Not null, unique
student_dob             DATE                Nullable (not always collected)
visa_code               VARCHAR(10)   FK    Not null → ref_visa_type.visa_code
is_international        BIT                 Not null, default 0
created_datetime        DATETIME2           Not null, system-entered
updated_datetime        DATETIME2           Not null, system-entered
_source_system          VARCHAR(20)         Not null (PARADIGM, HUBSPOT, DYNAMICS)

Fields verified against Elements of the Ideal Field:
  ✓ student_name NOT stored (multipart → split into first/last)
  ✓ student_address NOT stored here (multipart → belongs in a contact/address table if needed)
  ✓ age NOT stored (calculated from student_dob)
  ✓ student_email: single value, atomic, unique within database
  ✓ visa_code: single value (the primary visa — not "PR, Student" combined)
```

**course** (primary: course_code — natural key, stable, meaningful)
```
course_id               INT           PK    Auto-generated (simpler for FK use)
course_code             VARCHAR(20)   AK    Not null, unique (e.g., BSBA701)
course_name             VARCHAR(200)        Not null
qualification_level     VARCHAR(50)         Not null (Certificate III, Diploma, Bachelor, Master)
credit_point_count      INT                 Not null
duration_semester_count INT                 Not null
is_active               BIT                 Not null, default 1

Note: course_id as PK (surrogate) because course_code appears as FK in many tables;
      single integer FK is more efficient than VARCHAR(20). AK enforces uniqueness of code.
```

**unit** (primary: unit_id surrogate; AK on unit_code)
```
unit_id                 INT           PK    Auto-generated
unit_code               VARCHAR(20)   AK    Not null, unique (e.g., BSBITA401)
unit_name               VARCHAR(200)        Not null
credit_point_count      INT                 Not null, default 0
is_active               BIT                 Not null, default 1
```

**course_unit** (linking table — resolves course ↔ unit M:M)
```
course_id               INT           CPK/FK → course.course_id
unit_id                 INT           CPK/FK → unit.unit_id
unit_sequence_number    INT                 Not null (ordering within the course)
is_core                 BIT                 Not null, default 1

PK: (course_id, unit_id) — composite
```

**campus** (primary: campus_id surrogate; AK on campus_code)
```
campus_id               INT           PK    Auto-generated
campus_code             VARCHAR(10)   AK    Not null, unique
campus_name             VARCHAR(100)        Not null
state_code              CHAR(3)             Not null FK → ref_state (if needed)
country_name            VARCHAR(50)         Not null, default 'Australia'
is_active               BIT                 Not null, default 1
```

**semester** (primary: semester_id surrogate; AK on semester_code)
```
semester_id             INT           PK    Auto-generated
semester_code           VARCHAR(20)   AK    Not null, unique (e.g., 2024-S1)
semester_name           VARCHAR(100)        Not null
academic_year           INT                 Not null
semester_number         INT                 Not null (1 or 2)
start_date              DATE                Not null
end_date                DATE                Not null
```

**application** (primary: application_id — from HubSpot/Dynamics, stable)
```
application_id          VARCHAR(50)   PK
student_id              VARCHAR(20)   FK    Not null → student.student_id
course_id               INT           FK    Not null → course.course_id
campus_id               INT           FK    Nullable → campus.campus_id
semester_id             INT           FK    Not null → semester.semester_id
mode_code               CHAR(10)      FK    Not null → ref_delivery_mode.mode_code
agent_id                VARCHAR(50)   FK    Nullable → agent.agent_id
application_date        DATE                Not null
application_status_code VARCHAR(20)         Not null
source_system           VARCHAR(20)         Not null (HUBSPOT, DYNAMICS)
_loaded_at              DATETIME2           Not null, system
```

**enrolment** (primary: enrolment_id — from Paradigm SIS)
```
enrolment_id            VARCHAR(50)   PK
student_id              VARCHAR(20)   FK    Not null → student.student_id
unit_id                 INT           FK    Not null → unit.unit_id
course_id               INT           FK    Not null → course.course_id
campus_id               INT           FK    Not null → campus.campus_id
semester_id             INT           FK    Not null → semester.semester_id
mode_code               CHAR(10)      FK    Not null → ref_delivery_mode.mode_code
enrolment_date          DATE                Not null
enrolment_status_code   VARCHAR(20)   FK    Not null → ref_enrolment_status.status_code
withdrawal_date         DATE                Nullable
completion_date         DATE                Nullable
tuition_amount          DECIMAL(15,2)       Not null, default 0
_source_system          VARCHAR(20)         Not null
_loaded_at              DATETIME2           Not null, system

AK: (student_id, unit_id, semester_id) — grain constraint: one row per student-unit-semester

PK test — does enrolment_id exclusively identify the value of each field?
  enrolment_id → student_id?           YES ✓
  enrolment_id → unit_id?              YES ✓
  enrolment_id → enrolment_status_code? YES ✓
  enrolment_id → tuition_amount?        YES ✓
  enrolment_id → student_email?         NO ✗  (identified by student_id, belongs in student)
  enrolment_id → unit_name?             NO ✗  (identified by unit_id, belongs in unit)
→ student_email and unit_name do NOT appear in enrolment — they belong in their parent tables.
```

**assessment_attempt** (primary: attempt_id — from Canvas LMS)
```
attempt_id              VARCHAR(50)   PK
enrolment_id            VARCHAR(50)   FK    Not null → enrolment.enrolment_id
assessment_task_code    VARCHAR(50)         Not null
attempt_number          INT                 Not null, default 1
submission_datetime     DATETIME2           Nullable (null until submitted)
grade_score             DECIMAL(5,2)        Nullable (null until marked)
grade_code              VARCHAR(10)   FK    Nullable → ref_grade_code.grade_code
is_pass                 BIT                 Nullable (null until marked)
_loaded_at              DATETIME2           Not null, system
```

**agent** (primary: agent_id — from HubSpot/Dynamics)
```
agent_id                VARCHAR(50)   PK
agent_name              VARCHAR(200)        Not null
agent_email             VARCHAR(255)        Not null
agent_country_code      CHAR(3)             Nullable
commission_rate         DECIMAL(5,4)        Nullable
is_active               BIT                 Not null, default 1
```

---

## Phase 4 — Table Relationships

### Table matrix (key relationships identified)

```
                 student  course  unit  campus  semester  enrolment  application  assessment
student                                                     1:N          1:N
course                            1:N                       1:N          1:N
unit                    1:N                                 1:N
campus                                                      1:N          1:N
semester                                                    1:N          1:N
enrolment        N:1     N:1     N:1    N:1      N:1                               1:N
application      N:1     N:1                     N:1
assessment                                                  N:1

Many-to-many:
  course ↔ unit → resolved by course_unit linking table (already in Phase 3)
```

### Relationship diagrams — key characteristics

```
student ——|——<○—— enrolment
  Deletion rule:  Restrict (cannot delete student with any enrolment record)
  student:        Mandatory | (1,1) — each enrolment belongs to exactly one student
  enrolment:      Optional  | (0,N) — a student may have no enrolments yet (applicant only)

course ——|——<○—— enrolment
  Deletion rule:  Restrict
  course:         Mandatory | (1,1) — each enrolment is for one course
  enrolment:      Optional  | (0,N)

unit ——|——<○—— enrolment
  Deletion rule:  Restrict
  unit:           Mandatory | (1,1) — each enrolment covers one unit
  enrolment:      Optional  | (0,N)

enrolment ——|——<○—— assessment_attempt
  Deletion rule:  Cascade (if enrolment is deleted, assessment records are meaningless)
  enrolment:      Mandatory | (1,1) — each attempt belongs to one enrolment
  assessment_attempt: Optional | (0,N) — an enrolment may have no attempts yet

ref_enrolment_status ——|——<—— enrolment
  Deletion rule:  Restrict (cannot delete a status code that is in use)
  ref_enrolment_status: Mandatory | (1,1)
  enrolment:      Mandatory | (0,N) — enrolment must have a valid status

course ——<—— course_unit ——>—— unit
  (M:M resolved by linking table — see Ch10 pattern)
  course side: Mandatory | (1,N) — a course must have at least one unit
  unit side:   Mandatory | (1,N) — a unit must belong to at least one course
  Both sides:  Restrict deletion rule

student ——|——<○—— application
  Deletion rule:  Restrict
  student:        Mandatory | (1,1)
  application:    Optional  | (0,N)
```

### Foreign key specifications (sample — enrolment.student_id)

```
Field Name:        student_id
Parent Table:      enrolment
Spec Type:         Replica
Source Spec:       student_id from the student table
Description:       The identifier of the student enrolled in this unit. Enables tracking
                   of all enrolments for a given student across their time at JMC.
Key Type:          Foreign
Key Structure:     Simple
Uniqueness:        Non-Unique (one student → many enrolments)
Null Support:      No Nulls
Values Entered By: System (migrated from Paradigm)
Required Value:    Yes
Range of Values:   Any existing student_id in the student table
Edit Rule:         Enter Now, Edits Not Allowed
```

---

## Phase 5 — Business Rules

### Business Rule Specifications

**Rule BR-001: Enrolment status must be from the approved status list**
```
Statement:   Only approved enrolment status codes may be used.
Constraint:  enrolment.enrolment_status_code values limited to codes in ref_enrolment_status.
Type:        Database-oriented
Category:    Field-specific
Test on:     Insert, Update
Action:      FK relationship established: ref_enrolment_status ——|——<—— enrolment
             ref_enrolment_status.status_code is Mandatory, degree (1,1)
             enrolment.enrolment_status_code Range of Values:
               "Any status_code in ref_enrolment_status"
```

**Rule BR-002: Tuition amount cannot be negative**
```
Statement:   Tuition amounts recorded against enrolments must be zero or positive.
Constraint:  enrolment.tuition_amount >= 0.00
Type:        Database-oriented
Category:    Field-specific
Test on:     Insert, Update
Action:      Range of Values: >= 0.00
```

**Rule BR-003: A student can only have one enrolment per unit per semester**
```
Statement:   A student cannot be enrolled in the same unit twice in the same semester.
Constraint:  Unique constraint on (student_id, unit_id, semester_id) in enrolment table.
Type:        Database-oriented
Category:    Field-specific (uniqueness)
Test on:     Insert
Action:      AK: (student_id, unit_id, semester_id) on enrolment — enforced as unique constraint
```

**Rule BR-004: Grade score must be between 0 and 100**
```
Statement:   Grade scores for assessment attempts must be between 0 and 100 inclusive.
Constraint:  assessment_attempt.grade_score BETWEEN 0 AND 100
Type:        Database-oriented
Category:    Field-specific
Test on:     Insert, Update
Action:      Range of Values: 0.00 to 100.00
```

**Rule BR-005: An application must reference an active course**
```
Statement:   Students may only apply for courses that are currently active.
Constraint:  application.course_id values limited to courses where is_active = 1.
Type:        Database-oriented (enforced via validation view)
Category:    Field-specific
Test on:     Insert
Action:      Create validation view ACTIVE_COURSES over course where is_active = 1.
             application.course_id Range of Values:
               "Any course_id in ACTIVE_COURSES"
```

**Rule BR-006: A course must have at least one unit**
```
Statement:   Every course offered at JMC must be associated with at least one unit.
Constraint:  Single record in course cannot exist without at least one record in course_unit.
Type:        Database-oriented
Category:    Relationship-specific
Test on:     Delete (cannot remove last unit from a course)
Action:      Degree of participation for course in course ——<—— course_unit:
               Change from (0,N) to (1,N) — at least one unit required
```

**Rule BR-007: Agent may refer unlimited students**
```
Statement:   There is no limit on the number of applications an agent may refer.
Constraint:  application.agent_id degree of participation: (0,N).
Type:        Database-oriented
Category:    Relationship-specific
Test on:     Insert
Action:      Degree of participation confirmed as (0,N) for applications side.
             (Recorded explicitly because business originally proposed a cap.)
```

---

## Phase 6 — Views

### View 1 — STUDENT_ENROLMENT_DETAIL (Data view)
```
Base tables: student, enrolment, unit, course, campus, semester, ref_enrolment_status
Fields:
  student_id, student_first_name, student_last_name, student_email,
  is_international,
  enrolment_id, enrolment_date, enrolment_status_code, status_name,
  course_code, course_name, qualification_level,
  unit_code, unit_name, credit_point_count,
  campus_code, campus_name,
  semester_code, academic_year, semester_number,
  tuition_amount, withdrawal_date, completion_date
Calculated fields: none
Filters: none (returns all enrolments; filter by status/year in application)
Purpose: Core operational view for Student Services. Full lifecycle context.
```

### View 2 — APPLICATION_FUNNEL (Aggregate view)
```
Base tables: application, student, course
Fields:
  course_name (grouping)
  semester_code (grouping from semester join)
  academic_year (grouping)
  total_applications     = Count(application_id)
  total_offers           = Sum(CASE WHEN application_status_code = 'Offered' THEN 1 ELSE 0 END)
  total_accepted         = Sum(CASE WHEN application_status_code = 'Accepted' THEN 1 ELSE 0 END)
  avg_days_to_decision   = Avg(days_to_decision)  [calculated field]
Calculated fields: days_to_decision = accepted_date - application_date
Purpose: Admissions conversion tracking. Enrolment funnel by course and semester.
```

### View 3 — UNIT_PASS_RATE (Aggregate view)
```
Base tables: assessment_attempt, enrolment, unit, campus, semester
Fields:
  unit_code (grouping)
  unit_name (grouping)
  campus_name (grouping)
  semester_code (grouping)
  academic_year (grouping)
  total_attempts    = Count(attempt_id)
  pass_count        = Sum(CASE WHEN is_pass = 1 THEN 1 ELSE 0 END)
  pass_rate_pct     = (pass_count / NULLIF(total_attempts, 0)) × 100
Filter: is_pass IS NOT NULL (exclude unmarked attempts)
Purpose: Academic quality reporting. Identifies units and campuses with low pass rates.
```

### View 4 — ACTIVE_COURSES (Validation view)
```
Base table: course
Fields: course_id, course_code, course_name, qualification_level
Filter: is_active = 1
Purpose: Provides the valid list of course options for applications and enrolments.
         Prevents applications to discontinued courses.
```

### View 5 — STUDENT_LIFECYCLE_SUMMARY (Aggregate view)
```
Base tables: student, application, enrolment
Fields:
  student_id (grouping)
  student_first_name, student_last_name (grouping)
  total_applications    = Count(application_id)
  total_enrolments      = Count(enrolment_id)
  active_enrolments     = Sum(CASE WHEN enrolment_status_code = 'Active' THEN 1 ELSE 0 END)
  completed_units       = Sum(CASE WHEN enrolment_status_code = 'Completed' THEN 1 ELSE 0 END)
  withdrawn_units       = Sum(CASE WHEN enrolment_status_code = 'Withdrawn' THEN 1 ELSE 0 END)
  total_credit_points   = Sum(unit_credit_points from enrolment → unit join)
Purpose: Student Services overview. One row per student showing progression.
```

---

## Phase 7 — Data Integrity Review Summary

### Table-level integrity — confirmed

| Table | PK | Duplicate records possible? | Calculated fields? | Multipart? | Multivalued? |
|-------|----|-----------------------------|-------------------|-----------|-------------|
| student | student_id | No | No | No | No |
| course | course_id | No | No | No | No |
| unit | unit_id | No | No | No | No |
| course_unit | (course_id, unit_id) | No | No | No | No |
| enrolment | enrolment_id | No | No | No | No |
| assessment_attempt | attempt_id | No | No | No | No |
| application | application_id | No | No | No | No |

### Field-level integrity — confirmed

All fields comply with Elements of the Ideal Field. Key confirmations:
- `student_name` NOT stored (multipart → first + last)
- `tuition_revenue` NOT stored (calculated → view)
- `pass_rate_pct` NOT stored (calculated → view)
- `enrolment_status_code` has complete field spec with Replica type from ref table PK
- All FK fields have Replica specs referencing their source PK

### Relationship-level integrity — confirmed

All FKs share the same name as their source PK. Deletion rules documented for all relationships. Participation types and degrees explicitly set for all relationships.

### Business rules — confirmed

Seven rules documented on Business Rule Specifications sheets. Five database-oriented rules established. Two validation views created. Rule BR-003 enforced via AK constraint.

---

## Implementation DDL (Silver Layer — 3NF)

```sql
-- Validation tables first
CREATE TABLE silver.ref_enrolment_status (
    status_code   VARCHAR(20)   NOT NULL PRIMARY KEY,
    status_name   VARCHAR(100)  NOT NULL,
    is_terminal   BIT           NOT NULL DEFAULT 0,
    is_active     BIT           NOT NULL DEFAULT 1
);

INSERT INTO silver.ref_enrolment_status VALUES
    ('Active',    'Currently enrolled',         0, 1),
    ('Withdrawn', 'Withdrawn by student',        1, 1),
    ('Deferred',  'Deferred to future semester', 0, 1),
    ('Completed', 'Successfully completed',      1, 1),
    ('Excluded',  'Excluded by institution',     1, 1),
    ('LOA',       'Leave of absence',            0, 1);

CREATE TABLE silver.ref_visa_type (
    visa_code     VARCHAR(10)  NOT NULL PRIMARY KEY,
    visa_name     VARCHAR(100) NOT NULL,
    is_domestic   BIT          NOT NULL DEFAULT 0
);

CREATE TABLE silver.ref_delivery_mode (
    mode_code     CHAR(10)     NOT NULL PRIMARY KEY,
    mode_name     VARCHAR(50)  NOT NULL
);

INSERT INTO silver.ref_delivery_mode VALUES
    ('ONCAMPUS', 'On-campus'),
    ('ONLINE',   'Online'),
    ('BLENDED',  'Blended delivery');

-- Core tables
CREATE TABLE silver.student (
    student_id          VARCHAR(20)   NOT NULL,
    hubspot_contact_id  VARCHAR(50)   NULL,
    dynamics_contact_id VARCHAR(50)   NULL,
    student_first_name  VARCHAR(100)  NOT NULL,
    student_last_name   VARCHAR(100)  NOT NULL,
    student_email       VARCHAR(255)  NOT NULL,
    student_dob         DATE          NULL,
    visa_code           VARCHAR(10)   NOT NULL REFERENCES silver.ref_visa_type(visa_code),
    is_international    BIT           NOT NULL DEFAULT 0,
    created_datetime    DATETIME2     NOT NULL,
    updated_datetime    DATETIME2     NOT NULL,
    _source_system      VARCHAR(20)   NOT NULL,
    _loaded_at          DATETIME2     NOT NULL,
    CONSTRAINT pk_student         PRIMARY KEY (student_id),
    CONSTRAINT uq_student_email   UNIQUE (student_email)
);

CREATE TABLE silver.course (
    course_id               INT           NOT NULL IDENTITY(1,1),
    course_code             VARCHAR(20)   NOT NULL,
    course_name             VARCHAR(200)  NOT NULL,
    qualification_level     VARCHAR(50)   NOT NULL,
    credit_point_count      INT           NOT NULL,
    duration_semester_count INT           NOT NULL,
    is_active               BIT           NOT NULL DEFAULT 1,
    CONSTRAINT pk_course       PRIMARY KEY (course_id),
    CONSTRAINT uq_course_code  UNIQUE (course_code)
);

CREATE TABLE silver.unit (
    unit_id             INT           NOT NULL IDENTITY(1,1),
    unit_code           VARCHAR(20)   NOT NULL,
    unit_name           VARCHAR(200)  NOT NULL,
    credit_point_count  INT           NOT NULL DEFAULT 0,
    is_active           BIT           NOT NULL DEFAULT 1,
    CONSTRAINT pk_unit       PRIMARY KEY (unit_id),
    CONSTRAINT uq_unit_code  UNIQUE (unit_code)
);

CREATE TABLE silver.course_unit (
    course_id           INT  NOT NULL REFERENCES silver.course(course_id),
    unit_id             INT  NOT NULL REFERENCES silver.unit(unit_id),
    unit_sequence_number INT NOT NULL,
    is_core             BIT  NOT NULL DEFAULT 1,
    CONSTRAINT pk_course_unit PRIMARY KEY (course_id, unit_id)
);

CREATE TABLE silver.campus (
    campus_id    INT          NOT NULL IDENTITY(1,1),
    campus_code  VARCHAR(10)  NOT NULL,
    campus_name  VARCHAR(100) NOT NULL,
    state_code   CHAR(3)      NOT NULL,
    country_name VARCHAR(50)  NOT NULL DEFAULT 'Australia',
    is_active    BIT          NOT NULL DEFAULT 1,
    CONSTRAINT pk_campus       PRIMARY KEY (campus_id),
    CONSTRAINT uq_campus_code  UNIQUE (campus_code)
);

CREATE TABLE silver.semester (
    semester_id     INT          NOT NULL IDENTITY(1,1),
    semester_code   VARCHAR(20)  NOT NULL,
    semester_name   VARCHAR(100) NOT NULL,
    academic_year   INT          NOT NULL,
    semester_number INT          NOT NULL,
    start_date      DATE         NOT NULL,
    end_date        DATE         NOT NULL,
    CONSTRAINT pk_semester       PRIMARY KEY (semester_id),
    CONSTRAINT uq_semester_code  UNIQUE (semester_code)
);

CREATE TABLE silver.agent (
    agent_id         VARCHAR(50)   NOT NULL,
    agent_name       VARCHAR(200)  NOT NULL,
    agent_email      VARCHAR(255)  NOT NULL,
    agent_country_code CHAR(3)     NULL,
    commission_rate  DECIMAL(5,4)  NULL,
    is_active        BIT           NOT NULL DEFAULT 1,
    CONSTRAINT pk_agent PRIMARY KEY (agent_id)
);

CREATE TABLE silver.application (
    application_id          VARCHAR(50)  NOT NULL,
    student_id              VARCHAR(20)  NOT NULL REFERENCES silver.student(student_id),
    course_id               INT          NOT NULL REFERENCES silver.course(course_id),
    campus_id               INT          NULL     REFERENCES silver.campus(campus_id),
    semester_id             INT          NOT NULL REFERENCES silver.semester(semester_id),
    mode_code               CHAR(10)     NOT NULL REFERENCES silver.ref_delivery_mode(mode_code),
    agent_id                VARCHAR(50)  NULL     REFERENCES silver.agent(agent_id),
    application_date        DATE         NOT NULL,
    application_status_code VARCHAR(20)  NOT NULL,
    source_system           VARCHAR(20)  NOT NULL,
    _loaded_at              DATETIME2    NOT NULL,
    CONSTRAINT pk_application PRIMARY KEY (application_id)
);

CREATE TABLE silver.enrolment (
    enrolment_id            VARCHAR(50)   NOT NULL,
    student_id              VARCHAR(20)   NOT NULL REFERENCES silver.student(student_id),
    unit_id                 INT           NOT NULL REFERENCES silver.unit(unit_id),
    course_id               INT           NOT NULL REFERENCES silver.course(course_id),
    campus_id               INT           NOT NULL REFERENCES silver.campus(campus_id),
    semester_id             INT           NOT NULL REFERENCES silver.semester(semester_id),
    mode_code               CHAR(10)      NOT NULL REFERENCES silver.ref_delivery_mode(mode_code),
    enrolment_date          DATE          NOT NULL,
    enrolment_status_code   VARCHAR(20)   NOT NULL REFERENCES silver.ref_enrolment_status(status_code),
    withdrawal_date         DATE          NULL,
    completion_date         DATE          NULL,
    tuition_amount          DECIMAL(15,2) NOT NULL DEFAULT 0
        CONSTRAINT chk_tuition_positive CHECK (tuition_amount >= 0),
    _source_system          VARCHAR(20)   NOT NULL,
    _loaded_at              DATETIME2     NOT NULL,
    CONSTRAINT pk_enrolment        PRIMARY KEY (enrolment_id),
    CONSTRAINT uq_enrolment_grain  UNIQUE (student_id, unit_id, semester_id)  -- BR-003
);

CREATE TABLE silver.assessment_attempt (
    attempt_id           VARCHAR(50)   NOT NULL,
    enrolment_id         VARCHAR(50)   NOT NULL REFERENCES silver.enrolment(enrolment_id),
    assessment_task_code VARCHAR(50)   NOT NULL,
    attempt_number       INT           NOT NULL DEFAULT 1,
    submission_datetime  DATETIME2     NULL,
    grade_score          DECIMAL(5,2)  NULL
        CONSTRAINT chk_grade_range CHECK (grade_score IS NULL OR (grade_score >= 0 AND grade_score <= 100)),
    grade_code           VARCHAR(10)   NULL,
    is_pass              BIT           NULL,
    _loaded_at           DATETIME2     NOT NULL,
    CONSTRAINT pk_assessment_attempt PRIMARY KEY (attempt_id)
);
```

---

## dbt model examples (Silver to Gold)

```sql
-- models/silver/stg_enrolment.sql
WITH raw AS (
    SELECT * FROM {{ source('paradigm', 'enrolment') }}
),
cleaned AS (
    SELECT
        TRIM(enrolment_id)                                      AS enrolment_id,
        TRIM(student_id)                                        AS student_id,
        TRIM(unit_code)                                         AS unit_code,
        TRIM(course_code)                                       AS course_code,
        TRIM(campus_code)                                       AS campus_code,
        TRIM(semester_code)                                     AS semester_code,
        CAST(enrolment_date AS DATE)                            AS enrolment_date,
        UPPER(TRIM(status))                                     AS enrolment_status_code,
        TRY_CAST(withdrawal_date AS DATE)                       AS withdrawal_date,
        TRY_CAST(completion_date AS DATE)                       AS completion_date,
        ISNULL(TRY_CAST(tuition_amount AS DECIMAL(15,2)), 0)    AS tuition_amount,
        'PARADIGM'                                              AS _source_system,
        GETUTCDATE()                                            AS _loaded_at
    FROM raw
    WHERE enrolment_id IS NOT NULL AND student_id IS NOT NULL
),
deduped AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY enrolment_id ORDER BY _loaded_at DESC
        ) AS rn
    FROM cleaned
)
SELECT * EXCLUDE rn FROM deduped WHERE rn = 1
```

```yaml
# models/silver/schema.yml — field-level integrity as dbt tests
models:
  - name: stg_enrolment
    description: "Staged enrolment records from Paradigm SIS — cleaned and deduplicated."
    columns:
      - name: enrolment_id
        tests: [not_null, unique]
      - name: student_id
        tests: [not_null]
      - name: enrolment_status_code
        tests:
          - not_null
          - accepted_values:           # implements ref_enrolment_status validation
              values: ['Active','Withdrawn','Deferred','Completed','Excluded','LOA']
      - name: tuition_amount
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: "tuition_amount >= 0"  # BR-002
```

---

## Design decisions documented (rules bent/broken)

```
Decision 1: Bronze layer has no PKs or FKs enforced
  Rule violated:  Every table must have a primary key (Ch07/Ch08)
  Reason:         Bronze mirrors source systems exactly for auditability.
                  Sources may have duplicate records, Nulls in IDs, dirty data.
  Integrity cost: No table-level integrity in bronze. Duplicates possible.
  Mitigation:     Silver layer enforces all integrity. Bronze → Silver transformation
                  deduplicates and validates.
  Documentation:  This is Circumstance 1 (analytical/transit zone design) from Ch15.

Decision 2: Gold layer materialises calculated fields
  Rule violated:  Tables must not contain calculated fields (Ch07)
  Reason:         Power BI DirectLake mode requires pre-computed measures in Delta tables
                  for acceptable query performance on large fact tables.
  Integrity cost: Materialised values may become stale if source changes without reload.
  Mitigation:     dbt incremental models reload on schedule. Values recalculated on each run.
  Documentation:  Performance exception per Ch15. Silver layer remains pure.

Decision 3: dim_date includes flattened semester hierarchy
  Rule violated:  No redundant data (semester_name and academic_year both derivable)
  Reason:         Star schema standard for Power BI dimensional model. Avoids extra join.
  Integrity cost: Denormalised — semester_name change requires full dim_date reload.
  Mitigation:     Semester data is extremely stable (changes < once/year).
  Documentation:  Analytical design exception per Ch15.
```

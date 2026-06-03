# ch08 — Conceptual Data Models (CDM)

## What problem does this solve and why does it exist

Most analytics projects fail at scope, not at technology. The engineering team builds the wrong thing because the business question was never precisely defined, or different stakeholders had different answers to "what is a student?", or the team built the model for one department without knowing another department needed the same data at a different grain.

The CDM is the forcing function that makes these disagreements explicit before any code is written. It is the document you take into the room with the CCO, not the one you take to the database.

---

## The core definition

> A conceptual data model (CDM) is a set of symbols and text representing the key concepts and rules binding them for a specific business scope and audience.

**Key constraint:** Limit to one page. If it doesn't fit on one page, it's at the wrong level of abstraction. Aim for 10–20 concepts. A concept that does not make the top-20 list belongs in the logical model.

---

## Relational CDM vs Dimensional CDM

The same business domain produces two different CDMs depending on what you are building:

| | Relational CDM | Dimensional CDM |
|-|---------------|----------------|
| **Purpose** | Capture how the business works (rules) | Capture what the business is monitoring (questions) |
| **Output** | Entities and their relationships | Meters, dimensions, hierarchies |
| **Audience** | Business analysts, architects | Reporting consumers, analytics teams |
| **Contains** | M:M relationships OK | Grain matrix, measure-dimension mapping |

---

## Five-step CDM process

### Step 1 — Ask the five strategic questions

Before touching a whiteboard, answer these. Skipping them means you will revisit scope mid-project.

1. **What is this application going to do?** (Scope statement — two sentences max)
2. **"As is" or "to be"?** Are you modelling the current state or a desired future state?
3. **Is analytics a requirement?** If yes, at least part of the solution must be dimensional. Relational modelling captures rules; dimensional modelling captures questions.
4. **Who is the audience?** Who validates the CDM? Who uses it? If they vary widely in technical ability, you may need two forms.
5. **Flexibility or simplicity?** Flexible models use generic terms (Event, Party, Role) — they accommodate future change but are harder to read. Simple models use specific terms (Order, Customer, Employee) — easier to read but need rework if the business changes.

### Step 2 — Identify and define the concepts

**For relational:** Use the 6-category framework from ch04. Fill in a concept template:

```
JMC Academy — Student Platform Conceptual Concepts

Who:    Student, Staff, Agent, Employer
What:   Course, Unit, Qualification
When:   Semester, Academic Year
Where:  Campus, Delivery Mode
Why:    Application, Enrolment, Assessment Attempt, Completion
How:    Offer Letter, Academic Transcript, Invoice
```

Write a short definition for each concept. This is where disagreements surface. "Does Student include withdrawn students?" — answer this now.

**For dimensional:** Collect business questions, then extract measures and dimensions:

```
Business questions gathered from JMC stakeholders:

Q1: How many students enrolled by course and semester this year? [Admissions]
Q2: What is the assessment pass rate by unit and campus? [Academic Quality]
Q3: How many students completed vs withdrew by qualification and year? [Student Services]
Q4: What is the time from application to enrolment by intake? [Marketing]

Extracted:
Measures: student_count, pass_rate, completion_count, withdrawal_count, days_to_enrolment
Dimensions: course, unit, campus, semester, year, qualification, intake
```

### Step 3 — Capture the relationships

**For relational:** For each pair of entities, ask eight questions (two on participation, two on optionality, four on subtyping):

```
Can a Student have more than one Enrolment?          Yes → many on Enrolment side
Can an Enrolment belong to more than one Student?    No  → one on Student side
Can a Student exist without an Enrolment?            Yes → O (optional) on Enrolment side
Can an Enrolment exist without a Student?            No  → mandatory on Student side
Are there types of Student worth showing?            Yes → add subtyping (Domestic/International)
```

**For dimensional:** Build a **Grain Matrix**. Rows = dimensional levels, columns = measures, cells = which questions use each combination.

```
Grain Matrix — JMC Student Analytics

                            student_count   pass_rate   days_to_enrol
                            (Q1,Q3)         (Q2)        (Q4)
──────────────────────────────────────────────────────────────────────
Course                           ✓             ✓
Unit                                           ✓
Campus                           ✓             ✓
Semester                         ✓             ✓
Year                             ✓                           ✓
Qualification                    ✓
Intake                                                        ✓
Delivery Mode                    ✓             ✓
```

The grain matrix reveals which questions share the same dimensions (and can share the same fact table) and which need different grains.

### Step 4 — Determine the most useful form

**If the audience knows data modeling:** Use standard IE crow's foot notation.

**If the audience is non-technical:** Use the Axis Technique.

The Axis Technique puts the business process in the centre and dimensions on axes radiating outward. Notches on each axis represent levels of granularity.

```
                        Year
                         |
                    Semester
                         |
Campus ──── Delivery Mode ──── [Enrolment Count] ──── Course ──── Qualification
                         |
                        Unit
                         |
                     Instructor
```

This communicates the dimensional model to a CCO in two minutes without explaining what a fact table is.

### Step 5 — Review and confirm

Walk validators through the model. Changes at this stage cost nothing. Changes after the LDM is built cost time. Changes after the database is populated cost money.

---

## Decision criteria

**Use a relational CDM when:**
- You are building an OLTP system (CRM, enrolment system, LMS)
- The primary need is enforcing business rules and data integrity
- The audience cares about "how does the business work?"

**Use a dimensional CDM when:**
- You are building an analytics system (data mart, semantic model, Power BI dataset)
- The primary need is answering business questions at different granularities
- The audience cares about "how is the business performing?"

**You usually need both.**
Your JMC Fabric platform needs a relational model for the silver layer (3NF, source-of-truth) and a dimensional model for the gold layer (star schema, Power BI).

---

## Common mistakes

**Mistake 1 — Skipping the CDM and going straight to the LDM.**
You will discover scope disagreements six weeks into the project. The CCO's "student" is different from the admissions team's "student". A CDM surfaces this in week one.

**Mistake 2 — CDM with 50 concepts.**
If you have more than 20, you are at the wrong level. Collapse detail into higher concepts. `Enrolment` subsumes `EnrolmentStatus`, `EnrolmentDate`, `EnrolmentFee` — those are attributes, not concepts.

**Mistake 3 — Definitions are vague or missing.**
"Student: a person who studies at JMC" is incomplete. Does it include applicants? Withdrawn students? Graduated alumni? Get a written, agreed answer. The definition is more important than the box.

**Mistake 4 — Using the grain matrix for only one department.**
If Admissions and Academic Quality both need `student_count by course`, the grain matrix reveals they can share a single fact table. Building two separate mart tables for the same measure is expensive and creates inconsistency.

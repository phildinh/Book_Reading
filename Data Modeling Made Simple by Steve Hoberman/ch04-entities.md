# ch04 — Entities

## What problem does this solve and why does it exist

Before you can design any schema, you need a disciplined way to decide *what things* your system tracks. Without a framework, engineers track whatever seems obvious, and the result is: missing entities (things the business cares about that have no table), duplicate entities (the same concept stored in three different places), and misclassified entities (a transaction treated as a dimension).

An entity is the answer to "what does this business need to remember?" Hoberman's framework gives you a classification system for naming and categorising these things consistently.

---

## The core definition

> An entity represents a collection of information about something that the business deems important and worthy of capture. It is always a noun or noun phrase.

An entity at the logical level becomes a table at the physical level. An entity *instance* is a single row.

---

## The 6-category framework

Every entity fits one of six categories. This is a forcing function — if you cannot classify an entity, it probably isn't a real entity.

| Category | Question | Examples |
|----------|----------|---------|
| **Who** | Who is important to the business? | Student, Staff, Applicant, Employer |
| **What** | What does the business make or deliver? | Course, Unit, Qualification, Resource |
| **When** | What time intervals matter? | Semester, Academic Year, Intake, Term |
| **Where** | Where does business happen? | Campus, Classroom, Online Platform, State |
| **Why** | What events keep the business running? | Enrolment, Application, Withdrawal, Assessment |
| **How** | How are events documented? | Enrolment Form, Invoice, Certificate, Transcript |

**The Why vs How distinction matters.** These look similar but are different entities:

- **Why = the event itself.** An Enrolment is the act of a student becoming enrolled in a unit. It happens at a point in time.
- **How = the document recording it.** An Enrolment Confirmation is the PDF that proves the enrolment happened.

In a data model, the Why becomes a fact table. The How may become a separate document/audit entity or may be collapsed into the Why depending on reporting needs.

---

## Three levels of an entity

The same concept appears differently across model levels:

```
Conceptual          Logical                     Physical
──────────────      ──────────────────────      ──────────────────────────
Student         →   Student                 →   dim_student
                    Student_Address             (SCD Type 2, surrogate key,
                    Student_Contact_Pref        all attributes denormalised)
                    Student_Demographics
```

**At conceptual level:** one rectangle labelled "Student". High enough for a CCO to understand.

**At logical level:** Student expands into multiple normalised entities. Each attribute is a fact about only its primary key. No technology decisions yet.

**At physical level:** logical entities are collapsed, denormalised, indexed, and partitioned for query performance. Student and Student_Demographics might be merged into a single wide table if they are always queried together.

**The mistake:** jumping straight to the physical without going through the logical. You end up with a table that has no clear primary key, attributes that are facts about other attributes (hidden dependencies), and no agreed definitions for anything.

---

## How to identify entities in practice

Run the 6-category checklist against the business domain:

```
JMC Academy domain:

Who:   Student, Staff, Applicant, Agent, Employer, Instructor
What:  Course, Unit, Qualification, LMS Module
When:  Semester, Academic Year, Study Period, Intake Cohort
Where: Campus, Delivery Mode (Online/On-campus), State/Territory
Why:   Application, Enrolment, Assessment Attempt, Withdrawal, Deferral, Completion
How:   Offer Letter, Enrolment Confirmation, Academic Transcript, Invoice
```

If something cannot be classified, question whether it is an entity at all — it might be an attribute of another entity, or it might be a relationship resolving a many-to-many.

---

## The instance test

Before finalising an entity, ask: can I give three concrete examples of it?

- `Student` → Phil Dinh, Jane Smith, Bob Chen ✓
- `Enrolment` → Phil's enrolment in BSBITA401 in Semester 2 2024 ✓
- `Academic Performance` → not an entity, it's a set of attributes and calculations ✗

If you cannot name concrete examples, the entity is probably an attribute, a derived metric, or a concept that needs more decomposition.

---

## Associative entities (resolving many-to-many)

When two entities have a many-to-many relationship, the resolution is an associative entity. It sits between them and holds the intersection data.

```
Student ──<M:M>── Course

Becomes:

Student ──<1:M>── Enrolment ──<M:1>── Course

Enrolment is the associative entity.
It holds: enrolment_date, status, grade, etc.
```

**Rule:** Every many-to-many relationship in your LDM needs an associative entity. At the physical level, this becomes a fact table (for analytics) or a junction table (for OLTP).

---

## Common mistakes

**Mistake 1 — Naming an entity after a report, not a concept.**
`StudentEnrolmentReport` is not an entity. `Enrolment` is. Name entities after things that exist in the business, not after how you want to display them.

**Mistake 2 — Creating an entity for every source system table.**
`HubSpot_Contact` is a source table, not a business entity. The business entity is `Student` (or `Applicant`). Your bronze layer mirrors source tables; your silver/gold layers model business entities.

**Mistake 3 — Hiding a Why inside a What.**
Treating `EnrolledCourse` as a "What" entity when it is actually an Enrolment event (Why). If the concept has a date, a status, and changes over time, it is probably a Why.

**Mistake 4 — Not separating the event from its document.**
Merging the Enrolment event with the Enrolment Confirmation document into one entity causes confusion about grain. The event is one row per student+unit. The document is one row per issued confirmation, which may be re-issued.

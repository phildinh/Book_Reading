# ch04–05 — Conceptual Overview & Starting the Process

## What problem does this solve and why does it exist

Most database failures are failures of process, not failures of technical skill. A developer who starts building tables before understanding what the database is supposed to do will build the wrong thing, with high confidence. These chapters give you the complete map of the seven-phase process before you start, and teach you how to gather the information that makes everything else correct: the mission statement and mission objectives.

---

## Ch04 — The Seven-Phase Design Process

### Why you must complete the entire process

> The level of structural integrity and data integrity in your database is directly proportional to how thoroughly you follow the design process.

Partially following the design process is almost as bad as not following it at all. An incomplete design is a poor design. Hernandez is explicit and repeated on this point. Never shortcut.

### The seven phases in order

```
Phase 1: Define a mission statement and mission objectives
          Purpose anchor + task inventory

Phase 2: Analyze the current database
          Understanding where you are before designing where you go

Phase 3: Create the data structures
          Tables → fields → keys → field specifications

Phase 4: Determine and establish table relationships
          Identify relationships, establish FK connections, set characteristics

Phase 5: Determine and define business rules
          Field-specific and relationship-specific constraints

Phase 6: Determine and define views
          Data access, information retrieval, security

Phase 7: Review data integrity
          Final checklist across all four integrity levels
```

Each phase builds on all previous phases. Each produces specific artefacts that the next phase consumes. You cannot do Phase 3 properly without the artefacts from Phase 2. You cannot do Phase 4 without Phase 3. The process is sequential and cumulative.

---

## Ch05 — Starting the Process: Mission Statement & Objectives

### Conducting interviews

Interviews are the primary tool for gathering information throughout the entire design process. Everything downstream depends on what you learn in them. Hernandez's guidelines:

**Before the interview:**
- Tell participants what the subject is, who else is involved, the start time, and that it is not an assessment of their performance — you want them comfortable and open
- Prepare your questions in advance — never ad hoc
- Use **open-ended questions**: "How would you describe the work you do?" not "Does X happen?"
- Keep the number of participants per session small — reduces intimidation
- Conduct separate sessions for users and management (different perspectives, avoid conflict)

**During the interview:**
- Give everyone equal and undivided attention
- **Maintain control** — this is the single most important rule; redirect discussions that go off-topic
- Keep the pace moving; set personal time limits per question
- If a participant monopolises, diplomatically redirect

**Two essential techniques used in every subsequent interview:**

**Subject-Identification Technique** — listen for nouns that represent persons, places, things, or events. These become potential table candidates.

```
Response: "I accept land-use applications submitted by applicants, log them in,
          and set a hearing date with the hearing examiner."

Nouns identified: application, applicant, hearing date, hearing examiner
These are potential TABLES.
```

**Characteristic-Identification Technique** — after identifying a subject, ask follow-up questions about its characteristics. Listen for nouns that describe particular aspects of that subject.

```
Follow-up: "What facts are associated with an application?"
Response: "The type of application, the name of the subdivision, the purpose, a description."

Characteristics identified: type, subdivision name, purpose, description
These become FIELDS in the Application table.
```

The distinction: subjects are usually in possessive form ("the client's phone number") — the possessive noun signals the table. Characteristics are in singular form ("phone number") — these become fields.

### The mission statement

**Purpose:** Declare why the database exists. Provides a fixed anchor for all subsequent design decisions. If a proposed feature doesn't serve the mission, it doesn't belong in the database.

**Two rules for writing it:**
1. Succinct — one or two sentences. Verbose statements obscure purpose.
2. Free of task descriptions — tasks belong in mission objectives, not here.

**Bad example:**
> The purpose of the Whatcom County Hearing Examiner's database is to keep track of applications for land use, maintain data on applicants, keep a record of all hearings, keep a record of all decisions, keep a record of all appeals, maintain data on department employees, and maintain data for general office use.

Problems: verbose, unclear specific purpose, describes tasks, appears incomplete.

**Good rewrite:**
> The purpose of the Whatcom County Hearing Examiner's database is to maintain the data the examiner's office uses to make decisions on land-use requests submitted by citizens of Whatcom County.

Test: is the purpose unambiguous? Can someone read it and know exactly what this database does and doesn't do?

**Interview question templates for generating a mission statement:**
- "How would you describe the purpose of your organisation to a new client?"
- "What is the single most important function of your organisation?"
- "How would you define the main focus of your organisation?"

### Mission objectives

**Purpose:** Represent the general tasks users can perform against the data. Each objective represents exactly one task. Used throughout the design process to define table structures, field specifications, relationship characteristics, and views.

**Two rules for writing them:**
1. Each is a declarative sentence that clearly defines a single general task
2. Free from unnecessary detail — say *what*, not *how*

**Bad example:**
> We need to keep track of the entertainers we represent and the type of entertainment they provide, as well as the engagements that we book for them.

Problems: describes two tasks (entertainers + engagements), contains unnecessary detail ("type of entertainment").

**Good rewrite:**
> Maintain complete entertainer information.
> Keep track of all the engagements we book.

**Key technique:** "read between the lines." A participant who says "I book entertainment for clients" is implicitly telling you that you need tables for entertainers, clients, and engagements — even if they only explicitly mentioned booking.

**Validation rule:** review the complete list of objectives with all participants. When everyone agrees the list is relatively complete, commit it to a document. "Relatively complete" — you will always discover a few more as the process unfolds. That is normal.

### Applied example — JMC Academy

```
Mission Statement:
The purpose of the JMC Academy Student Platform database is to maintain
the data we use to support student services from first lead inquiry through
to graduation, including admissions, academic management, and compliance.

Mission Objectives:
- Maintain complete applicant and student information.
- Track all applications and their outcomes.
- Maintain complete course and unit information.
- Track enrolments and their statuses.
- Track assessment attempts and outcomes.
- Maintain campus and delivery mode information.
- Record semester and academic year scheduling.
- Maintain agent information for international student referrals.
- Track student withdrawals, deferrals, and completions.
- Support reporting across all stages of the student lifecycle.
```

Each objective is one task. Each will generate one or more table candidates in Phase 3. The mission statement will reject any proposed table that doesn't serve student lifecycle management.

---

## Common mistakes

**Mistake 1 — Confusing tasks with purpose in the mission statement.**
Tasks belong in objectives. The mission statement should express the *why*, not the *what*.

**Mistake 2 — Objectives that cover more than one task.**
"Maintain complete entertainer and engagement information" is two objectives. Split it.

**Mistake 3 — Skipping the review with participants.**
The list you produce from interviews will be incomplete. The review is how you catch the gaps. Without it, you miss tables.

**Mistake 4 — Using closed questions.**
"Does the system track invoice dates?" gives you yes/no. "How do you track billing for orders?" gives you a picture of the whole billing process. Open-ended questions reveal implicit information that closed questions cannot.

**Mistake 5 — Conducting joint user/management interviews.**
Management sees the strategic picture. Users see the operational detail. They interpret the same question differently. Mixing them creates conflict and suppresses candid answers from both groups.

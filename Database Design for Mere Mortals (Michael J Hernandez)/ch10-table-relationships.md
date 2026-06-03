# ch10 — Table Relationships

## What problem does this solve and why does it exist

Tables in isolation cannot answer real business questions. "Which students are enrolled in this course?" requires combining Student + Enrolment + Course. Relationships are what makes that combination valid, efficient, and integrity-safe. Without properly established relationships you get: data retrieval that's impossible or tedious, orphaned records (orders with no customer), duplicate data explosions, and insert/update/delete anomalies.

A properly defined relationship establishes **relationship-level integrity** — the third component of overall data integrity.

---

## Three types of relationships

### One-to-One (1:1)

A single record in Table A relates to at most one record in Table B, and a single record in Table B relates to at most one record in Table A.

**When it occurs:**
- Table B is a subset table of Table A (they share the same PK)
- Security or confidentiality split: sensitive fields (salary, medical) separated from non-sensitive fields (name, contact)

**Diagram pattern:** `A ——|——|—— B`

**Establish it:** Take a copy of Table A's PK → incorporate into Table B as FK. In most 1:1 cases, the FK in the child table also serves as that table's PK.

```
EMPLOYEES ——|——|—— COMPENSATION
Employee_ID (PK) in EMPLOYEES
Employee_ID (PK + FK) in COMPENSATION
→ The only way to find a salary record is by knowing the employee.
```

### One-to-Many (1:M)

A single record in Table A can relate to one or more records in Table B. A single record in Table B relates to only one record in Table A.

**Most common relationship in any database.** The table on the "one" side is the **parent**; the "many" side is the **child**.

**Establish it:** Take a copy of the parent's PK → incorporate into the child table as FK.

```
CUSTOMERS ——|——<—— ORDERS
Customer_ID (PK) in CUSTOMERS
Customer_ID (FK) in ORDERS
→ One customer → many orders. Each order → one customer.
```

**Diagram pattern:** `A ——|——<—— B` (crow's foot on the "many" side)

### Many-to-Many (M:M)

A single record in Table A can relate to many records in Table B, and vice versa.

**Cannot be established directly** — the inherent structure is unresolvable without a third table.

**The two wrong approaches (both fail):**
1. Adding multiple copies of Table B's PK into Table A (creates multivalued field — fixed maximum, query problems)
2. Adding a few Table B fields into Table A (introduces unnecessary duplicate fields, massive redundant data, insert/delete anomalies)

**The correct approach: a linking table (junction table, associative table).**

**Establish it in three steps:**
1. Create a new linking table using copies of the PK from each table as its composite PK
2. Each copied PK also serves as a FK establishing a 1:M with its source table
3. Name the linking table to reflect the relationship it represents; add to Final Table List

```
STUDENTS ——<—— STUDENT_CLASSES ——>—— CLASSES

STUDENT_CLASSES:
  Student_ID    (CPK / FK → STUDENTS.Student_ID)
  Class_ID      (CPK / FK → CLASSES.Class_ID)

Result:
  STUDENTS to STUDENT_CLASSES: 1:M
  CLASSES to STUDENT_CLASSES: 1:M
  Original M:M is dissolved into two 1:M relationships
```

**Fields in the linking table:** The linking table may need additional fields beyond the two FKs. For example, STUDENT_CLASSES might also hold `Term` (to allow a student to retake a class in a later term) or `Grade`.

**Checking for misplaced fields:** If Order and Product have an M:M relationship, `Quantity_Ordered` and `Quote_Price` belong in the ORDER_DETAILS linking table — not in Orders (where they repeat per line item) and not in Products (where they represent a transaction value, not a product property).

---

## Self-referencing (recursive) relationships

A relationship that exists between records **within the same table**. Can be 1:1, 1:M, or M:M.

**1:1 self-referencing (member sponsors one member):**
```
MEMBERS: Member_ID (PK), Sponsor_ID (FK → Member_ID), ...
```

**1:M self-referencing (manager manages many staff):**
```
STAFF: Staff_ID (PK), Manager_ID (FK → Staff_ID), ...
→ One manager can have many direct reports. Each staff member reports to one manager.
```

**M:M self-referencing (parts contain parts):**
```
PARTS: Part_ID (PK), ...
PART_COMPONENTS: Part_ID (CPK/FK), Component_ID (CPK/FK → Part_ID)
→ A clamp assembly contains 3 parts. That clamp assembly is itself a component of 2 assemblies.
```

**Caution with 1:M self-referencing:** The presence of a recursive relationship sometimes signals missing table structures. If staff has a "manages" relationship, should there be a DEPARTMENTS table? Interview users to determine whether the recursive relationship is the best model or whether new tables are needed.

---

## Identifying relationships — the table matrix method

Before establishing any relationship, identify all of them first using a **table matrix**.

1. List all table names across the top AND down the left side (same order)
2. For each pair, ask questions to determine the relationship from each perspective
3. Record the result in the matrix cell (1:1, 1:N, M:N)
4. Apply formulas to determine the **official** relationship type

**Two types of questions:**

*Associative:* "Can a single record in [Table A] be associated with one or more records in [Table B]?"

*Contextual — ownership:* "Can a single order contain one or more products?"
*Contextual — action:* "Does a single instructor teach one or more classes?"

**Formulas to determine official relationship:**
```
1:1 + 1:1 = 1:1
1:N + 1:1 = 1:N
1:N + 1:N = M:N
```

Apply the formula to both cells for each pair. The formula output is the official type.

```
Matrix example:

                Buildings   Classes   Rooms
Buildings                   1:N       1:N
Classes         1:1                   1:N
Rooms           1:1         1:1

BUILDINGS ↔ CLASSES:
  Buildings row, Classes column: 1:N (one building holds many classes)
  Classes row, Buildings column: 1:1 (one class is in one building)
  Formula: 1:N + 1:1 = 1:N   → official type: 1:N

Diagram: BUILDINGS ——|——<—— CLASSES
```

---

## Establishing relationships — the FK mechanics

### 1:1 and 1:M
1. Identify which table is parent (the "one" side — the more fundamental subject)
2. Take a copy of the parent's PK
3. Incorporate into the child table as a FK
4. Mark the FK with "FK" in your table structure

### M:M
Use the linking table procedure above.

### After establishing all relationships
Review each table structure one more time against the **Elements of the Ideal Table** (Ch07). New fields and tables were created during relationship establishment — verify they all comply.

---

## Refining foreign keys — Elements of a Foreign Key

Every FK must satisfy all three elements:

**Element 1 — Same name as the primary key from which it was copied.**
This is not a style preference — it is a design rule. If `Customer_ID` is the PK in CUSTOMERS, it must be named `Customer_ID` (not `Client_Num`, not `Cust_ID`) in ORDERS.

Why this matters: different names create ambiguity about whether the FK truly refers to the PK it claims to. After 6 months, no one will remember whether `Emp_Num` in ORDERS is the same field as `Employee_Number` in EMPLOYEES.

Exception: self-referencing relationships where the PK and FK must coexist in the same table — you must use a different name. (e.g., `Manager_ID` as the FK to `Staff_ID`.)

**Element 2 — Uses a Replica field specification based on the primary key's specification.**
The FK inherits most of the PK's field spec settings, with modifications to:
- Spec Type: set to Replica
- Parent Table: the FK's parent table
- Source Specification: identifies the source PK and its table
- Description: describes the FK's purpose in its own table
- Key Type: Foreign
- Uniqueness: Non-Unique (in 1:M relationships — one FK value can appear in many child rows)
- Values Entered By: User
- Range of Values: "Any existing [PK field] in the [parent table]"
- Edit Rule: Enter Now, Edits Allowed

**Element 3 — Draws its values from the primary key to which it refers.**
A FK value must exist as a PK value in the parent table. No FK value without a corresponding PK value. This is referential integrity enforcement at the design level.

---

## Relationship characteristics — three properties to set

For each relationship, set all three before moving on.

### 1 — Deletion rule

What should happen to child records when a parent record is deleted?

| Rule | What the RDBMS does |
|------|-------------------|
| **Deny** | Rejects the delete; marks parent as inactive instead |
| **Restrict** | Rejects the delete if related child records exist; you must delete children first |
| **Cascade** | Deletes the parent AND automatically deletes all related children |
| **Nullify** | Deletes the parent; sets FK values in related children to Null |
| **Set Default** | Deletes the parent; sets FK values in related children to a specified default |

**Default choice:** Use **Restrict** as your standard. Only deviate when you have a specific, documented reason.

**Question to determine the right rule:**
> "When a record in the [parent] table is deleted, what should happen to related records in the [child] table?"

**The question for self-referencing:**
> "When a record in [table] is deleted, what should happen to the FK values of other records that were related to it?"

```
EMPLOYEES → ORDERS:
"When an employee record is deleted, what happens to related orders?"
  → "Can't delete — mark the employee inactive." (Deny)
  → "Can't delete if open orders exist." (Restrict)
  → "Delete all their orders too." (Cascade — dangerous for business records)
  → "Null out the employee ID on affected orders." (Nullify)
  → "Reassign affected orders to the lead salesperson." (Set Default)
```

Mark the deletion rule on the relationship diagram below the connection line on the parent side: `(D)` `(R)` `(C)` `(N)` `(S)`.

### 2 — Type of participation

Does a record need to exist in Table A before you can enter a record in Table B?

| Type | Meaning |
|------|---------|
| **Mandatory** | At least one record must exist in this table before entering records in the related table |
| **Optional** | No requirement for records in this table before entering records in the related table |

Mark: a **vertical line** (|) for mandatory, a **circle** (○) for optional, placed outside the relationship cardinality symbol.

```
EMPLOYEES ——|——<○—— CUSTOMERS

EMPLOYEES: Mandatory (at least one employee must exist before assigning to a customer)
CUSTOMERS: Optional (an employee need not have any customers assigned)
```

### 3 — Degree of participation

The minimum and maximum number of records in one table that can be associated with a single record in the related table.

Written as `(min, max)`. Use `N` for unlimited maximum.

```
(1,1)  = exactly one
(0,1)  = zero or one
(0,N)  = zero or unlimited
(1,N)  = at least one, unlimited max
(1,15) = at least one, no more than 15
(5,20) = at least five, no more than 20
```

```
EMPLOYEES ——(1,1)——(D)——(0,N)—— CUSTOMERS

EMPLOYEES degree (1,1): Each customer must be assigned to exactly one employee.
CUSTOMERS degree (0,N): An employee may have zero to unlimited customers.
```

Place the degree of participation **above** the connection line on the appropriate table's side.

---

## Relationship-level integrity — what it guarantees

A relationship has relationship-level integrity when:
- The FK connection is sound (matching field names, same data type, same spec)
- New records can be inserted meaningfully (participation types are correct)
- Deleting a record does not create orphaned records (deletion rule is appropriate)
- The number of interrelated records makes business sense (degree is accurate)

---

## Applied to JMC Academy — relationship matrix

```
Table matrix (key relationships):

                student  course  unit  enrolment  application  assessment
student                                  1:N          1:N
course                          1:N      1:N          1:N
unit            1:N      1:N              1:N
enrolment       N:1      N:1    N:1                              1:N
application     N:1      N:1
assessment                               N:1

Official relationships:
Student → Enrolment:      1:N (one student, many enrolments)
Course → Enrolment:       1:N (one course, many enrolments — at course level)
Unit → Enrolment:         1:N (one unit, many enrolments)
Student → Application:    1:N (one student, many applications over time)
Course → Application:     1:N (one course, many applications)
Enrolment → Assessment:   1:N (one enrolment, many assessment attempts)

Course ↔ Unit:            M:N (one course has many units; one unit appears in many courses)
→ COURSE_UNIT linking table (course_id CPK/FK, unit_id CPK/FK, sequence_number, is_core)

Relationship characteristics — Student → Enrolment:
  Deletion rule:      Restrict (cannot delete a student with active enrolments)
  Student:            Mandatory | degree (1,1) — each enrolment belongs to one student
  Enrolment:          Optional  | degree (0,N) — a student may have zero enrolments (just an applicant)

Relationship characteristics — Enrolment → Assessment_Attempt:
  Deletion rule:      Cascade (if enrolment is deleted, assessment records have no meaning)
  Enrolment:          Mandatory | degree (1,1) — each attempt belongs to one enrolment
  Assessment_Attempt: Optional  | degree (0,N) — an enrolment may have no attempts yet
```

---

## Common mistakes

**Mistake 1 — Mismatched FK names.**
`Client_Num` pointing to `Customer_ID` is ambiguous and creates maintenance debt. Name FKs identically to the PK they reference.

**Mistake 2 — Never setting deletion rules.**
Defaulting to "whatever the RDBMS does by default" (usually Restrict or no action) without consciously deciding. Different relationships require different rules. Think through each one.

**Mistake 3 — Defaulting all participation to Optional.**
Optional is not the safe default — it is a deliberate design decision. "Optional" means "records in this table can exist without any related records in the other table." Be intentional.

**Mistake 4 — Not checking for extra fields in linking tables.**
When you create a linking table for M:M, ask: are there fields in either source table that should move to the linking table? `Quantity_Ordered` and `Quote_Price` on an order line belong in ORDER_DETAILS, not in ORDERS or PRODUCTS.

**Mistake 5 — Ignoring self-referencing relationships.**
A self-referencing 1:M means "one parent, many children within the same table." If a manager relationship exists, ask whether a DEPARTMENTS table is needed. Recursive relationships are often signals that the structure is incomplete.

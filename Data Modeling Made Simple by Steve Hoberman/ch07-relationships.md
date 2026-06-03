# ch07 — Relationships

## What problem does this solve and why does it exist

Data does not live in isolation. A student belongs to a course; an enrolment connects a student to a course at a point in time; a staff member teaches many units. Without a formal way to express these connections and their rules, engineers implement FKs inconsistently, cardinality assumptions get baked into application code invisibly, and JOIN conditions become tribal knowledge.

Relationships are business rules made visible and enforced.

---

## The core definition

> A relationship is a line connecting two entities that captures a business rule or navigation path between them.

The relationship line has two ends. Each end has cardinality symbols. Cardinality tells you: how many instances of entity B can relate to one instance of entity A?

---

## Cardinality — reading the notation

Three symbols, all you need:

```
|    =  one (mandatory: "must")
O    =  zero (optional: "may")
<    =  many (crow's foot)
```

**Reading rule:** Always start from the parent (one side). Use the word "each".

```
Author ──|O──<── Title

Read parent to child:  "Each Author may write one or many Titles."
Read child to parent:  "Each Title must be written by one Author."
```

`O` (zero) → optional → "may"  
No `O` → mandatory → "must"

**The sentence test.** If you cannot read a relationship as a clear English sentence, the cardinality is wrong or the relationship label is missing.

---

## Relationship labels

The verb on the relationship line is how you read the sentence. Use precise verbs.

```
Good labels:        contain, write, enrol in, employ, teach, process, submit
Ambiguous labels:   has, have, associate, relate, be, participate
```

```
Student ──|O──<── Enrolment ──>──|──── Course

Read: "Each Student may have one or many Enrolments."
      "Each Enrolment must belong to one Student."
      "Each Enrolment must reference one Course."
      "Each Course may have one or many Enrolments."
```

---

## Identifying vs non-identifying relationships

This is the decision that drives your FK design and surrogate key strategy.

### Non-identifying (dashed line) — child is independent

The child entity has its own identity. It does not need the parent's key to be unique.

```
Customer ─ ─ ─ ─ ─ ─< Order

Order has its own Order_Number PK.
You do not need Customer_ID to find a specific Order.
Customer_ID appears in Order as a FK, but is NOT part of Order's PK.
```

### Identifying (solid line) — child is dependent

The child cannot be uniquely identified without the parent's key. The parent's PK is part of the child's composite PK.

```
Order ──────────────< Order_Line

Order_Line PK = (Order_Number, Line_Number)
You cannot find a specific Order_Line without knowing which Order it belongs to.
```

**Decision rule:**

1. Can each instance of the child entity be found using only attributes it owns? → **Non-identifying** (independent entity)
2. Does the child need the parent's key to be unique? → **Identifying** (dependent entity)

**In practice for analytics:** Most fact tables are dependent entities. `fact_enrolment` is identified by (`student_sk`, `course_sk`, `semester_sk`) — it needs all three to be unique. At the physical level you typically add a surrogate PK for simplicity, but the logical structure is identifying.

---

## Cardinality patterns and what they mean at the physical level

| Logical pattern | Physical outcome |
|----------------|-----------------|
| One-to-many | FK column in the child table |
| Many-to-many | Requires an associative entity → junction/fact table |
| One-to-one | Merge tables (denormalise), or vertical partition (physical split) |
| Recursive 1:M | Self-referencing FK (`manager_id` → `employee_id`) |
| Recursive M:M | Associative table on the same entity |

---

## Recursion

A recursive relationship exists between instances of the same entity.

**Hierarchy (1:M recursive)** — one parent per instance:
```
Employee
├── manages →  Employee
└── managed by → at most one Employee

SQL:
CREATE TABLE employee (
    employee_id    INT PRIMARY KEY,
    manager_id     INT NULL REFERENCES employee(employee_id),
    ...
);
```

**Network (M:M recursive)** — multiple parents per instance:
```
Course ──< prerequisite >── Course
(A course can have multiple prerequisites;
 a course can be a prerequisite for multiple other courses)

Requires associative entity:
CREATE TABLE course_prerequisite (
    course_id           INT REFERENCES course(course_id),
    prerequisite_id     INT REFERENCES course(course_id),
    PRIMARY KEY (course_id, prerequisite_id)
);
```

**Trade-off of recursion:** Flexible but hides rules. A hierarchy that allows any depth also allows illogical structures (A is parent of B, B is parent of A). If the hierarchy has known rules (max 3 levels, specific types per level), consider a non-recursive design instead and document the trade-off.

---

## Subtyping

Subtyping groups common attributes at the supertype level while retaining what is unique at each subtype.

```
Person (supertype)
├── Student (subtype)    → has: student_number, enrolment_date
├── Staff (subtype)      → has: employee_id, employment_type
└── Agent (subtype)      → has: agency_code, commission_rate
```

**Two classification axes:**

| Axis | Options | Meaning |
|------|---------|---------|
| Completeness | Complete / Incomplete | Are all subtypes listed? Person → {Male, Female} is complete. Account → {Checking, Savings} is incomplete (Brokerage also exists). |
| Exclusivity | Exclusive / Inclusive | Can an instance belong to multiple subtypes? A Person is exclusively Male OR Female. A Contact can be both a Student AND a Staff member (inclusive). |

**Subtyping resolves to three physical patterns (covered in ch10):**
- Identity: keep supertype + subtype as separate tables with 1:1 FK
- Rolldown: collapse supertype columns into each subtype table
- Rollup: collapse all subtype columns into the supertype table

---

## Common mistakes

**Mistake 1 — Missing the associative entity.**
Modelling a Student–Course relationship as M:M without creating an Enrolment entity. The M:M has no place to store the enrolment_date, grade, or status. Always resolve M:M into two 1:M with an associative entity in between.

**Mistake 2 — Wrong optionality.**
Making a mandatory relationship optional "just in case". If every Enrolment must belong to a Student, model it as mandatory (solid one end, no circle). Optional should be a deliberate decision, not a default.

**Mistake 3 — Ambiguous label.**
Labelling a relationship "has" or "relates to". These add no information. The reader still doesn't know what the rule is. Every label should be a specific verb that makes the business rule legible.

**Mistake 4 — Recursive hierarchy without a max depth guard.**
An unlimited self-referencing hierarchy is theoretically fine but can lead to infinite recursion in queries. Always document the expected depth and add a `level` or `depth` column when querying.

```sql
-- Recursive CTE for hierarchy traversal — always add a depth guard
WITH RECURSIVE org_hierarchy AS (
    SELECT employee_id, manager_id, 0 AS depth
    FROM employee
    WHERE manager_id IS NULL  -- root nodes

    UNION ALL

    SELECT e.employee_id, e.manager_id, h.depth + 1
    FROM employee e
    JOIN org_hierarchy h ON e.manager_id = h.employee_id
    WHERE h.depth < 10  -- depth guard prevents infinite loops
)
SELECT * FROM org_hierarchy;
```

**Mistake 5 — Using "is" or "has" as relationship names.**
These are reserved words in English grammar, not business rules. Replace with the actual verb that describes what one entity does to another.

# ch09 — Logical Data Models (LDM)

## What problem does this solve and why does it exist

A CDM tells you what concepts exist and how they relate. An LDM tells you exactly what data those concepts require — every attribute, its type, its key membership, its constraints — without making any technology decisions. The LDM is the business solution. The PDM is the technical solution.

Skipping the LDM means you mix business decisions with technology decisions. When you need to change one (the business rule changes), you have to untangle it from the other (the index type or partition strategy). That untangling is expensive.

---

## Relational LDM vs Dimensional LDM

| | Relational LDM | Dimensional LDM |
|-|---------------|----------------|
| **Purpose** | How the business works | What the business is monitoring |
| **Technique** | Normalization (mandatory) + Abstraction (optional) | Meters, dimensions, snowflakes |
| **Output** | 3NF entity-attribute model | Fact (meter) + dimension hierarchy model |
| **Maps to** | Silver layer (source of truth) | Gold layer (analytics) |

---

## Normalization

> Normalization is a formal process of asking business questions. It ensures that every attribute is a fact about the key (1NF), the whole key (2NF), and nothing but the key (3NF).

The one-sentence rule: **every attribute is single-valued and provides a fact completely and only about its primary key.**

Break this into three passes:

### 1NF — "single-valued" (the key)

**Problem:** Repeating groups and multi-valued attributes.

**Test question per attribute:** "Can a [entity] have more than one [attribute]?"
**Test question per attribute:** "Does [attribute] contain more than one piece of business information?"

```
Initial chaos — Student table:
student_id | dept_code | phone_1        | phone_2        | phone_3        | student_name    | dept_name    | start_date
123        | A         | 973-555-1212   | 678-333-3333   | 343-222-1111   | Henry Winkler   | Data Admin   | 4/1/2012

Problems:
- phone_1, phone_2, phone_3 are repeating groups → move to a separate Phone entity
- student_name contains first + last → split into student_first_name, student_last_name
- phone_2 varies by department (it's the dept phone) → belongs on Department, not Student
- phone_3 is the same for all rows (it's the org phone) → belongs on Organisation
```

**After 1NF:**
```
Student: student_id, dept_code, student_first_name, student_last_name, start_date
Department: dept_code, dept_name, dept_phone
Organisation: org_phone
StudentPhone: student_id, phone_number, phone_type_code
```

### 2NF — "completely" (the whole key)

Only applies when the PK is composite. Every non-key attribute must depend on the **entire** PK, not just part of it.

**Test question:** "Are all attributes in the primary key needed to retrieve a single instance of [attribute]?"

```
Example: OrderLine(order_id, product_id, quantity, product_name, product_price)

PK = (order_id, product_id)

- quantity: needs both order_id AND product_id → stays ✓
- product_name: needs only product_id → 2NF violation, move to Product table ✗
- product_price: needs only product_id → 2NF violation, move to Product table ✗

After 2NF:
OrderLine: order_id, product_id, quantity
Product:   product_id, product_name, product_price
```

### 3NF — "only" (nothing but the key)

Every attribute must be directly dependent on the PK, not on another non-key attribute.

**Test question:** "Is [attribute] a fact about any other attribute in this same entity?"

```
Example: Employee(employee_id, dept_code, dept_name, start_date, is_vested)

- dept_name is a fact about dept_code, not about employee_id → 3NF violation
  (if you know the dept_code, you know the dept_name — it doesn't need employee_id)
- is_vested is derived from start_date → remove (do not store derived attributes in LDM)

After 3NF:
Employee:   employee_id, dept_code, start_date
Department: dept_code, dept_name
```

**The practical summary:**
```
1NF: each column holds one value; no repeating column groups
2NF: every non-key column depends on the entire PK (not just part of it)
3NF: every non-key column depends directly on the PK (not on another non-key column)

Memory hook:
"The key (1NF), the whole key (2NF), and nothing but the key (3NF)"
```

---

## Normalization in SQL terms

```sql
-- BEFORE normalization: chaos in one table
CREATE TABLE student_chaos (
    student_id       INT,
    dept_code        CHAR(2),
    phone_1          VARCHAR(20),  -- repeating groups
    phone_2          VARCHAR(20),
    phone_3          VARCHAR(20),
    student_name     VARCHAR(100), -- multi-valued
    dept_name        VARCHAR(100), -- transitive dependency on dept_code
    start_date       DATE,
    is_vested        BIT           -- derived attribute
);

-- AFTER 3NF
CREATE TABLE student (
    student_id          INT PRIMARY KEY,
    dept_code           CHAR(2) NOT NULL REFERENCES department(dept_code),
    student_first_name  VARCHAR(50) NOT NULL,
    student_last_name   VARCHAR(50) NOT NULL,
    start_date          DATE NOT NULL
);

CREATE TABLE department (
    dept_code   CHAR(2) PRIMARY KEY,
    dept_name   VARCHAR(100) NOT NULL,
    dept_phone  VARCHAR(20)
);

CREATE TABLE student_phone (
    student_id       INT REFERENCES student(student_id),
    phone_number     VARCHAR(20),
    phone_type_code  CHAR(10) CHECK (phone_type_code IN ('MOBILE','WORK','HOME')),
    PRIMARY KEY (student_id, phone_number)
);
```

---

## Abstraction — optional, use carefully

Abstraction collapses concrete entities into more generic structures to gain flexibility.

```
Before abstraction:
Student entity  →  student_id, student_first_name, ...
Staff entity    →  staff_id, staff_first_name, ...
Agent entity    →  agent_id, agent_first_name, ...

After abstraction:
Party entity    →  party_id, first_name, last_name, ...
Party_Role      →  party_id, role_type_code ('STUDENT','STAFF','AGENT')
```

**When to use abstraction:** When you anticipate new roles or types emerging frequently.

**The three costs of abstraction — always weigh these:**

| Cost | What you lose | Example |
|------|-------------|---------|
| Communication | Concrete names disappear | "Student" is now just a role_type_code value |
| Business rules | Rules on subtypes become application code | "Student must have enrolment_date" can't be enforced at DB level |
| Development complexity | Loading and querying becomes harder | ETL must translate `role_type_code = 'STUDENT'` to know what columns apply |

**Decision rule:** If you have 3+ subtypes that are identical in structure and the list will keep growing, abstract. If subtypes have meaningfully different attributes and rules, keep them separate.

---

## Dimensional LDM terminology

The dimensional LDM uses different vocabulary from the relational LDM:

| Dimensional term | Relational equivalent | What it means |
|-----------------|----------------------|---------------|
| **Meter** (= Fact table at CDM/LDM level) | — | A bucket of related measures. Named after the business process it measures: "Enrolment Meter", "Assessment Meter" |
| **Dimension** | Reference entity | Adds context to measures. Filter, group, and sort by dimensions |
| **Snowflake** | Higher-level reference entity | A parent of a dimension. Forms a hierarchy |
| **Hierarchy** | Parent-child relationship | One parent per child at most. Year → Quarter → Month → Day |

**Four types of meters (fact table types):**

| Type | What it captures | Example |
|------|-----------------|---------|
| **Aggregate** | Pre-summarised measures | Monthly enrolment count by campus |
| **Atomic** | Lowest grain available | Individual enrolment event row |
| **Cumulative** | Time-to-complete a process | Days from application to enrolment decision |
| **Snapshot** | State at a specific point in time | Student headcount at end of each semester |

**Six types of dimensions:**

| Type | What it is | Example |
|------|-----------|---------|
| **Fixed** (SCD Type 0) | Never changes | Country Code |
| **Degenerate** | No attributes worth a separate table; moved to fact | Order Number stored in fact_order |
| **Multi-valued** | One fact row needs multiple dimension values | A student may have multiple diagnosed learning needs |
| **Ragged** | Hierarchy with variable depth | Org chart where some branches are 3 levels, others 5 |
| **Shrunken** | Subset of the fact's dimensions | A dimension for "courses with assessments" only |
| **Slowly Changing** (SCD) | Changes over time — handled by Type 0/1/2/3/6 | Student's campus changes mid-degree |

---

## Common mistakes

**Mistake 1 — Skipping 1NF because "it's obvious".**
Phone_1 / Phone_2 / Phone_3 columns appear in real-world schemas constantly. They break when a student has a 4th phone number, and they make querying by phone type impossible. Always ask the repeating group question.

**Mistake 2 — Stopping at 1NF.**
Splitting `student_name` into first/last is 1NF. Moving `dept_name` out of `Student` into `Department` is 3NF. Do both.

**Mistake 3 — Storing derived attributes in the LDM.**
`is_vested`, `age`, `days_enrolled`, `gpa` — all calculated. Do not store them in the LDM. Calculate at query time (or materialise in gold-layer views for performance).

**Mistake 4 — Using an aggregate meter when you need an atomic one.**
Building `fact_monthly_enrolment` when stakeholders also need daily analysis. Start atomic. Aggregates are additive on top.

**Mistake 5 — Conflating the snowflake (hierarchy parent) with the dimension itself.**
`Year` is a snowflake of `Semester`. It is not a separate dimension; it is a level within the time hierarchy. Do not create a separate `dim_year` fact table unless you have measures that only make sense at year level.

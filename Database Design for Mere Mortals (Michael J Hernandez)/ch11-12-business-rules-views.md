# ch11 — Business Rules

## What problem does this solve and why does it exist

The first three integrity levels (table, field, relationship) deal with structure. Business rules deal with **meaning** — the way your specific organisation perceives and uses its data. Two organisations can have identical table structures and still need completely different constraints on what values are valid or how records relate, because they operate differently.

Business rules are the final component of overall data integrity. Without them, structurally sound data can still be contextually wrong.

---

## What is a business rule?

> A statement that imposes some form of constraint on a specific aspect of the database, based on the way the organisation perceives and uses its data.

```
Examples:
"A ship date cannot be prior to an order date."
"An agent can represent no more than 20 entertainers."
"We conduct business exclusively with vendors from the Pacific Northwest."
"Every vendor must supply at least one product."
```

Each of these constraints comes from *how this organisation works* — another organisation might have no such rules, or different ones.

---

## Two types of business rules

| Type | Where established | How |
|------|------------------|-----|
| **Database-oriented** | Within the logical design | Modify field specification elements or relationship characteristics |
| **Application-oriented** | Outside the logical design — in application code or physical RDBMS implementation | Programming code, triggers, stored procedures |

**Focus during design:** Database-oriented rules only. Document application-oriented rules for use during implementation.

**Decision test:** Can you meaningfully establish this constraint by modifying a field spec element or a relationship characteristic? If yes → database-oriented. If no → application-oriented.

```
Database-oriented (can establish in design):
"We conduct business only with Pacific NW vendors."
→ Modify VendState Range of Values: WA, OR, ID, MT

Application-oriented (cannot establish in design):
"A preferred customer receives a 15% discount on all purchases."
→ No field to store the discount amount (calculated value)
→ No way to embed the "if status = Preferred" criterion in the schema
→ Must be enforced in application code
```

---

## Two categories of database-oriented business rules

### Field-specific rules

Impose constraints on field specification elements. May affect one element or several.

**Process — five steps:**
1. Select a table
2. Review each field — ask: "Based on how this table is used, is a constraint necessary for any element in this field's specification?"
3. If yes: define the business rule as a clear statement
4. Establish the rule by modifying the appropriate field spec elements
5. Determine which actions test the rule (Insert, Update, Delete) — then document on a Business Rule Specifications sheet

```
Field: CustCounty
Current Range of Values: King, Kitsap

User response: "We've just added Pierce and Snohomish counties. 
                And every customer MUST have a county recorded now."

Business rules derived:
  Rule 1: "A county must be associated with each customer."
    → Required Value: Yes
    → Null Support: No Nulls
    → Edit Rule: Enter Now, Edits Allowed

  Rule 2: "Only King, Kitsap, Pierce, and Snohomish are valid county entries."
    → Range of Values: King, Kitsap, Pierce, Snohomish

Test on: Insert, Update
```

### Relationship-specific rules

Impose constraints on relationship characteristics — deletion rule, type of participation, or degree of participation.

```
"Each class must have a minimum of five students, but cannot exceed 20."
→ Affects: degree of participation for STUDENT_CLASSES
→ Change: (0,N) → (5,20)
→ Also implies: type of participation for STUDENT_CLASSES becomes Mandatory
                (the class can only exist if at least 5 students are registered)

Test on: Insert, Delete
```

---

## Business Rule Specifications sheet

Document every business rule in a structured way. The sheet captures:
- **Statement** — the plain-English rule
- **Constraint** — what specifically is being limited
- **Type** — Database-oriented or Application-oriented
- **Category** — Field-specific or Relationship-specific
- **Test on** — Insert / Update / Delete (which DML operations could violate this rule?)
- **Structures affected** — which field names and table names
- **Field elements affected** — which elements of the field spec changed
- **Relationship characteristics affected** — Deletion Rule, Type of Participation, Degree of Participation
- **Action taken** — date, who made the change, what exactly was modified

Review all sheets after completion. Business rules change as the organisation grows — this documentation makes future modifications traceable and reversible.

---

## Validation tables — implementing field-specific rules with a lookup table

When a business rule limits a field's range of values to a finite, potentially growing list, use a **validation table** (lookup table) rather than a hard-coded list in the field spec.

**Advantages over hard-coded lists:**
- New valid values can be added without schema changes
- Values can have descriptions (display names, is_active flags)
- Values can be reused across multiple fields in multiple tables
- Enforcement is through a FK relationship — the RDBMS enforces it natively

**Two-step process:**

**Step 1 — Create the validation table and establish the relationship:**

```
Business rule: "Any supplier must be from one of 13 western states."

Create: STATES (State CHAR(2) PK, State_Name VARCHAR(50))

Relationship: STATES ——|——<○—— SUPPLIERS
  Deletion rule:        Restrict (cannot delete a state that suppliers reference)
  STATES participation: Mandatory | degree (1,1)
  SUPPLIERS participation: Optional | degree (0,N)
```

**Step 2 — Update the FK field's Range of Values element:**

```
SUPPLIERS.State Range of Values:
  "Any value within the State field of the STATES table."
```

The FK relationship now enforces the rule. Any `State` value entered into SUPPLIERS that doesn't exist in STATES will be rejected by the RDBMS.

---

## Applied to JMC Academy — business rules

```
Field-specific rules:

Rule: "Enrolment status must be one of our defined statuses."
  Table: enrolment
  Field: enrolment_status_code
  Action: Create validation table REF_ENROLMENT_STATUS
          Establish FK relationship: REF_ENROLMENT_STATUS ——|——<○—— enrolment
          Range of Values: "Any status_code in REF_ENROLMENT_STATUS"
  Test on: Insert, Update

Rule: "Tuition amount cannot be negative."
  Table: enrolment
  Field: tuition_amount
  Range of Values: >= 0.00
  Test on: Insert, Update

Rule: "Visa type must be from our approved visa category list."
  Table: student
  Field: visa_code
  Action: Create validation table REF_VISA_TYPE
          Establish FK relationship
  Test on: Insert, Update

Relationship-specific rules:

Rule: "A course must have at least one active unit to be offered."
  Relationship: course ——<—— course_unit
  Change: course_unit degree from (0,N) to (1,N) for course side
  Test on: Delete (when deleting a unit from a course)

Rule: "An assessment attempt must belong to an active enrolment."
  Relationship: enrolment ——<—— assessment_attempt
  Type of participation: enrolment = Mandatory
  Test on: Insert (cannot create an attempt without a valid enrolment)
```

---

# ch12 — Views

## What problem does this solve and why does it exist

Tables store atomic data. Users need aggregated, filtered, and combined information. Views are the bridge: they allow you to present data from one or more tables in exactly the form users need, without duplicating or restructuring the data.

---

## What is a view?

> A virtual table that draws data from base tables rather than storing data itself.

The RDBMS rebuilds and repopulates the view every time you access it, using the most current data from base tables. The view's structure is stored — the data is not.

**Five reasons to use views:**
1. Work with data from multiple related tables simultaneously
2. Always reflect the most current information (rebuilt on every access)
3. Customise for specific users or groups (restrict which fields they see)
4. Enforce data integrity (validation views)
5. Security — restrict access to sensitive fields while exposing safe ones

---

## Three types of views

### Data view — examine and work with data

**Single-table data view:** Selected fields from one base table. Data can usually be modified (edits flow through to the base table, subject to field specs).

```
EMPLOYEE_PHONE_LIST view:
Base table: Employees
Fields:     Employee_ID, EmpFirst_Name, EmpLast_Name, EmpPhone_Number

Purpose:    Make a directory available to all staff without exposing
            salary, address, or other sensitive fields.
```

**Multitable data view:** Fields from two or more related tables. Tables in the view MUST be related to each other — this guarantees the information is valid and meaningful.

```
CLASS_ROSTER view:
Base tables: Students, Student_Classes (linking), Classes
Fields:      Class_Name (from Classes), StudFirst_Name, StudLast_Name (from Students)

Result: Every class paired with every enrolled student.
        The RDBMS uses the relationships to produce valid combinations.
```

Note: multitable views often display apparent redundancy (Class_Name repeating for each enrolled student). This is acceptable — the data is drawn from base tables where it is stored correctly. The view does not physically duplicate anything.

Data views do **not** have their own primary key. They are virtual tables, not real tables.

### Aggregate view — summarised information

Uses aggregate functions to compute summary data from base tables. Can use one or more base tables.

**Common aggregate functions:**
- `Count()` — total number of occurrences
- `Sum()` — total of a numeric field
- `Avg()` — arithmetic mean
- `Min()` / `Max()` — smallest / largest value

**Characteristics:**
- Contains **calculated fields** (the aggregate expressions)
- Contains **grouping fields** (the data fields that define the groups)
- Grouping fields cannot be modified
- **You cannot modify any data in an aggregate view** — it is entirely composed of computed and grouped results

```
CLASS_REGISTRATION view:
Base tables: Classes, Student_Classes
Fields:      Class_Name (grouping field), Total_Students_Registered (= Count(Student_ID))

Purpose:     "How many students are registered for each class?"
             Without this view, you'd have to manually count rows in Student_Classes.
```

### Validation view — enforce data integrity with a subset of a table

Works like a validation table, but instead of creating a separate lookup table, you create a view over an existing table that restricts which fields (and records) are visible.

```
Scenario: PROJECT_SUBCONTRACTORS.Subcontractor_ID must reference a valid subcontractor,
          but users should only see four of the subcontractor fields (not the full table).

APPROVED_SUBCONTRACTORS validation view:
Base table: Subcontractors
Fields:     Subcontractor_ID, SCName, SCPhone_Number, SCEmail

→ Users can only reference subcontractors through this restricted view.
→ The full SUBCONTRACTORS table (with address, financial details) is not directly accessible.
→ Relationship characteristics from SUBCONTRACTORS carry through the view.
```

Advantages of validation views over validation tables:
- No separate data to maintain
- Can draw from multiple tables
- Can apply filtering criteria
- Inherits relationship constraints from base tables

---

## Defining views — the process

**Sources for identifying needed views:**
1. Review your design notes for mentioned reports
2. Review data collection and information presentation samples from Ch06
3. Examine tables and the subjects they represent — certain combinations are obvious
4. Analyse table relationships — every 1:M and M:M relationship is a candidate
5. Review business rules — validation views enforce field-specific constraints

**For each view, determine:**
1. What information must it provide?
2. Which base tables contain the necessary fields?
3. Which fields from those tables are needed?
4. Are calculated fields required?
5. Does the view need filters (criteria)?

---

## Calculated fields in views

Views can contain calculated fields. Tables cannot. This is where your Calculated Field List (from Ch06) gets used.

```
CUSTOMER_CALL_LIST view — calculated fields:
  Customer_Name = CustLast_Name & ", " & CustFirst_Name
  Last_Purchase_Date = Max(Order_Date)

These replace CustFirst_Name, CustLast_Name, and Order_Date in the view structure.
```

**When to use calculated fields in views:**
- When they provide information that users need but cannot be stored in a table
- When they aggregate or combine values meaningfully
- When they format data for presentation (concatenated full names, formatted dates)

---

## Filtering (criteria)

Apply criteria to a view to restrict the records it displays. The filter is applied to a specific field — that field must be included in the view's structure.

```
PREFERRED_CUSTOMERS view:
Base table: Customers
Fields:     Customer_ID, Customer_Name (calculated), CustHome_Phone, Status
Filter:     Status = "Preferred"

→ Only records where Status = "Preferred" appear.
```

Use the minimum number of criteria that reliably return the required records. Be aware that the same city name exists in multiple states — if filtering by city, also filter by state.

---

## View Specifications sheet

Every view gets a View Specifications sheet (accompanies the view diagram):
- **Name** — follows table naming guidelines (can implicitly identify more than one subject)
- **Type** — Data, Aggregate, or Validation
- **Description** — purpose of the view, information it provides
- **Base tables** — list all tables the view draws from
- **Calculated field expressions** — field name + expression for each calculated field
- **Filters** — field name + condition for each filter criterion

---

## Applied to JMC Academy — view examples

```
Data view: STUDENT_ENROLMENT_DETAIL
Base tables: student, enrolment, unit, course, campus, semester
Fields:      student_id, student_first_name, student_last_name, student_email,
             course_code, course_name, unit_code, unit_name,
             semester_code, academic_year, campus_name, delivery_mode_name,
             enrolment_status_code, enrolment_date
Purpose:     Core operational view for Student Services staff. Provides
             all enrolment details without accessing raw tables separately.

Aggregate view: UNIT_PASS_RATE_SUMMARY
Base tables: assessment_attempt, enrolment, unit, campus, semester
Fields:      unit_code, unit_name (grouping),
             campus_name (grouping), semester_code (grouping),
             total_attempts   = Count(attempt_id),
             pass_count       = Sum(CASE WHEN is_pass = 1 THEN 1 ELSE 0 END),
             pass_rate_pct    = (pass_count / total_attempts) × 100
Purpose:     Academic quality reporting. Enables drill-down by unit/campus/semester.

Aggregate view: APPLICATION_FUNNEL
Base tables: application, student, course
Fields:      course_name (grouping), semester_name (grouping),
             total_applications = Count(application_id),
             offers_made        = Sum(is_offer_made),
             offers_accepted    = Sum(is_offer_accepted)
Purpose:     Admissions conversion tracking from application to enrolment.

Validation view: ACTIVE_COURSES
Base table: course
Fields:     course_id, course_code, course_name, qualification_level
Filter:     is_active_flag = 1
Purpose:    When creating new applications or enrolments, only active courses
            should be selectable. This view provides that restricted list.
```

---

## Common mistakes — business rules and views

**Business rules:**

**Mistake 1 — Treating all rules as application-oriented.**
"We'll handle that in code" is a common response. But if the constraint can be expressed through field specs or relationship characteristics, it belongs in the design — not in code that may be inconsistently applied.

**Mistake 2 — Not documenting application-oriented rules.**
If a rule can't be established in the logical design, at least document it on the Business Rule Specifications sheet so implementation doesn't forget it.

**Mistake 3 — Not testing rules against Insert, Update, AND Delete.**
Many constraints are violated on delete (orphaned records) as much as on insert (invalid values). Check all three operations.

**Views:**

**Mistake 4 — Using views to replace proper table design.**
A view cannot fix a poorly designed table. If your base tables have structural problems, views will surface them, not hide them.

**Mistake 5 — Building a view on unrelated tables.**
Every table in a multitable view must have a relationship to at least one other table in the view. Otherwise the result is a Cartesian product — every record in A paired with every record in B, producing meaningless data.

**Mistake 6 — Forgetting the View Specifications sheet.**
The diagram shows the structure. The spec sheet shows the expressions and filters. Without the spec sheet, the implementation team cannot build the view correctly.

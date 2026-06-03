# ch09 — Field Specifications

## What problem does this solve and why does it exist

Tables and keys tell you *what* data you have and *where* it lives. Field specifications tell you *everything* about each individual field: what type of value it holds, how long it can be, whether it can be empty, what values are valid, who enters it, and when it can be changed. Without field specifications, you have table structures with no rules — any value can go anywhere, and data quality is impossible to guarantee.

> Field-level integrity is what makes the information you retrieve accurate, consistent, and meaningful.

Hernandez is direct: people who don't have time to define field specs spend three times as long fixing their databases afterward. This step is not optional.

---

## Three categories of elements in a field specification

Every field gets a complete specification covering three categories. Together they constitute the "data dictionary" of your database.

---

## General Elements

These describe the identity and context of the field.

### Field Name
The unique, descriptive name you created and refined in Ch07. Use it here as-is.

### Parent Table
The table the field primarily belongs to. The only other tables it should appear in are those with a relationship to this one (via foreign keys).

### Specification Type — the most important general element

| Type | When to use | Key behaviour |
|------|-------------|---------------|
| **Unique** | Default for all fields. Applied when the field appears only once in the database, or when it is a primary key. | All elements defined specifically for this field only. |
| **Generic** | When this field serves as a **template** for other fields of the same general type. Uses a non-specific name (e.g., "State" not "CustState"). Elements as broad as possible. | No Parent Table, no Shared By, no Aliases, no Source Specification. |
| **Replica** | When this field is **based on** a Generic spec, or when it is a **foreign key**. Inherits element settings from the source spec; any can be overridden. | Must specify Source Specification. Used for all FK field specs. |

**The Generic/Replica pattern in practice:**

```
Generic spec: "State"
  Data Type:        Alphanumeric
  Length:           2
  Character Support: Letters only
  Null Support:     No Nulls
  Required Value:   Yes
  Range of Values:  All US state abbreviations

Replica spec: "VendState" (based on "State")
  Source Spec:      State
  Parent Table:     Vendors
  Description:      The state where the vendor's headquarters are located.
  Range of Values:  CA, ID, MT, OR, WA  ← narrowed by business rule
```

Create Generic specs for any field type that will appear in multiple tables: State, Zipcode, Phone Number, Email Address, Date fields, Currency Amount fields.

### Source Specification
Set only on Replica specs. Names the Generic (or primary key) spec this field is based on. Include the source table name: "Employee_ID from the Employees table."

### Shared By
Lists all other tables that share this field (because it participates in a relationship). If Employee_ID appears in the Employees table and in the Orders table as a FK, the Employees Unique spec lists "Orders" here.

### Alias(es)
A different name used for a field in specific circumstances — almost always when two occurrences of the same field must appear in one table.

```
SUBSIDIARIES table needs two employee references:
  President_ID    (alias for Employee_ID)
  Vice_President_ID (alias for Employee_ID)

Both are logically the same field (Employee_ID), but need different names in this table.
```

Use aliases sparingly. They can obscure the true meaning of data if overused.

### Description
A complete interpretation of the field. Not a restatement of the name. Should state what it represents AND why it matters to the organisation.

**Guidelines for descriptions:**
- Accurately identifies and clearly states purpose
- Clear and succinct — no verbose or ambiguous sentences
- Does NOT restate or rephrase the field name
- No technical jargon, acronyms, or abbreviations
- No implementation-specific info (what screen it appears on, what code uses it)
- Not dependent on another field's description
- No examples

```
Bad:  CustLast_Name — the last name of a customer.
                      (just restates the name)

Good: CustLast_Name — the surname of a customer, whether original or by marriage,
                      that we use in all formal communications and correspondence.
```

---

## Physical Elements

These describe the structure of the field as it will be stored.

### Data Type
Three general types (your RDBMS will have more specific subtypes):

| Type | Stores | Notes |
|------|--------|-------|
| **Alphanumeric** | Any combination of letters, numbers, keyboard characters, special characters | Use when the value contains letters, mixed content, or codes with leading zeros |
| **Numeric** | Whole numbers and real numbers only | Will not preserve leading zeros. Use for values that will be calculated. |
| **DateTime** | Dates, times, or both | Use for any temporal value |

### Length
Total number of characters a user can enter. Set conservatively — too short causes data loss, too long wastes storage. For numeric fields, many RDBMSs set length automatically based on subtype.

### Decimal Places
For real numbers: number of digits to the right of the decimal point. Currency typically needs 2; scientific data may need 4+.

### Character Support
Which types of characters are valid in this field's values. Enforcing this prevents meaningless data entry.

| Character type | Examples |
|----------------|---------|
| Letters (A–Z) | Alphabetic characters including accented letters (é, ñ) |
| Numbers (0–9) | Digits only |
| Keyboard ( . , / $ # % ) | Standard punctuation and symbols |
| Special ( © ® ™ Σ π ) | Characters requiring special key combinations |

```
CustState:
  Data Type:        Alphanumeric
  Length:           2
  Character Support: Letters only
  → Cannot enter "WA1" or "W A" — only two letters
```

---

## Logical Elements

These govern what values go into the field and who can change them. This is where most data integrity is enforced.

### Key Type
`Non` / `Primary` / `Alternate` / `Foreign` — reflects the key designation from Ch08.

### Key Structure
`Simple` (single-field key) or `Composite` (multi-field key).

### Uniqueness
- Set to `Unique` when Key Type is Primary, or when a non-key field requires unique values (e.g., a manager can only manage one department at a time)
- Set to `Non-Unique` for most non-key fields and for foreign keys (a FK value can and should repeat — that is how relationships work)

### Null Support
- `No Nulls` — field must always have a value (primary keys, required fields, foreign keys in mandatory relationships)
- `Nulls Allowed` — value may be missing or unknown

**Decision rule:** Set No Nulls unless there is a genuine, documented reason a value might not exist. Do not use Null as a default.

### Values Entered By
- `User` — a person enters the value
- `System` — a database application or RDBMS generates the value automatically (e.g., auto-increment ID, system timestamp)

### Required Value
- `Yes` — user must enter a value when creating a new record
- `No` — value is optional at record creation (but may still be required later, via Edit Rule)

Primary keys: always `Yes`. Foreign keys in mandatory relationships: `Yes`.

### Range of Values
Specifies every possible valid value for the field. Three categories:

| Category | What it defines | When you set it |
|----------|----------------|----------------|
| **General** | The complete set of all possible valid values for the field | During this phase (Ch09) |
| **Integrity-specific** | Values constrained by a relationship (FK values must match existing PK values) | During Ch10 (Relationships) |
| **Business-specific** | Values constrained by how the organisation uses the data | During Ch11 (Business Rules) |

```
General:    All valid state abbreviations (AL, AK, AZ, ...)
Integrity:  Any existing Employee_ID value in the EMPLOYEES table
Business:   Only WA, OR, ID, MT — because we only work with Pacific NW vendors
```

**Never use "Other" or "Miscellaneous" as a Range of Values entry.** These are meaningless and signal that the field needs further analysis.

### Edit Rule
When a user can enter a value and whether they can modify it:

| Rule | Meaning |
|------|---------|
| `Enter Now, Edits Allowed` | Must enter at record creation; can modify anytime |
| `Enter Now, Edits Not Allowed` | Must enter at creation; cannot change afterward |
| `Enter Later, Edits Allowed` | Optional at creation; must enter at some future point; can then edit |
| `Enter Later, Edits Not Allowed` | Optional at creation; must enter eventually; once entered, locked |
| `Not Determined At This Time` | Defer to implementation phase |

Primary keys: `Enter Now, Edits Not Allowed` (once assigned, should not change).

Foreign keys: `Enter Now, Edits Allowed` (allow correction of mistakes — wrong employee assigned to an order).

---

## Standard settings for primary key fields

Any field serving as a PK should have:
```
Key Type:         Primary
Key Structure:    Simple (or Composite if CPK)
Uniqueness:       Unique
Null Support:     No Nulls
Values Entered By: System (if auto-generated) or User
Required Value:   Yes
Edit Rule:        Enter Now, Edits Not Allowed
```

---

## How to define field specifications efficiently

**Strategy:** Define as many as you can independently first, then review with users and management to complete the rest.

Steps:
1. Define specs for all fields you understand well
2. Identify fields where you are unsure about Range of Values or Null Support
3. Meet with users and management; walk through all specs
4. For each field, ask: are the element settings suitable and correct?
5. Participants often reveal constraints you had not considered — update accordingly
6. When consensus is reached, the field specification is complete

---

## Applied to JMC Academy — sample field specifications

```
Field: student_id
  Parent Table:      student
  Spec Type:         Unique
  Description:       The unique identifier assigned to a student by Paradigm SIS.
                     Remains with the student throughout their relationship with JMC.
  Data Type:         Alphanumeric
  Length:            20
  Character Support: Letters, Numbers
  Key Type:          Primary
  Key Structure:     Simple
  Uniqueness:        Unique
  Null Support:      No Nulls
  Values Entered By: System (migrated from Paradigm)
  Required Value:    Yes
  Range of Values:   Any existing Paradigm student identifier
  Edit Rule:         Enter Now, Edits Not Allowed

Field: enrolment_status_code
  Parent Table:      enrolment
  Spec Type:         Unique
  Description:       The current status of a student's enrolment in a unit.
                     Reflects the most recent administrative or academic action.
  Data Type:         Alphanumeric
  Length:            20
  Character Support: Letters
  Key Type:          Non
  Uniqueness:        Non-Unique
  Null Support:      No Nulls
  Values Entered By: System (set by Paradigm, migrated)
  Required Value:    Yes
  Range of Values:   Active, Withdrawn, Deferred, Completed, Excluded, LOA
  Edit Rule:         Enter Now, Edits Allowed

Field: tuition_amount
  Parent Table:      enrolment
  Spec Type:         Generic (template for all currency amounts)
  Description:       A currency amount expressed in Australian dollars.
  Data Type:         Numeric
  Length:            15
  Decimal Places:    2
  Character Support: Numbers
  Key Type:          Non
  Uniqueness:        Non-Unique
  Null Support:      No Nulls
  Values Entered By: System
  Required Value:    Yes
  Range of Values:   >= 0
  Edit Rule:         Enter Now, Edits Allowed
```

---

## Common mistakes

**Mistake 1 — Skipping field specs entirely.**
The most common. Engineers jump straight to implementation and rely on application code to enforce constraints. Field specs make your constraints explicit, portable, and implementation-independent.

**Mistake 2 — Setting Range of Values to "Any valid value" without specifying what valid means.**
This is not a range. A range must be precise: either a list, a min/max boundary, or a reference to another field's values.

**Mistake 3 — Setting Null Support to "Nulls Allowed" by default.**
Default should be No Nulls. Use Nulls Allowed only when there is a documented, genuine case where a value may truly not exist.

**Mistake 4 — Not distinguishing Generic from Unique specs.**
If you have 12 State fields across 8 tables and each has slightly different element settings because you didn't use a Generic template, you have 12 maintenance points instead of one.

**Mistake 5 — Setting Edit Rule to "Enter Now, Edits Not Allowed" for foreign keys.**
You need to be able to correct mistakes. If an order was assigned to the wrong employee, you must be able to fix the FK. Always `Enter Now, Edits Allowed` for FKs.

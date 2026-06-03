# ch08 — Keys

## What problem does this solve and why does it exist

Without a reliable way to uniquely identify every record in every table, you cannot:
- Retrieve a specific record with certainty
- Establish relationships between tables
- Enforce that duplicate records don't exist
- Guarantee data integrity at any level

Keys are the mechanism that makes all of this possible. Every type of key has a distinct purpose. Getting them right early prevents structural problems that are expensive to fix later.

---

## Why keys are important — the three functions

1. **Precise identification** — a key ensures every record in a table can be identified and distinguished from all others
2. **Integrity enforcement** — keys are the foundation of both table-level integrity and relationship-level integrity
3. **Relationship establishment** — you use keys to establish logical connections between tables (covered in Ch10)

---

## The four types of keys

```
Candidate Key        ← eligible to be PK; found or created
    ├── Primary Key  ← the chosen candidate key
    └── Alternate Key← remaining candidate keys (not chosen)
Foreign Key          ← copy of a PK from another table; establishes relationships
Non-key              ← every other field; its value is determined by the PK
```

---

## Candidate keys — Elements of a Candidate Key

A candidate key is a field or set of fields that uniquely identifies a single instance of the table's subject. Every table must have at least one.

**A field cannot serve as a candidate key if it fails even one of these elements:**

| Element | Explanation |
|---------|-------------|
| Cannot be a multipart field | Multipart fields cause retrieval problems (already resolved in Ch07) |
| Must contain unique values | Duplicate values mean you cannot reliably identify a single record |
| Cannot contain Nulls | A Null cannot identify a record — it represents absence of a value |
| Its value cannot breach security or privacy rules | Do not use SSN, passport number, or passwords as keys |
| Its value is not optional in whole or in part | Optional implies possible Null — see above |
| Comprises minimum fields necessary for uniqueness | If 4 fields are listed but only 3 are needed, use only 3 (composite CK rule) |
| Values uniquely and exclusively identify each record | Ensure no two records could have the same key value |
| Value exclusively identifies the value of each field in a given record | The PK test — more on this below |
| Its value can be modified only in rare or extreme cases | Unstable keys create cascading integrity problems |

### How to identify candidate keys in practice

Load the table with sample data (on paper is fine — 5–7 rows). Try each field as a candidate. Any field that could repeat, be Null, be optional, be a privacy concern, or be multipart is disqualified immediately.

```
EMPLOYEES table sample:
Employee_ID  |  SSN         |  First_Name  |  Last_Name  |  Home_Phone     |  Zipcode
1000         |  987-65-9938 |  Kira        |  Bently     |  221 363-9948   |  98157
1001         |  987-65-6531 |  Katherine   |  Erlich     |  235 322-6992   |  98046
1002         |  987-65-0039 |  Bill        |  Champlin   |  235 527-4992   |  98115
1003         |  987-65-1299 |  Shannon     |  Black      |  221 336-5992   |  98136
1004         |  987-65-6529 |  Susan       |  Black      |  230 572-9948   |  98115
```

Evaluation:
- Employee_ID → eligible (unique, not null, not optional, not multipart) ✓
- SSN → INELIGIBLE (could be Null for non-US citizens; privacy violation)
- Last_Name → INELIGIBLE (duplicate: "Black" appears twice)
- First_Name + Last_Name → INELIGIBLE (could produce duplicates as org grows)
- Home_Phone → INELIGIBLE (could be Null; could duplicate — family members at same address)
- Zipcode → INELIGIBLE (many employees share a zipcode)

Result: Employee_ID is the only candidate key.
```

A composite candidate key (CCK) uses two or more fields together. Mark with "CCK" in your table structure — if you have two CCKs, use CCK1 and CCK2.

---

## Artificial candidate keys — when to create one

When no field or combination of fields qualifies as a candidate key, create an artificial (surrogate) candidate key. This is a new field you manufacture specifically to serve as the identifier.

**Create an artificial CK when:**
- No natural candidate key qualifies (table has no reliable unique identifier from real data)
- A composite CCK would be complex (3+ fields) and another CK would be simpler
- Using the natural key would expose sensitive data as a FK throughout the database

```
PARTS table — no field or combination qualifies:
Part_Name:      duplicate (Faust Brake Levers appears twice with different model numbers)
Model_Number:   can be Null (some parts have no model number)
Part+Model:     disqualified because Model_Number can be Null

Solution: Create Part_Number as an artificial CK
Part_Number | Part_Name            | Model_Number | Manufacturer
41000       | Shimka XT Cranks     | XT-113       | Shimka Inc.
41001       | Faust Brake Levers   | BL/45        | Faust USA
41002       | MiniMite Pump        | MiniMite     | ...
```

**Best practice:** Use an ID field (`Part_Number`, `Customer_ID`, `Enrolment_ID`) as the artificial CK — it always qualifies, identifies the table's subject in its name, and simplifies relationship establishment.

---

## Primary key — selecting from candidate keys

The primary key is the candidate key chosen to officially identify the table. It is the most important key in the table.

**Two selection guidelines when you have multiple candidate keys:**
1. Prefer a simple (single-field) CK over a composite CCK
2. Choose the CK whose name incorporates the table name (e.g., `Customer_ID` for the CUSTOMERS table)

**The critical test — exclusively identify the value of each field:**

Before finalising the primary key, test it against every other field in the table:

> Does this primary key value **exclusively** identify the current value of `<fieldname>`?

If the answer is **no** for any field, that field does not belong in this table. Remove it.

```
SALES_INVOICES table, testing Invoice_Number as PK:

Invoice_Number  Invoice_Date  CustFirst  CustLast  EmpFirst  EmpLast  EmpHome_Phone

For record 13002:
- Does Invoice_Number 13002 exclusively identify Invoice_Date?     YES ✓
- Does Invoice_Number 13002 exclusively identify CustFirst_Name?   YES ✓
- Does Invoice_Number 13002 exclusively identify CustLast_Name?    YES ✓
- Does Invoice_Number 13002 exclusively identify EmpFirst_Name?    YES ✓
- Does Invoice_Number 13002 exclusively identify EmpLast_Name?     YES ✓
- Does Invoice_Number 13002 exclusively identify EmpHome_Phone?    NO ✗

Why NO? EmpHome_Phone is actually identified by EmpFirst+EmpLast, not by the Invoice_Number.
If you change the employee name, you must also change the phone number.
The PK does not exclusively identify it — EmpFirst+EmpLast do.

Action: REMOVE EmpHome_Phone from SALES_INVOICES.
         (It already exists in the EMPLOYEES table where it belongs.)
```

This test is where normalization's concept of "functional dependency" is embedded. If the PK does not exclusively identify a field's value, the field is in the wrong table.

**Two rules for primary keys:**
1. Each table must have **one and only one** primary key
2. Each primary key within the database must be **unique** — no two tables share the same PK unless they have a 1:1 relationship or one is a subset table

Mark primary keys with **PK** in your table structure. Composite primary keys are marked **CPK** (and each field in the composite gets the CPK label).

---

## Alternate keys

After selecting a PK, all remaining candidate keys become **alternate keys**. They are not less important — they still uniquely identify records. Mark them **AK** (or **CAK** for composite alternate keys).

Alternate keys are useful in your RDBMS as unique constraints and unique indexes, giving users a second reliable way to look up records.

```
EMPLOYEES table final key structure:
Employee_ID     PK       ← chosen primary key
SSN             AK       ← disqualified from PK (privacy), kept as AK
EmpFirst_Name   CAK      ← composite alternate key part 1
EmpLast_Name    CAK      ← composite alternate key part 2
...other fields (non-keys)
```

---

## Non-keys

A non-key is any field that does not serve as a candidate, primary, alternate, or foreign key. Its sole purpose: represent a characteristic of the table's subject. Its value is determined by the primary key. No special marking needed.

---

## Table-level integrity — what it guarantees

Table-level integrity is established when you correctly assign a primary key to every table and enforce the Elements of a Primary Key.

**Table-level integrity ensures:**
- No duplicate records in the table
- The primary key exclusively identifies each record
- Every primary key value is unique
- Primary key values are never Null

Without table-level integrity, everything downstream — relationship integrity, business rules, views — is built on sand.

---

## Applied to JMC Academy

```
Key decisions for major tables:

student:
  student_id (VARCHAR 20) — CK from Paradigm SIS
  → No natural composite works; student_id is stable, unique, meaningful
  → PK: student_id
  → AK: student_email (unique constraint)

course:
  course_code (VARCHAR 20) — e.g., "BSBA701", unique, stable
  → PK: course_code directly (meaningful, no surrogate needed)
  → AK: course_name (should be unique)

enrolment:
  enrolment_id (VARCHAR 50) — from Paradigm SIS
  → PK: enrolment_id
  → No natural composite would be simpler; Paradigm's key is reliable
  → AK: (student_id, unit_id, semester_id) — enforce unique grain

assessment_attempt:
  attempt_id (VARCHAR 50) — from Canvas LMS
  → PK: attempt_id
  → AK: (student_id, assignment_id, attempt_number)

application:
  application_id (VARCHAR 50) — from HubSpot/Dynamics 365
  → PK: application_id

Test for ENROLMENT table — does enrolment_id exclusively identify each field?
- enrolment_id → student_id?           YES (this enrolment is for one student)
- enrolment_id → unit_id?              YES (this enrolment covers one unit)
- enrolment_id → enrolment_date?       YES ✓
- enrolment_id → enrolment_status?     YES ✓
- enrolment_id → tuition_amount?       YES ✓
- enrolment_id → student_email?        NO ✗
  (student_email is identified by student_id, not enrolment_id)
  → REMOVE student_email from enrolment table — it belongs in student table
```

---

## Common mistakes

**Mistake 1 — Using SSN as a primary key.**
SSN can be Null (non-US employees), violates privacy rules when it propagates as a FK across the database, and may change in edge cases. Never use SSN as a PK.

**Mistake 2 — Not testing the PK against every field.**
Skipping the "exclusively identifies" test means hidden dependencies survive into the final design, violating 3NF without knowing it.

**Mistake 3 — Using a composite CCK when a surrogate would be simpler.**
A 4-field composite PK becomes a 4-column FK in every related table. Create an artificial single-field key instead.

**Mistake 4 — Forgetting to create alternate keys for disqualified candidate keys.**
The SSN that you can't use as a PK is still unique. Enforce it as an AK (unique constraint). It will help users locate records reliably.

**Mistake 5 — Assuming an existing system's key is valid.**
Legacy systems often have "IDs" that are actually multipart (encode department + sequence), can be Null, or have duplicates due to bad data entry. Test every claimed key against the Elements of a Candidate Key.

# ch07 — Establishing Table Structures

## What problem does this solve and why does it exist

Raw field lists and subject lists are unorganised. You know *what* data exists but not *where* it belongs or *what structure* will hold it cleanly. This chapter answers: which tables should exist, what belongs in each one, and what the rules are for fields and tables being "ideal." This is the longest and most detailed design phase — it is also where most design errors either happen or are prevented.

---

## The Preliminary Table List — three sources

You build the table list from three separate sources and merge them. Using all three gives you cross-checks that reduce the chance of missing tables.

### Source 1 — Preliminary Field List

Review the field list and ask: "What subject does this group of fields describe?" Fields tend to cluster around subjects naturally.

```
Field group:        course_code, course_name, credit_points, qualification_level
Implied subject:    Course

Field group:        student_id, first_name, last_name, email, date_of_birth
Implied subject:    Student

Field group:        enrolment_id, enrolment_date, enrolment_status, tuition_amount
Implied subject:    Enrolment
```

### Source 2 — Subjects list (from interviews)

Merge the subjects list from interviews with the table list from Source 1. Three-step merge:

1. **Resolve duplicates** — if the same name appears in both lists, check whether they represent the same subject. If yes, keep one. If they represent different subjects (same name, different meaning), rename both to disambiguate.
2. **Resolve synonyms** — "Clients" and "Customers" may represent the same subject with different names. Pick the name most meaningful to the organisation.
3. **Combine remaining items** — add anything still on the subject list that does not yet appear on the table list.

### Source 3 — Mission objectives

Apply the Subject-Identification Technique to each mission objective. Cross-check every noun against the current Preliminary Table List. Add any genuinely new subjects not already represented.

```
Mission objective:  "Maintain complete agent information."
Subject identified: Agent
Cross-check:        Not yet in table list → add Agent table
```

---

## The Final Table List — four table types

Once the Preliminary Table List is complete, transform it into a **Final Table List** by adding:
- **Table type** (one of four)
- **Table description** (a clear definition of the subject and why it matters)
- **Refined table name** (following naming guidelines)

### The four table types

| Type | Description | Example |
|------|-------------|---------|
| **Data** | Represents a subject important to the organisation. Primary information source. | Student, Course, Enrolment |
| **Linking** | Establishes the connection in a many-to-many relationship | Student_Classes (resolves Student ↔ Class M:M) |
| **Subset** | Contains fields specific to a subordinate subject of a data table | Equipment (subset of Inventory), Full_Time_Employees (subset of Employees) |
| **Validation** | Stores static reference data used to enforce data integrity | Enrolment_Status, Visa_Type, Delivery_Mode |

### Guidelines for table names

- **Unique and descriptive** — self-explanatory without needing the description
- **Unambiguous** — "Dates" is vague; "Hearing_Schedule" is not
- **Minimum necessary words** — not too short ("TD1"), not too long ("Multiuse Vehicle Maintenance Equipment")
- **No physical terms** — avoid "file", "record", "table" in the name (adds confusion)
- **Represents a single subject** — if the name feels vague or covers multiple things, the table probably represents multiple subjects

### Guidelines for table descriptions

- Explicitly defines the subject the table represents
- States why the subject is important to the organisation
- Not dependent on any other table's description
- No examples, no technical jargon
- Written in plain English, readable by anyone in the organisation

---

## Elements of the Ideal Field — the six-element test

Every field must satisfy all six. These elements collectively absorb what normalization calls "scalar values", "functional dependencies", and "domain integrity".

**Element 1 — Represents a distinct characteristic of the subject.**
Each field should describe one aspect of the table's subject, and only that subject.

**Element 2 — Contains only a single value.**
No multivalued fields. A field named `phone_numbers` containing "555-1234, 555-5678" violates this. One value per field per record.

**Element 3 — Cannot be deconstructed into smaller components.**
No multipart fields. `customer_name` containing "John Smith" violates this — it should be `customer_first_name` and `customer_last_name`. `street_address` containing "7403 Kingman Drive, Redmond, WA 98115" should be four fields.

**Element 4 — Does not contain a calculated or concatenated value.**
No calculated fields in tables. `order_total`, `full_name` (concatenated), `is_vested` (derived from hire date) — all belong in the Calculated Field List, not in the table.

**Element 5 — Unique within the entire database structure.**
No field name should appear more than once, except when a field is used to establish a relationship between tables (foreign keys) or when a field appears in a subset table.

**Element 6 — Retains a majority of its characteristics when it appears in more than one table.**
When a field appears in two or more tables (because it establishes a relationship), it should carry the same name, data type, and properties in each occurrence.

### Field naming guidelines

- Unique and descriptive
- Accurately identifies the characteristic
- Minimum necessary words
- No acronyms (except widely understood ones like ID, SSN)
- No words that confuse the field's meaning
- Does not implicitly or explicitly identify more than one characteristic
- Singular form

---

## Resolving multipart fields

A multipart field contains two or more distinct items in one value. You cannot sort, filter, or extract individual components without parsing — which is always fragile and slow.

**Decision rule:** Ask "What specific items does this field's value represent?" Transform each item into its own field.

```
Before:
InstName = "Kira Bently"          → multipart (first + last name)
InstAddress = "3131 Mockingbird Lane, Seattle, WA 98157"   → multipart

After:
inst_first_name   = "Kira"
inst_last_name    = "Bently"
inst_street       = "3131 Mockingbird Lane"
inst_city         = "Seattle"
inst_state        = "WA"
inst_zipcode      = "98157"
```

**Hidden multipart fields:** When a coded field embeds multiple concepts in one value, that is also multipart.

```
instrument_id = "GUIT2201"
         ↑         ↑
     category   ID number
```
Solution: split into `instrument_category_code = "GUIT"` and `instrument_number = 2201`.

---

## Resolving multivalued fields — the three-step procedure

A multivalued field stores multiple occurrences of the same type of value (e.g., "DTP, SS, WP" in `categories_taught`).

**Why you can't "flatten" it:** Creating `category_1`, `category_2`, `category_3` columns is wrong. It limits the count to a fixed maximum, makes queries that search across all three tedious, and cannot be sorted meaningfully.

**Why you can't just make it single-value:** One category per record forces duplicating all other instructor fields for each category — massive redundant data.

**The correct three-step solution:**

1. **Remove the multivalued field** from the original table. If needed, rename it (singular form).
2. **Create a new table** using the removed field. Include a field (or fields) from the original table to relate the two tables — this becomes the FK in the new table.
3. **Name, type, and describe** the new table and add it to the Final Table List.

```
Before:
Instructors(inst_id, inst_first_name, inst_last_name, categories_taught, ...)
         categories_taught = "DTP, SS, WP"   ← multivalued

Step 1: Remove categories_taught from Instructors
Step 2: Create Instructor_Categories(inst_id FK, category_taught)
Step 3: Add Instructor_Categories to Final Table List (type: Data)

After:
Instructors(inst_id PK, inst_first_name, inst_last_name, ...)
Instructor_Categories(inst_id FK, category_taught)
         — one row per instructor per category, no limit on count
```

**When multiple multivalued fields are correlated:** If `categories_taught` and `max_level_taught` have a one-to-one correspondence per record, they must travel together to the new table.

```
Instructor_Categories(inst_id FK, category_taught, max_level_taught)
```

---

## Elements of the Ideal Table — the six-element checklist

Every table must satisfy all six. These embed what normalization addresses through 1NF–5NF.

**Element 1 — Represents a single subject (object or event).**
If a table represents more than one subject, split it. The most common violation: one giant table for everything.

**Element 2 — Has a primary key.**
Every table must have exactly one field (or combination of fields) that uniquely identifies each record. You cannot have a properly designed table without one.

**Element 3 — Does not contain multipart or multivalued fields.**
You should have addressed these already. This is a final check.

**Element 4 — Does not contain calculated fields.**
Same — check your work. These belong in views.

**Element 5 — Does not contain unnecessary duplicate fields.**
Duplicate fields appear in two or more tables for three reasons:
- To relate two tables (the only acceptable reason)
- To indicate multiple occurrences of the same value type (resolve as multivalued)
- To supply reference information from another table (resolve by removing — use a view instead)

**Element 6 — Contains only an absolute minimum amount of redundant data.**
A perfectly normalised database will always contain some redundant data (foreign key values repeat). Your goal is to minimise it, not eliminate it entirely.

### Testing a table on paper

When in doubt about whether a table is ideal, draw it on paper and load it with 5–7 representative rows. Anomalies become visible immediately:
- Multipart values become obvious when you try to sort by city
- Multivalued values show their commas
- Reference fields show identical values repeating across many rows
- Missing primary key becomes obvious when you try to find a specific record

---

## Subset tables — when and how to create them

A subset table represents a subordinate subject of a particular data table. Use it when:
- A table contains fields that do not always have values for every record (many Nulls)
- Investigation reveals those fields describe a specific subtype of the main subject

```
Inventory table (bad):
item_name, current_value, insured_value, date_entered,
manufacturer, model, warranty_expiry,     ← only for Equipment
publisher, author, isbn, category          ← only for Books

→ The table represents three subjects: Inventory, Equipment, Books.

Fix:
Inventory(item_name PK, current_value, insured_value, date_entered)  ← common fields
Equipment(item_name PK/FK, manufacturer, model, warranty_expiry)      ← subset
Books(item_name PK/FK, publisher, author, isbn, category)             ← subset
```

**Rules for subset tables:**
- Shares the same primary key as its parent data table
- Contains only fields unique to the subordinate subject
- Does NOT contain fields shared with the parent table (those stay in the parent)

**Identifying previously unknown subset tables:** When two tables have nearly identical structures with only a few unique fields each, they are often subtypes of the same subject. Extract the common fields into a new parent data table.

---

## Common mistakes

**Mistake 1 — Starting table design from the current database structure.**
The legacy tables will have all the problems you are trying to fix. Start from the Preliminary Field List, not the old tables.

**Mistake 2 — Creating a table named "Miscellaneous".**
This name means you cannot identify the subject. Stop, reexamine the fields, apply the process methodically. There is no legitimate miscellaneous table.

**Mistake 3 — Keeping duplicate reference fields.**
`INSTRUMENTS` table containing `manufacturer_phone` and `manufacturer_website` that already exist in a `MANUFACTURERS` table — these are reference fields. Remove them. Use a view when you need to display both together.

**Mistake 4 — Flattening multivalued fields into multiple columns.**
`category_1`, `category_2`, `category_3` is not a solution. It is a new version of the original problem with a hard cap. Use the three-step multivalued resolution procedure.

**Mistake 5 — Not loading sample data to validate.**
Anomalies that are invisible in a table structure become immediately obvious with five rows of real data. Always test on paper before committing to the structure.

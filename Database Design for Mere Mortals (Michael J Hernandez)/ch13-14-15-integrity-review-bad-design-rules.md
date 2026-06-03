# ch13 — Reviewing Data Integrity

## What problem does this solve and why does it exist

You have been careful throughout the entire process. But the final design review exists because careful is not the same as perfect. You have made dozens of decisions across seven phases. The review is a structured final check before handing the logical design to whoever will implement it. The peace of mind is worth the time.

---

## The final review checklist — four integrity levels

Work through each level sequentially. Each level has its own checklist.

### Table-level integrity checklist

For every table in the database, verify:
- [ ] No duplicate fields exist in the table (except FKs)
- [ ] No calculated fields exist in the table (they belong in views)
- [ ] No multivalued fields exist
- [ ] No multipart fields exist
- [ ] No duplicate records can exist (PK enforces this)
- [ ] Every record is identified by a primary key value
- [ ] The primary key conforms to all Elements of a Primary Key
- [ ] The primary key exclusively identifies the value of each non-key field

If problems exist: use techniques from Ch07 (table structures) and Ch08 (keys).

### Field-level integrity checklist

For every field in the database, verify:
- [ ] Field conforms to all Elements of the Ideal Field
- [ ] A complete field specification exists for the field
- [ ] The field's definition is consistent wherever it appears in the database
- [ ] The Range of Values is specific and meaningful (no "Other", no "Miscellaneous")
- [ ] Edit Rule, Null Support, and Required Value settings are deliberate and correct

If problems exist: use techniques from Ch09 (field specifications).

### Relationship-level integrity checklist

For every relationship in the database, verify:
- [ ] The relationship is properly established (correct FK mechanics, matching names)
- [ ] The appropriate deletion rule is defined
- [ ] The type of participation is correctly identified for each table
- [ ] The degree of participation is correctly set for each table
- [ ] Each FK conforms to all Elements of a Foreign Key
- [ ] FK field specifications use Replica type and reference the source PK

If problems exist: use techniques from Ch10 (table relationships).

### Business rules checklist

For every business rule in the database, verify:
- [ ] The rule imposes a meaningful constraint
- [ ] The correct category is assigned (field-specific or relationship-specific)
- [ ] The rule is properly established (field spec elements or relationship characteristics modified)
- [ ] The appropriate validation tables are created where needed
- [ ] A Business Rule Specifications sheet is complete for every rule
- [ ] The actions that test the rule (Insert, Update, Delete) are identified

If problems exist: use techniques from Ch11 (business rules).

### Views checklist

For every view in the database, verify:
- [ ] Each view is built on the correct base tables
- [ ] All necessary fields are assigned
- [ ] Calculated fields provide meaningful information
- [ ] Filters return the correct set of records
- [ ] A view diagram exists
- [ ] A View Specifications sheet accompanies every view diagram

If problems exist: use techniques from Ch12 (views).

---

## Assembling the design documentation

After the review, assemble all artefacts into a central repository. This documentation is what the implementation team uses to build the physical database.

**Complete documentation package:**
- Final Table List (name, type, description for each table)
- Field Specifications sheets (one per field)
- Calculated Field List
- Table structure diagrams
- Relationship diagrams (with deletion rules, participation, degree)
- Business Rule Specifications sheets
- View diagrams + View Specifications sheets
- Interview notes (appendix)
- Data collection and presentation samples (appendix)

**Why this documentation matters:**
1. Complete record of the logical structure — any question about the design can be answered from this package
2. Blueprint for implementation — the implementer has precise specs for every field, every relationship, every constraint
3. Impact analysis — if someone proposes a structure change later, this documentation shows exactly what else is affected

---

---

# ch14 — Bad Design: What Not to Do

## What problem does this solve and why does it exist

Now that you know what good design looks like, you can recognise and diagnose bad design when you encounter it. These three patterns are the most common causes of poorly structured databases. You will see all of them in legacy systems.

---

## Pattern 1 — Flat-file design ("throw everything into one big table")

Everything in a single table. Zero normalisation. Maximum chaos.

**Example structure — CUSTOMER_ORDERS:**
```
Order_Number, Order_Date, Ship_Date, Order_Amount, Sales_Rep_Name,
Customer_Number, Customer_Name, Customer_Address, Customer_Phone,
Item_1, Quantity_1, Price_1, Extension_1,
Item_2, Quantity_2, Price_2, Extension_2,
Item_3, Quantity_3, Price_3, Extension_3
```

**Every violation in one structure:**
- Multipart fields: `Sales_Rep_Name` (first + last), `Customer_Name`, `Customer_Address`
- Calculated fields: `Extension_1` = Quantity_1 × Price_1; `Order_Amount` = sum of all extensions
- Unnecessary duplicate fields: `Item_1`, `Item_2`, `Item_3` (flattened multivalued field)
- No true primary key: `Order_Number` repeats if a customer orders more than 3 items
- Multiple subjects: customer, order, and order line items all in one table

**Consequences:** Redundant data, inconsistent data, no data integrity, queries that are near-impossible to write, inability to add more than 3 items to an order.

---

## Pattern 2 — Spreadsheet design

Using a spreadsheet as a database. This is appropriate for calculations and financial modelling. It is not appropriate for collecting, storing, and relating data across subjects.

**Characteristic violations:**
- Duplicate fields across columns (each column is essentially the same field repeated)
- Multipart fields (name and phone in one cell)
- Multivalued fields (multiple assistants in one cell)
- No primary key possible
- Impossible to relate to other "tables"
- Difficult to query (finding all managers named "Smith" requires manual scanning)

**The spreadsheet view mindset to break:** In a spreadsheet you see data exactly as it appears in the report. In a proper database, the data is stored normalised; the report is a view. These are different things. You must stop designing tables to look like reports.

---

## Pattern 3 — Designing around the RDBMS software

Building a database by opening the RDBMS and creating tables directly, guided by what the software makes easy rather than what the data requires.

**Why this is dangerous:**
- You make design decisions based on your perception of what the RDBMS can or can't do — not based on the organisation's requirements
- The RDBMS dictates the design instead of the design dictating what to ask of the RDBMS
- Your design is constrained by your current knowledge of the tool
- Result: technically functional database with poor structural design, insufficient integrity, inconsistent data

**The principle:** Always design the logical structure independently of any RDBMS. Complete the logical design first. Then choose the RDBMS. Then implement. The logical design belongs to the organisation, not to any software vendor.

---

---

# ch15 — Bending or Breaking the Rules

## What problem does this solve and why does it exist

Hernandez is emphatic throughout the book: follow the process completely. But he is equally pragmatic: two circumstances exist where departing from proper design is not only acceptable — it may be necessary. This chapter defines those circumstances precisely and gives you the discipline to handle them correctly.

---

## The default position

Breaking design rules always introduces data-integrity problems. This is not a maybe — it is a certainty. Every rule you break is a trade-off of some form of correctness for something else (usually performance or expediency).

The question is never: "Can I break this rule?" The question is: "Is the benefit worth the integrity cost, and have I exhausted all alternatives?"

---

## Two circumstances where bending rules is acceptable

### Circumstance 1 — Designing an analytical database

An analytical database (data warehouse, dimensional model) stores and tracks historical, time-dependent data for trend analysis and statistical projections. It requires a radically different methodology than what this book covers.

Analytical databases legitimately contain:
- Calculated fields (storing the state of calculated values at a point in time)
- Aggregate values materialised into the table
- Denormalised structures for query performance

**The rule:** Design the operational database properly first. Build the analytical database separately using a proper dimensional methodology. Never let analytical requirements contaminate the operational design.

### Circumstance 2 — Improving processing performance (last resort only)

If multitable queries or complex reports are genuinely too slow, and hardware/software optimisation has not resolved the problem, you may need to restructure the data. This is the most common pressure to break rules.

**Always try these alternatives first, in this order:**
1. **Hardware upgrade** — faster CPU, more RAM, SSD storage, faster network — the easiest and least disruptive
2. **Operating system tuning** — optimise OS configuration for database workload
3. **Review the database design** — a poorly designed database is inherently slower; fix structural problems first
4. **Review the implementation** — are indexes defined? Are query plans correct? Is the RDBMS configured correctly?
5. **Review application code** — are queries written efficiently? Are reports poorly designed?

**Only after all of the above:** consider structural modifications for performance.

**If you do break a rule:** the integrity consequences are yours to manage. Any rule you break introduces inconsistent data risk, redundant data risk, or relationship integrity risk. You must compensate through application code. Document everything.

---

## Documenting rule violations — mandatory

If you depart from proper design, document every single action.

**What to record:**
- **The reason** for breaking the rule — what performance problem was being addressed
- **The design principle being violated** — which element, which rule, which table or relationship
- **The specific aspect being modified** — which table, field, or relationship was changed
- **The exact modifications made** — what changed from what to what
- **The anticipated effects** — on data integrity, on views, on application programs, on queries

Add this document to the design repository. Even if you reverse the changes later, the record prevents someone from making the same mistake again.

---

## Applied summary — when rules can be relaxed in JMC's context

```
In the JMC Fabric Lakehouse stack:

Bronze layer (raw landing):
  Rules relaxed: Multipart fields from source, calculated fields from source,
                 no primary keys enforced. Bronze mirrors sources faithfully.
  Why acceptable: Bronze is not operational. It is a transit zone.
  Required: Document every structural violation and why it exists.

Silver layer (3NF source of truth):
  Rules: Full design discipline. No exceptions.
  Primary keys, FK relationships, field specs enforced.

Gold layer (dimensional model for Power BI):
  Rules relaxed: Calculated fields materialised (enrolment_count, pass_rate_pct),
                 denormalised dimensions (dim_date includes semester hierarchy),
                 aggregate fact tables.
  Why acceptable: This is an analytical model — Circumstance 1 above.
  Required: Design the silver layer first. Gold is derived from silver.
            Document each denormalisation decision and its integrity trade-off.

In Microsoft Fabric specifically:
  Calculated fields in Direct Lake mode: materialise into Delta table columns
  where query performance requires it; document as performance exception.
  Never in silver layer tables.
```

---

## Final principle

Hernandez closes with this:

> People who understand the fundamental principles of proper database design have a better comprehension of their RDBMS and the tools it provides than those who know little about design.

Knowing *why* a rule exists lets you use your tool more effectively, recognise when the tool is guiding you toward a bad decision, and know precisely when and how to deviate from the rule with full awareness of the consequences.

The rules are not bureaucracy. They are the accumulated results of 50 years of database failures and the mathematical foundations that prevent them.

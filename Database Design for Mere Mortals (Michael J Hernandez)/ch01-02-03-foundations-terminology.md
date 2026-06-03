# ch01–03 — Relational Database Foundations & Terminology

## What problem does this solve and why does it exist

You cannot design something you do not understand. These three chapters establish the conceptual bedrock: what a relational database *is*, why it is worth designing properly, and what every term means before you use it. Hernandez's explicit argument is that the #1 cause of database problems is not insufficient SQL knowledge — it is insufficient design knowledge. You can implement a bad design perfectly and still have bad data.

---

## Ch01 — The Relational Database

### Operational vs analytical databases

| Type | What it stores | Data state | Primary use case |
|------|---------------|------------|-----------------|
| **Operational** | Current state of business transactions | Dynamic — changes constantly | OLTP: insert, update, delete every day |
| **Analytical** | Historical snapshots for trend analysis | Static — rarely modified | OLAP: trend tracking, statistical projections |

**Why this matters to you:** This entire book is about designing **operational** databases. An analytical database (data warehouse, dimensional mart) requires a radically different methodology. Never confuse the two mid-design — it leads to hybrid structures that serve neither purpose well.

### The mathematical foundation

Codd published "A Relational Model of Data for Large Shared Databanks" in June 1970. He applied two branches of mathematics:
- **Set theory** — the name "relational" comes from *relation*, a term in set theory (not from the fact that tables relate to each other, which is a common misconception)
- **First-order predicate logic** — provides the formal basis for querying

This mathematical foundation is what makes the relational model predictable, reliable, and capable of guaranteeing accurate information. Every behaviour you can count on — uniqueness enforcement, referential integrity, join semantics — derives from this foundation.

### Key structural facts

- Data in a relational database is stored in **relations** (perceived by users as **tables**)
- Each relation is composed of **tuples** (records) and **attributes** (fields)
- The **physical order of records or fields is completely immaterial** — records are identified by values, not by position
- A user does not need to know the physical location of a record to retrieve it

### The four advantages you can promise stakeholders

1. **Built-in multilevel integrity** — field level, table level, relationship level, business rules
2. **Logical and physical data independence** — changes to the logical structure need not affect applications; changes vendors make to physical storage need not affect the model
3. **Guaranteed data consistency and accuracy** — enforced at multiple levels
4. **Easy data retrieval** — data from any number of related tables can be retrieved in an almost unlimited number of ways

---

## Ch02 — Design Objectives

### Why design matters

The primary reason for poor-quality databases is not bad code — it is bad design. Inaccurate information is the worst possible outcome: your organisation makes decisions based on data the database cannot guarantee is correct.

Hernandez's analogy: you would not hire a contractor to build your house without first engaging an architect. The architect produces blueprints (logical design). The contractor builds from them (physical implementation). Skipping the blueprints means the contractor builds whatever they think you want.

### Hernandez's design method vs traditional normalization

Traditional methods: requirements analysis → data modelling → **normalization as a separate, later step**.

Hernandez's insight: normalization is not a separate step. It is **transparently incorporated throughout** the entire process via specific guidelines:

| My methodology component | What it absorbs from normalization |
|--------------------------|-----------------------------------|
| Elements of the Ideal Field | Scalar values, multivalued/multipart fields, calculated values |
| Elements of the Ideal Table | Functional/multivalued/transitive/join dependencies, duplicate fields |
| Elements of a Candidate/Primary Key | Functional dependencies, determinates |
| Elements of a Foreign Key | Referential integrity, modification anomalies |
| Field Specifications | Domains, domain integrity |
| Relationship Characteristics | Cardinality, optionality, deletion rules |

**The practical result:** if you follow Hernandez's methodology faithfully, you get a fully normalised database without ever directly applying 1NF/2NF/3NF tests. The tests are embedded in the guidelines. This is why he can say "normalization is transparent" — not because it doesn't apply, but because its results are baked into every earlier decision.

### Five objectives of good design

Every table decision you make should be tested against these:

1. The database supports both required and ad hoc information retrieval
2. Tables are constructed properly and efficiently (single subject, distinct fields, unique identifier)
3. Data integrity is imposed at the field, table, and relationship levels
4. The database supports relevant business rules
5. The database lends itself to future growth

### Four benefits you get when you meet those objectives

1. Structure is easy to modify and maintain
2. Data is easy to modify (changes to one field don't require changes everywhere else)
3. Information is easy to retrieve (well-constructed tables make queries obvious)
4. End-user applications are easy to build

### The rule you must internalise

> There's never time to do it right, but there's always time to do it over.

Hernandez is explicit: **do not take shortcuts**. An incomplete design is a poor design. The level of structural integrity in your database is directly proportional to how thoroughly you follow the process.

---

## Ch03 — Terminology

### Value-related terms

**Data vs Information — the most important distinction in the book**

> Data is what you store; information is what you retrieve.

This single axiom explains the entire purpose of database design. You design a database to produce meaningful information. If the data is improperly stored, the information will be inaccurate. If the structure doesn't support a query, the information is unavailable.

**Null** — represents a *missing or unknown* value. It is NOT:
- A zero (zero can be a meaningful value — current balance, stock level)
- A blank space (a blank is a valid character to SQL)
- A zero-length string ('' is meaningful in some contexts)

Null has a severe side effect: **any mathematical operation involving a Null evaluates to Null**. `(25 × 3) + Null = Null`. An aggregate function like `COUNT(<fieldname>)` will show 0 for an unspecified category, hiding the fact that records with no category exist. This is an undetected error.

Use Nulls only when a value is genuinely missing or unknown. If "N/A" or "Not Applicable" is a meaningful state, store it as a real value.

### Structure-related terms

**Table** — the chief structure in the database. Always represents a **single, specific subject**. Two types:

| Type | What it holds | Data state |
|------|--------------|-----------|
| **Data table** | The primary information of interest | Dynamic — you constantly interact with it |
| **Validation table** (lookup table) | Values used to implement data integrity | Static — rarely changes after initial population |

**Field** — the smallest structure. Represents ONE characteristic of the table's subject. Stores data. Has a name that identifies exactly what type of value it holds. Contains one and only one value.

Three problematic field types you must eliminate:
- **Multipart field** — contains two or more distinct items (e.g., `customer_name` = "John Smith")
- **Multivalued field** — contains multiple instances of the same type (e.g., `phone_numbers` = "555-1234, 555-5678")
- **Calculated field** — stores the result of a concatenation or mathematical expression

**Record** — represents one unique instance of the table's subject. Composed of all fields in the table. Identified by a primary key value.

**View** — a virtual table. Does not store data. Draws data from base tables. Rebuilt and repopulated every time you access it. Three types:
- Data view (single-table or multitable)
- Aggregate view (calculated summaries)
- Validation view (restricts access to a subset of a table's fields)

**Key** — a logical structure for identifying records.
**Index** — a physical structure your RDBMS uses to optimise data retrieval. **These are not the same thing.** Keys are logical design decisions. Indexes are implementation details. Never confuse them.

### Relationship-related terms

**Three relationship types:**

| Type | Definition | Example |
|------|-----------|---------|
| **One-to-one (1:1)** | Single record in A relates to at most one record in B, and vice versa | Employee → Compensation |
| **One-to-many (1:M)** | Single record in A can relate to many records in B; single record in B relates to one record in A | Customer → Orders |
| **Many-to-many (M:M)** | Single record in A can relate to many records in B, and vice versa | Students ↔ Classes |

M:M is "unresolved" — you cannot efficiently associate records from A with records in B without a **linking table** (also called an associative table or junction table). The linking table uses the primary keys of both A and B as its composite primary key.

**Three ways to characterise every relationship:**

1. **Type** — 1:1, 1:M, or M:M
2. **Participation** — Mandatory or Optional
   - **Mandatory:** at least one record must exist in this table before you can enter records in the related table
   - **Optional:** no requirement for any records in this table before entering records in the related table
3. **Degree of participation** — minimum and maximum number of records that can be interrelated
   - Written as `(min, max)` — e.g., `(1,15)` means at least 1, no more than 15
   - Use `N` for unlimited maximum: `(0,N)`

### Integrity-related terms

**Field specification** — the complete set of elements defining every attribute of a field. Hernandez calls this a "domain" in traditional terminology. Three categories:
- **General elements:** Field Name, Parent Table, Specification Type, Source Specification, Shared By, Alias(es), Description
- **Physical elements:** Data Type, Length, Decimal Places, Character Support
- **Logical elements:** Key Type, Key Structure, Uniqueness, Null Support, Values Entered By, Required Value, Range of Values, Edit Rule

**Four types of data integrity:**

| Type | What it ensures |
|------|----------------|
| **Table-level** (entity integrity) | No duplicate records; primary key is unique and never Null |
| **Field-level** (domain integrity) | Every field's structure is sound; values are valid, consistent, accurate; same-type fields are consistently defined |
| **Relationship-level** (referential integrity) | Relationships between tables are sound; records are synchronised when data changes |
| **Business rules** | Restrictions based on how the organisation perceives and uses its data |

All four levels work together. A crack in any one of them can make information inaccurate. You cannot compensate for poor table-level integrity with strong field-level integrity.

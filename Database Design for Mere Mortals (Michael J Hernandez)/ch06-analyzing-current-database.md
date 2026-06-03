# ch06 — Analyzing the Current Database

## What problem does this solve and why does it exist

You are almost never starting from nothing. There is always an existing system — paper forms, a legacy database, spreadsheets, a human knowledge base. If you do not understand the current system first, you will either replicate its flaws or miss data that already exists and needs to be migrated. This chapter teaches you how to extract maximum useful information from whatever exists, without being constrained by its structure.

**The most important rule in this chapter:**

> Do not adopt the current database structure as the basis for the new database structure.

The current structure is what you are analysing, not what you are copying. Every hidden problem in the current database will transfer into the new one if you copy it. If the current database didn't have problems, you wouldn't be building a new one.

---

## Three types of current databases

| Type | Description | Difficulty to analyse |
|------|-------------|----------------------|
| **Paper-based** | Forms, notebooks, file cabinets. Often inconsistent, erroneous, duplicate, redundant. | High — find someone who understands the whole system |
| **Legacy database** | Exists for 5+ years. Often poorly designed, sometimes by people with no design training. Two common flaws: duplicate fields, redundant data. | Medium — structures are explicitly defined |
| **Human knowledge base** | Lives in one or two people's heads. Crucial to the business, fragile, undocumented. | Highest — completely dependent on interviews |

---

## Three steps in the analysis process

### Step 1 — Review how data is collected

Gather samples of everything used to collect data:
- Paper forms, handwritten lists, index cards
- Application program data entry screens (screenshots)
- Web forms

For each: gather one sample as complete as possible, store in a folder labelled with source, program name, and date. Do not discuss samples in depth yet — just collect them.

### Step 2 — Review how information is presented

Gather samples of everything used to present data as information:
- Reports (handwritten, printed, or generated)
- Screen presentations / slide shows that incorporate database data
- Web pages that display database information

Same procedure: collect, screenshot, label, file. Only review presentations that directly use database data.

### Step 3 — Conduct interviews with users and management

Now you have context. The samples show you how data is collected and presented. The interviews let you ask specific questions about them.

Four things user interviews accomplish:
1. Clarify aspects of samples you don't understand
2. Reveal how users work with the data daily
3. Define preliminary field and table structures
4. Expose future information requirements

Interview procedure for analysis: alternate between open-ended and closed questions. Open-ended first to establish subjects, then progressively narrow.

```
Open-ended:     "How would you define the work you do on a daily basis?"
                → Generates subjects (table candidates)

Follow-up:      "Let's discuss sales orders. What does it take to complete one?"
                → Generates characteristics (field candidates)

More specific:  "Tell me about the client's fax number and shipping address."
                → Nails down specific field details
```

Apply the Subject-Identification Technique and Characteristic-Identification Technique from Ch05 throughout.

**Interview users first.** They see the operational detail. Their answers help you understand management's answers later.

---

## Compiling the Preliminary Field List

After reviews and interviews, compile everything into a **Preliminary Field List** — a complete, raw collection of all data items mentioned.

**Rules for the Preliminary Field List:**
- Every field has a **unique name** — if two fields from different sources seem to describe the same characteristic, investigate whether they are truly the same before merging
- Record each field once, even if it appears in multiple sources
- Attach a **value list** to any field that uses a restricted set of values — document those values now (e.g., `status_code: Active, Inactive, Pending`)
- If the same data is collected on multiple forms with different names, resolve to one name now

**Do not guess — record what you find.** If you are unsure about a field's purpose, note the uncertainty and resolve it in the next interview.

### The Calculated Field List

Before the Preliminary Field List is finished, remove every calculated field and put it on a **separate Calculated Field List**.

A calculated field stores the result of:
- A mathematical expression (`order_total = quantity × unit_price`)
- A string concatenation (`full_name = first_name + ' ' + last_name`)
- An aggregate function result (`total_units_sold = SUM(quantity)`)

**Why remove them now?** Calculated fields cannot exist in a properly designed table (they violate the Elements of the Ideal Field, which you will learn in Ch07). But they are not lost — you will use them later when defining views (Ch12).

**Name signals for calculated fields:** fields containing words like *amount*, *total*, *sum*, *average*, *minimum*, *maximum*, *count*, *subtotal*, *average age*, *customer count* are almost certainly calculated.

```
Preliminary Field List:
customer_first_name, customer_last_name, customer_phone,
order_date, ship_date, product_name, quantity, unit_price,
order_subtotal, order_tax, order_total          ← move these three

Calculated Field List:
order_subtotal = quantity × unit_price
order_tax      = order_subtotal × tax_rate
order_total    = order_subtotal + order_tax
```

### Review both lists with users and management

This is a brief review. Your goal: confirm no fields are missing. They will almost certainly identify a few you forgot. This is expected. Add them, move calculated fields to the right list, and declare the lists "final" — meaning: complete enough to proceed.

Never expect the list to be truly complete. Fields you did not think of will emerge during Phase 3 when you review table structures. That is normal. Design is iterative.

---

## Practical example — Legacy source system analysis (JMC scenario)

Applying this to JMC Academy (sources: HubSpot, Dynamics 365, Paradigm, Canvas):

```
Step 1 — Data collection review:
  HubSpot:    Lead capture forms (first_name, last_name, email, phone,
              course_interest, lead_source, agent_code)
  Dynamics:   Application forms (application_date, course_code, campus,
              delivery_mode, visa_type, agent_id)
  Paradigm:   Enrolment records (enrolment_id, student_id, unit_code,
              semester, status, tuition_amount, start_date)
  Canvas:     Submission records (assignment_id, student_id, submitted_at,
              grade, grade_type)

Step 2 — Information presentation review:
  Admissions report: applications by course/semester/agent
  Academic report:   pass rates by unit/campus/semester
  Finance report:    tuition revenue by course/campus
  Student services:  withdrawal and deferral counts by qualification/year

Preliminary Field List (excerpt):
student_id, first_name, last_name, email, phone,
date_of_birth, visa_type, is_international,
application_id, application_date, application_status,
course_code, course_name, unit_code, unit_name,
campus_code, campus_name, delivery_mode,
semester_code, academic_year,
enrolment_id, enrolment_date, enrolment_status,
tuition_amount,
assignment_id, grade_score, is_pass,
agent_id, agent_name, agent_commission_rate

Calculated Field List:
days_to_decision    = offer_date - application_date
pass_rate_pct       = (pass_count / attempt_count) × 100
tuition_revenue     = SUM(tuition_amount)
completion_rate_pct = (completed_count / enrolled_count) × 100
```

---

## Common mistakes

**Mistake 1 — Adopting the current database structure.**
The legacy system was designed by someone who didn't know database design. It has multipart fields, duplicate fields, no primary keys. If you copy it, you import all of those problems.

**Mistake 2 — Not creating the Calculated Field List.**
Leaving calculated fields in the Preliminary Field List means they will accidentally end up in table structures in Phase 3, violating the Elements of the Ideal Field. Remove them now.

**Mistake 3 — Skipping source system review and going straight to interviews.**
The samples let you ask specific, concrete questions. Without them, your interviews are abstract and incomplete.

**Mistake 4 — Assigning one large field list to everyone.**
Different source systems use different names for the same thing. `customer_id` in HubSpot and `client_number` in Paradigm may be the same concept or different concepts. Investigate before merging.

**Mistake 5 — Resolving field name conflicts arbitrarily.**
When two sources use different names for what appears to be the same field, confirm it with interviews. Do not assume.

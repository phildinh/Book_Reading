# Chapter 8 — CRM: Bridge Tables for Multivalued Dimensions

## The Core Rule Being Broken

A fundamental rule of dimensional modelling: every dimension attached to a fact table must take exactly ONE value per fact row at the declared grain.

Most dimensions respect this naturally. One student per enrolment. One date per transaction. One course per registration.

But some dimensions naturally have multiple values at the fact grain. A course co-taught by two lecturers. A bank account held by two people. A patient with twelve simultaneous diagnoses. A student receiving three scholarships at once.

You cannot solve this by adding more FK columns — that changes the grain and introduces empty columns for most rows. The bridge table is the correct solution.

---

## Two Approaches to Multivalued Dimensions

### Option 1 — Positional Design

Add a separate column for each possible value:
```
fact_table:
  lecturer_1_key (FK)
  lecturer_2_key (FK)
  lecturer_3_key (FK)
```

**Works when:** the maximum number of values is small, fixed, and known upfront.

**Breaks when:** the number of values is unpredictable or large. You'd need to alter the schema every time the maximum increases. Most rows have many null columns.

### Option 2 — Bridge Table

Introduce an intermediate table between the fact table and the dimension. The fact table carries one FK to a "group" key. The bridge table maps groups to individual dimension members.

---

## Bridge Table Structure

### Three Components

**1. Lecturer Dimension** (one row per lecturer, as normal)
```
lecturer_dim:
  lecturer_key (PK)
  lecturer_name
  department
  ...
```

**2. Lecturer Group Bridge** (the bridge table)
```
lecturer_group_bridge:
  lecturer_group_key (FK)
  lecturer_key (FK)
  weight
```

**3. Fact Table** (one FK to the group, not to individual lecturers)
```
course_registration_fact:
  term_key (FK)
  student_key (FK)
  course_key (FK)
  lecturer_group_key (FK)   ← one FK, no matter how many lecturers
  registration_count = 1
```

### How Groups Work

If Audio Engineering is co-taught by Dr Kim and Mr Osei → they share `lecturer_group_key = 1`:
```
Bridge:
  group_key=1 | lecturer_key=201 | weight=0.5   (Dr Kim)
  group_key=1 | lecturer_key=202 | weight=0.5   (Mr Osei)
```

If Music Production is taught by Ms Novak alone → she gets her own group `lecturer_group_key = 2`:
```
Bridge:
  group_key=2 | lecturer_key=203 | weight=1.0   (Ms Novak)
```

---

## The Double-Counting Risk

### Without Weighting

Query: "How many enrolments and what total fees per lecturer?"

```sql
SELECT l.lecturer_name, COUNT(f.student_key) AS enrolments, SUM(f.enrolment_fee) AS fees
FROM enrolment_fact f
JOIN lecturer_group_bridge b ON b.lecturer_group_key = f.lecturer_group_key
JOIN lecturer_dim l ON l.lecturer_key = b.lecturer_key
GROUP BY l.lecturer_name
```

If 2 students enrolled in Audio Engineering at $15,000 each:
- Dr Kim gets: 2 enrolments, $30,000 ← doubled
- Mr Osei gets: 2 enrolments, $30,000 ← doubled
- Grand total fees: $60,000 — actual total was $30,000

The enrolment COUNT is correct (both lecturers genuinely participated in both enrolments). The fee SUM is wrong — each dollar is counted twice.

### With Weighting — The Fix

```sql
SELECT l.lecturer_name, 
       COUNT(f.student_key)          AS enrolments,
       SUM(f.enrolment_fee * b.weight) AS weighted_fees
FROM enrolment_fact f
JOIN lecturer_group_bridge b ON b.lecturer_group_key = f.lecturer_group_key
JOIN lecturer_dim l ON l.lecturer_key = b.lecturer_key
GROUP BY l.lecturer_name
```

Result:
- Dr Kim: 2 enrolments, $15,000 ✓
- Mr Osei: 2 enrolments, $15,000 ✓
- Grand total: $30,000 ✓

The weights (each 0.5) divide the financial facts proportionally. Grand total always equals the actual total.

---

## Two Types of Reports — Know Which You're Building

**Weighted report** — multiplies facts by bridge weights. Grand totals are correct. Financial allocation is accurate. Use when: revenue attribution, cost allocation, fee analysis.

**Impact report** — does NOT apply weights. Each dimension member gets the full fact attributed. Grand totals overcount. Use when: "what is the total volume of courses this lecturer is exposed to?" — a measure of exposure, not financial attribution.

Both are valid. The business question determines which one you need.

---

## When NOT to Use Weights

Healthcare is the clearest example. A patient has twelve simultaneous diagnoses. It is impossible to weight the financial impact of each diagnosis on the total treatment cost. In this case, omit weights entirely and acknowledge that any sum across the bridge table is an impact report (overcounting). The analysis shifts to counts and proportions rather than allocated financial sums.

---

## Behaviour Study Groups

A behaviour study group is a small physical table containing just the durable natural keys of a specific customer cohort. Build it once from a complex query. Reuse it to constrain any fact table to that cohort without rerunning the complex logic.

```sql
-- Build the study group once
CREATE TABLE at_risk_students AS
SELECT DISTINCT s.student_natural_id
FROM course_registration_fact r
JOIN student_dim s ON s.student_key = r.student_key
LEFT JOIN attendance_fact a ON a.student_key = r.student_key 
    AND a.course_key = r.course_key
WHERE a.student_key IS NULL;  -- enrolled but never attended

-- Reuse across any fact table
SELECT c.course_name, SUM(e.enrolment_fee) AS revenue
FROM enrolment_fact e
JOIN student_dim s ON s.student_key = e.student_key
JOIN at_risk_students g ON g.student_natural_id = s.student_natural_id
JOIN course_dim c ON c.course_key = e.course_key
GROUP BY c.course_name;
```

**Why use the durable natural key (not the surrogate key):** surrogate keys change with SCD Type 2 updates. The natural key is stable across all historical versions of a student's dimension row. The study group remains valid even as students' attributes change.

---

## Step Dimensions for Sequential Behaviour

When you need to place an individual event into the context of a sequence (e.g. which page is this in the user's overall session? Which step of the application process did this student drop off at?), a step dimension captures positional context:

```
step_dim:
  step_key (PK)
  total_steps_in_sequence
  this_step_number
  steps_remaining
```

Pre-populate for all possible sequence lengths. Attach to the fact table with multiple role-playing step FKs for different subsequence contexts (overall session, successful completion, abandonment).

**What this enables:** "which step in the application process do most students abandon?" — simply filter where `steps_remaining = 0` and `step_key → abandonment_step` role.

---

## Customer Data Integration — Partial Conformity

In environments with many customer-facing source systems (HubSpot, Dynamics, Canvas, Paradigm), building one perfectly identical conformed customer dimension is sometimes impractical.

**Lighter weight approach:** identify a subset of attributes that have significance across all systems and conform only those. Start with a high-level `customer_category` attribute. Plant it in every customer-related dimension across all sources.

This least-common-denominator conformity still enables drill-across on those shared attributes. Over time, incrementally expand the set of conformed attributes. Each expansion increases analytical integration without requiring a complete redesign.

---

## Bridge Table vs Positional Design — Decision Guide

| Factor | Use Positional | Use Bridge Table |
|---|---|---|
| Number of possible values | Small and fixed (2–5) | Large or unpredictable |
| Schema changes | Acceptable if max increases | Not acceptable |
| Query complexity | Simple — users can write SQL | Acceptable — hidden in BI layer |
| Null columns | Acceptable | Not acceptable |
| Financial allocation | Use weighting or positional | Weighting factors in bridge |
| AND/OR querying | Simple WHERE clauses | Requires UNION/INTERSECT logic |

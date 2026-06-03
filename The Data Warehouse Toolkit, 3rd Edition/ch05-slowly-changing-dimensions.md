# Chapter 5 — Procurement: Slowly Changing Dimensions in Depth

## The Problem

Dimensions are not static. Products change departments. Customers move postcodes. Staff change roles. When an attribute value changes in the operational world, you must decide: **how does the data warehouse respond?**

For each dimension table attribute, you must specify a change-handling strategy. The choice determines whether historical fact rows continue to reflect what was true when they were loaded, or whether they silently change meaning.

---

## SCD Type 0 — Retain Original

The attribute never changes after first load. Appropriate for truly immutable attributes.

**Examples:**
- Customer's original sign-up date
- Student's original application source
- Date dimension attributes

---

## SCD Type 1 — Overwrite

Overwrite the old value with the new value. No history is preserved.

```
Before: | product_key | sku     | product_name | department  |
        | 12345       | ABC-922 | IntelliKidz  | Education   |

After:  | product_key | sku     | product_name | department  |
        | 12345       | ABC-922 | IntelliKidz  | Strategy    |
```

**Fact table:** untouched. All historical fact rows now appear as if the product always belonged to Strategy.

**Use when:**
- The change is a correction (wrong data, not a real business change)
- History genuinely doesn't matter for the attribute
- Business has explicitly agreed that current value is all they need

**Warning:** Type 1 invalidates any aggregate tables or OLAP cubes built on the changed attribute. They must be rebuilt.

---

## SCD Type 2 — Add New Row (The Primary Workhorse)

Insert a new dimension row for the new attribute profile. Old row is expired. Fact table untouched.

### Before the change:
```
| product_key | sku     | department | effective_date | expiration_date | current_row |
| 12345       | ABC-922 | Education  | 2022-01-01     | 9999-12-31      | Current     |
```

### After the change (department moves to Strategy on 2024-02-01):
```
| product_key | sku     | department | effective_date | expiration_date | current_row |
| 12345       | ABC-922 | Education  | 2022-01-01     | 2024-01-31      | Expired     |
| 25984       | ABC-922 | Strategy   | 2024-02-01     | 9999-12-31      | Current     |
```

**Fact table:** all rows loaded before 2024-02-01 still reference product_key 12345 (Education). New rows reference 25984 (Strategy). History is perfectly partitioned automatically.

### Required Admin Columns
Every Type 2 dimension needs three extra columns:
- `row_effective_date` — first date this row's attributes are valid
- `row_expiration_date` — `9999-12-31` for the current row
- `current_row_indicator` — `'Current'` or `'Expired'`

### ETL Logic (MERGE operation)
```python
# When attribute change detected:
# 1. UPDATE old row
UPDATE dim SET row_expiration_date = yesterday, current_row_indicator = 'Expired'
WHERE natural_key = X AND current_row_indicator = 'Current'

# 2. INSERT new row
INSERT INTO dim (new_surrogate_key, natural_key, new_attribute_values, 
                 effective_date = today, expiration_date = '9999-12-31', 
                 current_row_indicator = 'Current')

# 3. Fact table — DO NOTHING
```

**Use when:** must know what attribute value was in effect when each fact event occurred. The safest default if you're unsure.

### Counting Correctly with Type 2
Because a student can have multiple rows (one per campus they've been at), `COUNT(student_key)` overcounts. Use `COUNT(DISTINCT natural_key)` to count unique individuals. The `current_row_indicator` quickly restricts to current profiles.

---

## SCD Type 3 — Add New Column (Partial History)

Add a new column to capture the prior value. The current column is overwritten (Type 1 behaviour). Only one prior value is preserved.

```
Before: | product_key | department_name |
        | 12345       | Education       |

After:  | product_key | department_name | prior_department_name |
        | 12345       | Strategy        | Education             |
```

**Use when:**
- A significant mass reorganisation occurs and users need both old and new views simultaneously for a transition period
- Only one prior value needs to be preserved
- Changes are predictable and infrequent

**Not useful for:** attributes that change unpredictably — you would end up with a prior_department column from an arbitrary point in time that means nothing.

---

## SCD Type 4 — Mini-Dimension

Split rapidly changing attributes into a separate small dimension table. The fact table carries two FKs: one to the large main dimension, one to the mini-dimension.

```
Customer Dimension (stable attributes):
  customer_key (PK), customer_name, address, ...

Demographics Mini-Dimension (volatile attributes):
  demographics_key (PK), age_band, income_band, engagement_score_band

Fact Table:
  customer_key (FK → Customer Dim)
  demographics_key (FK → Demographics Mini-Dim)   ← current at time of event
  ... facts
```

**Why:** large dimensions (millions of rows) with rapidly changing attributes (monthly credit scores, weekly engagement scores) would generate an unsustainable number of Type 2 rows. Splitting the volatile attributes out keeps the main dimension stable.

**Tradeoff:** attribute values must be banded (ranges) rather than specific values to keep the mini-dimension manageable.

---

## SCD Type 6 — Add Type 1 Attributes to Type 2 Rows

Combines Type 1 + Type 2 in one dimension. Each row has:
- A historically accurate attribute (the value when this row was effective) — Type 2 behaviour
- A current attribute (always the latest value, overwritten on all rows) — Type 1 behaviour

```
| product_key | sku     | historic_department | current_department | effective | expiration | current |
| 12345       | ABC-922 | Education           | Strategy           | 2022-01-01| 2024-01-31 | Expired |
| 25984       | ABC-922 | Strategy            | Strategy           | 2024-02-01| 9999-12-31 | Current |
```

**What this enables:**
- Filter by `historic_department = 'Education'` → see fact rows from when the product was in Education
- Filter by `current_department = 'Strategy'` → see ALL fact rows for the product (both periods) attributed to its current home

**Use when:** need both "what was true when the event happened?" AND "roll up all history by the current attribute value" from the same dimension.

---

## SCD Type 7 — Dual Keys in Fact Table

The fact table carries two foreign keys: one surrogate key (pointing to the Type 2 history row in effect when the fact was loaded) and one durable natural key (pointing to a current-profile-only view).

Delivers the same capability as Type 6 but at the cost of an extra column in the fact table. Easier ETL than Type 6 because the current-profile table is just a view filtering `current_row_indicator = 'Current'`.

---

## Mixing SCD Types in One Dimension

A single dimension can — and usually should — mix SCD types across different attributes.

**Example for a Student dimension:**
| Attribute | SCD Type | Reason |
|---|---|---|
| Student name | 1 | Corrections only |
| Email address | 1 | Corrections only |
| Campus | 2 | Must know campus at time of each enrolment |
| Declared course | 2 | Course changes matter for academic reporting |
| Class level | 2 | Year 1/2/3 determines curriculum expectations |
| Engagement score | 4 | Changes weekly — mini-dimension to avoid bloat |

**When Type 1 occurs in a Type 2 dimension:** if the attribute should update on ALL rows (not just the current one), both the expired and current rows must be updated. Example: a data correction to a student's date of birth should propagate to all their historical dimension rows.

---

## SCD Design Checklist

For every dimension attribute, answer:
1. Does the business care about historical changes to this attribute? If no → Type 1.
2. How often does it change? If frequently and the dimension is large → Type 4.
3. Do users need both as-was (when fact occurred) AND as-is (current value) simultaneously? If yes → Type 6 or 7.
4. Is this a mass reorganisation where users need both old and new structures for a transition period? If yes → Type 3.
5. Everything else where history matters → Type 2.

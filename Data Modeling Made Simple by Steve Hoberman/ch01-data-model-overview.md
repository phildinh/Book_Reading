# ch01 — Data Model Overview

## What problem does this solve and why does it exist

Every organisation stores data, but without a shared picture of what that data represents, different teams build different interpretations of the same reality. The result: the Sales team's "customer" is not the same as Finance's "customer", and you discover this in production.

A data model is a formal, shared picture of what the data means — before any database is built. It forces agreement on terminology, scope, and rules while that agreement is still cheap. After the database is in production, disagreement is expensive.

---

## The core definition

> A data model is a set of symbols and text used to precisely explain a subset of real information, with the goal of improving communication and leading to a more flexible, stable application environment.

The key word is **precisely**. Two people looking at the same model must arrive at the same meaning. If they disagree on whether a symbol means A or B, the model has failed.

---

## Four uses of a data model

Hoberman identifies four reasons to build one — not just "before you build a database":

| Use | What it does | Example |
|-----|-------------|---------|
| Design new applications | Captures requirements before implementation | Star schema for a new analytics system |
| Understand existing applications | Reverse-engineer what a legacy system actually does | Document what 20-year-old tables mean before migrating them |
| Manage risk | Impact analysis — what breaks if you change this? | Before modifying a FK column, see every downstream table it touches |
| Educate team members | Onboard new engineers or business stakeholders faster | Walk a new analyst through the medallion layers using the model |

**Why this matters for you at JMC / Ettason:** You are regularly doing all four. When you add a column to `WeeklyPlanning`, you are doing risk management. When you onboard a new analyst to Power BI, you are doing education. The model is your documentation — even if it lives in dbt schema files and Power BI descriptions rather than a formal ER diagram.

---

## Three levels, one concept

The same entity exists at three levels of abstraction. Each level has a different audience and a different purpose:

```
Conceptual  →  Logical  →  Physical
  (scope)     (business)   (technology)

"Customer"  →  Customer entity with  →  dim_customer table with
               all attributes &         surrogate key, SCD Type 2,
               business rules           indexed columns
```

The CDM is for business stakeholders. The LDM is for data architects. The PDM is for DBAs and engineers. The same concept flows through all three — you are not building three different things, you are showing the same thing at different resolutions.

---

## The 80/20 rule for data modeling

Hoberman's practical heuristic: 20% of the time gets you to 80% complete. The remaining 20% requires answering edge-case questions that may not matter until the system is live.

**Implication:** Do not try to perfect a model before building. Iterate. A working 80% model in production is more valuable than a 100% model that never shipped.

**Anti-pattern:** Spending three weeks refining a CDM before writing a single dbt model. Get to the LDM and start building. The edge cases will surface from real data, not from whiteboard sessions.

---

## Rules to apply in real work

**Rule 1 — Agree on definitions before you build.**
The question "What is a student?" must have a single written answer before you create the `dim_student` table. If it doesn't, you will have multiple incompatible implementations of `student_id` across pipelines.

**Rule 2 — One model per audience.**
A CDM for the VP of Education is not the same document as an LDM for dbt model design. Don't show a business stakeholder a table with 47 columns and surrogate keys.

**Rule 3 — Keep the conceptual to one page.**
If your CDM has more than 20 concepts, it is at the wrong level of abstraction. Collapse child entities into parents until you are at the level the business talks about daily.

**Rule 4 — A model that lives only in your head is not a model.**
It must be written down, reviewed, and agreed upon. dbt schema.yml descriptions, Power BI dataset documentation, and Confluence pages all count.

---

## Common mistakes

**Mistake 1 — Starting at the physical.**
Building tables before agreeing on what they represent. The result is a schema that encodes someone's implementation assumptions, not the business rules. You cannot easily change it later.

**Mistake 2 — Confusing model levels.**
Mixing logical (business rule) decisions with physical (performance) decisions in the same document. Normalization is a logical decision. Partitioning is a physical decision. They belong in different conversations.

**Mistake 3 — Treating the model as a one-time artifact.**
A model that is not updated when the business changes is worse than no model — it is actively misleading. Treat your schema.yml and semantic layer definitions as the living model.

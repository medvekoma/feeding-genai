---
marp: true
theme: default
paginate: true
style: |
  section {
    font-family: 'Segoe UI', Arial, sans-serif;
    font-size: 28px;
  }
  section.cover {
    background: #0a0a23;
    color: #ffffff;
    text-align: center;
    justify-content: center;
  }
  section.cover h1 {
    font-size: 48px;
    color: #ffffff;
    margin-bottom: 16px;
  }
  section.cover p {
    color: #aaaacc;
    font-size: 24px;
  }
  section.section-header {
    background: #0057b8;
    color: #ffffff;
    justify-content: flex-end;
  }
  section.section-header h1 {
    font-size: 56px;
    color: #ffffff;
    margin: 0;
  }
  section.section-header p {
    color: #cce0ff;
    font-size: 22px;
    margin-top: 8px;
  }
  section.section-header .number {
    font-size: 120px;
    font-weight: 700;
    color: rgba(255,255,255,0.15);
    position: absolute;
    top: 20px;
    right: 40px;
    line-height: 1;
  }
  section.demo {
    background: #f5f5f5;
  }
  section.demo h1 {
    color: #0057b8;
  }
  section.demo ol {
    font-size: 26px;
  }
  section.takeover {
    background: #0a0a23;
    color: #ffffff;
    text-align: center;
    justify-content: center;
  }
  section.takeover h1 {
    color: #ffffff;
    font-size: 52px;
  }
  h1 { color: #0057b8; }
  h2 { color: #444; font-size: 26px; margin-top: 0; }
  table { font-size: 22px; width: 100%; }
  th { background: #0057b8; color: white; }
  code { background: #f0f4f8; padding: 2px 6px; border-radius: 3px; }
  pre { background: #f0f4f8; padding: 16px; border-radius: 6px; font-size: 18px; }
---

<!-- _class: cover -->

# Feeding GenAI
## Data Sources, Serialization and Other Tricks

Lessons from building a retail market analytics platform

---

# Agenda

1. **The Platform** — What We Built
2. **Semantic Layer** — Databricks Metric Views
3. **Serialization** — TOON vs JSON
4. **Structured Output** — Controlling Report Structure
5. **Key Takeaways**

---

# What We Built

- Retail market analytics platform
- Aggregates business metrics across multiple dimensions
- Generates natural-language reports powered by LLMs

![bg right:40%](img/architecture.png)

**Three unexpected problems — three elegant solutions**

---

<!-- _class: section-header -->

<span class="number">01</span>

# Semantic Layer

Define business metrics once, use everywhere

---

# Every Team Writes Their Own Metrics

- Business metrics must be aggregated across many dimensions
- Classic approach: custom SQL per query, per application
- Metric definitions **diverge** between apps over time
- A logic change must be replicated everywhere

```sql
-- App A
SELECT SUM(revenue) / NULLIF(SUM(units), 0) AS avg_price ...

-- App B (slightly different)
SELECT AVG(unit_price) AS avg_price ...
```

> Fragile and expensive to maintain

---

# Define Once, Use Everywhere

Databricks **Metric Views** — define business metrics in YAML

```yaml
metrics:
  - name: avg_selling_price
    label: Average Selling Price
    type: ratio
    numerator: revenue
    denominator: units_sold
```

- Platform generates SQL with the `MEASURE` aggregate function
- Single source of truth across all consumers
- Non-transitive expressions (averages, ratios) **work seamlessly**

---

<!-- _class: demo -->

# Demo — Metric Views in Action

1. Show original hand-crafted SQL
2. Show Metric View YAML definition
3. Show derived SQL with `MEASURE` function

**Key moment:** Highlight that weighted averages and ratios aggregate correctly across dimensions — without special casing

---

<!-- _class: section-header -->

<span class="number">02</span>

# Serialization

Respecting the token budget

---

# Text Is Still the Language of LLMs

- Even multi-modal models need structured data **as text**
- A dozen datasets to include in each prompt
- JSON is the default — but is it the right choice?
- Every token costs money and competes for the model's attention

![bg right:35%](img/token-budget.png)

---

# JSON Has Terrible Token Efficiency

Every row repeats all attribute names:

```json
[
  {"product": "Milk", "region": "North", "units": 120, "revenue": 240.0},
  {"product": "Milk", "region": "South", "units": 98,  "revenue": 196.0},
  {"product": "Bread","region": "North", "units": 200, "revenue": 180.0}
]
```

- Structure adds noise: `{ } " : ,`
- A 100-row dataset can be **70% structural repetition**
- LLMs spend attention budget on format, not content

---

# TOON: Compact, Readable, Token-Efficient

Same data in TOON format:

```
product  | region | units | revenue
Milk     | North  |   120 |  240.0
Milk     | South  |    98 |  196.0
Bread    | North  |   200 |  180.0
```

- Header row defines field names — like a CSV
- Rows are compact value sequences
- Typed, structured, and human-readable
- **~70% smaller** than equivalent JSON

---

<!-- _class: demo -->

# Demo — JSON vs TOON

1. Show original JSON file — attribute repetition highlighted
2. Show TOON-formatted equivalent — concise and scannable
3. Show file size comparison

---

**Bonus story:**

The `toons` Python library wraps high-performance **Rust** code.

It had a bug: incorrect serialization of certain Unicode characters.

> Fixed it with Claude — without knowing Rust.

*GenAI helped fix the tool that feeds GenAI.*

---

<!-- _class: section-header -->

<span class="number">03</span>

# Structured Output

LLM for reasoning, code for presentation

---

# "Can You Move That Section?"

The report generated *Conclusions and Insights* at the end.

**Customer:** "We'd like it right after the overview."

Sounds trivial — it wasn't.

![bg right:40%](img/report-structure.png)

---

# How Would You Solve This?

| Approach | Drawback |
|---|---|
| Change the system prompt | Conclusions written before findings — **less insightful** |
| Two-pass generation | Slower and **more expensive** |
| Text / Markdown parsing | LLM output is unpredictable — **brittle** |

---

# Ask for JSON, Reorder in Code

Use **structured output** — generate the report as JSON:

```json
{
  "title": "...",
  "overview": "...",
  "detailed_report": "...",
  "conclusions_and_insights": "..."
}
```

Then serialize to markdown **in the desired order** in application code.

---

# Why This Works Better

- Sections always reliably separated — no parsing heuristics
- Conclusion quality **preserved** (written after full report generation)
- Zero performance penalty — single LLM call
- Order is a **presentation** concern, not a generation concern

---

# What These Have in Common

| Trick | Principle |
|---|---|
| Semantic Layer | Move business logic into a governed layer |
| TOON | Respect the token budget — format is not neutral |
| Structured Output | LLM for reasoning, code for presentation order |

**Theme:** Separate concerns.
The LLM is for reasoning, not for formatting or metric definition.

---

# Things to Take Home

- Define metrics once — use a semantic layer
- Audit your serialization — JSON is often the wrong choice for LLM input
- Use structured output whenever you need control over output structure
- Claude can fix Rust bugs even if you can't read Rust

---

<!-- _class: takeover -->

# Questions?

---

*Thank you*

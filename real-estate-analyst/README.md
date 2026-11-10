# Real Estate Investment Analyst

Takes a set of property search criteria, pulls matching listings from a
property API, runs full investment maths on each one, and emails a ranked
report.

---

## The problem

Property listing sites tell you the asking price. They don't tell you
whether the property makes money.

Working that out means, for every listing: mortgage payment at current
rates, property tax, insurance, maintenance reserve, then rent minus all
of it, then cap rate and cash-on-cash return. That's a spreadsheet
exercise per property, and it's why most people compare on price and
bedrooms instead.

---

## How it works

```
Form submission (location, price range, beds, baths, sort, multi-family)
    ↓
Zillow API  →  matching listings
    ↓
Split Out  →  one item per property
    ↓
Code node  →  full investment calculation per property
    ↓
Google Sheets (append or update)
    ↓
Aggregate  →  single payload
    ↓
AI Agent (+ Calculator tool, + Gmail tool)
    ↓
Ranked HTML report, emailed
```

---

## Design notes

### The maths is in code, not in the model

This is the decision I'd defend hardest.

An LLM asked to calculate a mortgage payment will produce a number that
looks right. Sometimes it is. The amortisation formula is not something
to approximate:

```javascript
const downPaymentPct     = 0.20;
const closingCostPct     = 0.03;
const annualInterestRate = 0.05;
const loanTermYears      = 30;
const propertyTaxRate    = 0.012;
const insuranceRate      = 0.003;
const maintenanceRate    = 0.01;
```

The Code node derives, per property: down payment, closing costs, loan
amount, monthly mortgage payment, monthly tax, insurance, maintenance,
total monthly expenses, monthly and annual cash flow, cap rate and
cash-on-cash ROI.

Deterministic, auditable, identical every run.

The model's job is what models are good at — ranking, summarising,
spotting the properties with negative cash flow, and writing it up
readably. It never computes anything load-bearing.

### Assumptions at the top, in one block

Interest rate, down payment, tax rate and maintenance reserve are
constants in a labelled block at the top of the Code node rather than
scattered through the calculation.

Rates change. When they do, the edit is one line in an obvious place
instead of a hunt through the file.

### Sheets before the model

Results are written to Google Sheets *before* the AI step, using append-
or-update rather than append. So the run is recorded whether or not the
report generates, and re-running the same search updates rows instead of
duplicating them.

### Agent with tools, not a plain chat model

The AI Agent has a Calculator tool and a Gmail tool attached. It computes
portfolio-wide averages via the calculator rather than doing arithmetic
in the completion, and sends the report itself rather than passing text
to a separate send node.

---

## Stack

n8n · Zillow API (RapidAPI) · OpenAI · Google Sheets · Gmail ·
JavaScript

## Files

- `workflow.json` — full export, credentials by reference
- `investment-calc.js` — the calculation logic from the Code node

---

## Note

Built as part of structured n8n training. The separation of concerns —
deterministic financial maths in code, interpretation and presentation in
the model — is the pattern I'd reuse anywhere numbers have to be right.

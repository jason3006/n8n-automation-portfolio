# n8n Automation Portfolio

**Jason Emefiele** — Computer Engineering graduate building workflow
automation in n8n.

I build systems that connect APIs, move data between services, and run
without supervision. Most of what I've learned has come from things
breaking in ways that didn't throw an error.

Each project below includes what it does, how it's built, and — the part
I think matters most — what went wrong and how I found it.

📍 Calabar, Nigeria · Open to remote (WAT / UTC+1)
📧 jasonemefiele22@gmail.com · [LinkedIn](https://linkedin.com/in/jason-emefiele)

---

## Projects

### [Multi-Timeframe Trading Workflow](./trading-workflow)
Pulls market data across four timeframes from a REST API, routes it
through an AI agent for analysis, and executes via a broker API with
OAuth session handling, duplicate-position checks and risk-based
position sizing written in JavaScript.

**Stack:** n8n · Twelve Data API · Capital.com REST API · Anthropic
Claude via OpenRouter · JavaScript

**What it taught me:** four separate bugs that produced wrong results
without ever throwing an error. Write-up in the project README.

---

### [Internal Correspondence Drafting Assistant](./drafting-assistant)
A form-based AI assistant for a manager who spends hours drafting formal
internal correspondence. Built around a hard constraint: it could not
touch the organisation's email system, and sensitive data could not leave
their network.

**Stack:** n8n · Form Trigger · Anthropic Claude via OpenRouter

**What it taught me:** the interesting part of this build was what I
chose *not* to automate.

---

### [Rate Logger — Diagnostic Case Study](./rate-logger-diagnostic)
A scheduled workflow logging exchange rates that wrote wrong values for
five days while reporting success on every run. Full diagnostic: symptom,
hypothesis, node-by-node trace, root cause, fix and verification.

**Stack:** n8n · Frankfurter API · JavaScript

**What it taught me:** silent failures cost more than crashes. A workflow
that stops gets fixed the same morning.

---

### [AI Invoice Processing Pipeline](./invoice-processing)
Watches an inbox for PDF attachments, classifies whether each is actually
an invoice before extracting anything, pulls structured data, and files it
in Notion with a link back to the stored document.

**Stack:** n8n · Gmail · Google Drive · OpenAI · Notion

**Design decision worth naming:** a cheap classifier gate runs before the
expensive extraction, because a model asked to extract invoice fields from
a newsletter will invent plausible ones.

---

### [Real Estate Investment Analyst](./real-estate-analyst)
Pulls property listings matching search criteria, runs full investment
maths on each — mortgage, tax, insurance, maintenance, cap rate,
cash-on-cash ROI — and emails a ranked report.

**Stack:** n8n · Zillow API · OpenAI · Google Sheets · Gmail · JavaScript

**Design decision worth naming:** every number is calculated in code, not
by the model. The model ranks and writes; it never computes anything
load-bearing.

---

### [Business Ideas Research Agent](./business-ideas-agent)
Mines Reddit for recurring problems and turns them into structured
business ideas. Runs scheduled across fixed communities, and on demand
against any subreddit and keyword.

**Stack:** n8n · Reddit API · OpenAI · Google Sheets · Gmail

**Design decision worth naming:** posts are normalised down to title and
body before merging, then sent as one batched LLM call rather than one per
post — cheaper, and the model can see repetition across communities.

---

## Background

Completed 100+ guided builds in n8n covering AI agents, RAG pipelines,
voice agents, document processing, API integration and workflow
orchestration.

Certified in n8n AI Agents & Automations, and in Prompt Engineering for
Large Language Models.

Two industrial internships in IT infrastructure and cybersecurity —
security alert triage across a 200+ endpoint network at Lafarge Africa,
and IT support for 100+ staff at Flour Mills of Nigeria.

---

## A note on the workflow files

Every exported JSON in this repository has been stripped of credentials.
Where a workflow needs an API key, the node references a credential by
name rather than value.

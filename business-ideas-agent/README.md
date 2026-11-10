# Business Ideas Research Agent

Mines Reddit for problems people are complaining about, and turns them
into structured business ideas. Runs two ways: on a schedule across fixed
communities, and on demand against any subreddit and keyword.

---

## The problem

Reddit is full of people describing problems they'd pay to have solved.
It's also full of noise, and reading four communities daily to find the
signal is not a good use of anyone's morning.

Two different needs, though:

- **Standing research** — what's coming up repeatedly in the
  entrepreneurship communities?
- **Targeted research** — I have a specific market in mind; what are
  people in *that* community complaining about?

---

## How it works

Two entry points into the same idea:

### Path 1 — Scheduled

```
Schedule Trigger
    ↓
4 × Reddit nodes  →  hot posts from 4 subreddits, 20 each
    ↓
4 × Set nodes  →  normalise to title + selftext
    ↓
Merge  →  Aggregate  →  single array
    ↓
LLM  →  extract startup ideas as structured JSON
    ↓
Split Out  →  Append rows to Google Sheets
```

### Path 2 — On demand

```
Form (subreddit + keywords)
    ↓
Reddit search
    ↓
Set  →  Aggregate
    ↓
LLM  →  pain points + matching ideas, as an HTML email
    ↓
Gmail
```

---

## Design notes

### Fan out, normalise, fan in

Four Reddit nodes run in parallel, each with its own Set node, before a
Merge and an Aggregate.

The Set nodes matter. They strip each post down to `title` and
`selftext` before merging. Without that, four subreddits' worth of full
Reddit objects — scores, awards, flair, thumbnails, dozens of fields —
go into the prompt, and almost none of it is signal.

Then everything collapses into **one LLM call** rather than one per
post. Eighty posts in a single request is both cheaper and better, since
the model can see repetition across communities that it couldn't see
post by post.

### A schema that captures evidence, not just ideas

The extraction prompt doesn't only ask for ideas. It asks whether the
author said the idea *worked*, and for the exact phrase if so:

```
- startupIdea: one-sentence summary
- success: true if the author says it succeeded
- successDetails: the exact phrase or metric if success=true, else null
- goodIdea: your assessment, based on market saturation and viability
- pros / cons: one sentence each
```

`successDetails` is the useful field. "I made $10k in three months" is
evidence. An idea with no evidence attached is just an idea, and the
schema keeps them distinguishable.

### Two outputs for two use cases

The scheduled path writes to Sheets — it's building a dataset over time,
and you want to scan and sort it.

The on-demand path emails HTML — you asked a question, you want an
answer you can read now.

Same extraction, different destination, chosen by what the output is
for.

---

## Stack

n8n · Reddit API · OpenAI · Google Sheets · Gmail

## Files

- `workflow.json` — full export, credentials by reference

---

## Note

Built as part of structured n8n training. The dual-trigger pattern —
scheduled accumulation into a dataset, plus on-demand query into a
report — is the part I'd reuse.

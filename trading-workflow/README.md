# Multi-Timeframe Trading Workflow

An n8n workflow that pulls market data across four timeframes, routes it
through an AI agent for analysis, and places orders through a broker's
REST API.

> **Note:** this is an engineering project, not a trading system I'm
> recommending. It's here for the API integration, error handling and
> state management — not for its results. It has no verified track record
> and I make no claim about profitability.

---

## The problem

I wanted to build something that exercised every hard part of workflow
automation at once: multiple external APIs, authenticated sessions,
rate limits, conditional branching, state that persists between runs,
and a failure mode where a wrong answer costs real money rather than
just looking wrong.

Market data turned out to be a good vehicle for that. The APIs are
public, the data is messy in realistic ways, and the consequences of a
silent bug are immediate.

---

## How it works

```
Schedule Trigger (15 min)
    ↓
Loop over 5 instruments
    ↓
4 × HTTP Request  →  15m / 1h / 4h / 1day candles
    ↓
Code node  →  calculate RSI, MACD, EMA, ATR per timeframe
    ↓
Aggregate  →  single payload per instrument
    ↓
AI Agent  →  structured BUY / SELL / HOLD with entry, stop, target
    ↓
Code node  →  parse output, compute R:R
    ↓
IF gate  →  action, R:R threshold, trading session, no warnings
    ↓
Broker auth  →  market rules  →  duplicate position check
    ↓
Code node  →  risk-based position sizing
    ↓
POST order
```

**Design decisions worth naming:**

- Indicators are calculated in the workflow rather than requested from an
  API, so the model receives the same derived features every run.
- The AI agent is gated, not trusted. Its output passes through a parser,
  an R:R threshold, a session filter and a sizing validator before
  anything reaches the broker.
- The sizing node can refuse. If the model returns incoherent levels, or
  price has moved too far since the decision, it throws rather than
  adjusting the trade to fit.

---

## The hard part

Four bugs. None of them threw an error. All four produced plausible
output while being wrong.

### 1. One timeframe was returning the same instrument every time

The 4h data node used:

```javascript
$('The 5 pairs').first().json.ticker
```

`.first()` returns the first item in the loop, not the current one. So
every instrument's 4-hour data was actually the *first* instrument's.

The prompt's first instruction was to establish higher-timeframe bias
from the daily and 4h candles. That bias was wrong for four of five
instruments, and had been since the workflow was built.

Found it by comparing `meta.symbol` in each timeframe node's output
across two consecutive loop iterations. Three changed, one didn't.

### 2. The duplicate-position check never matched

The workflow checked for an existing open position before placing a new
one, filtering on:

```javascript
p.position.epic === currentInstrument
```

The broker returns each position as two sibling objects — `position` and
`market` — and the instrument identifier lives under `market.epic`.
`position.epic` was `undefined`. Comparing `undefined` to anything
returns false, the filter returned an empty array, and the "no existing
position" condition passed every time.

The workflow opened a second position on an instrument that already had
one. I found it by reading the actual API response rather than the schema
I assumed.

### 3. `parseFloat` truncating at the thousands separator

Prices above 1,000 were being logged as two-digit numbers. The code
formatted a value for display, then parsed the formatted string back into
a number:

```javascript
display: rate.toLocaleString()      // "18,148.00"
value:   parseFloat(display)         // 18
```

`parseFloat` reads left to right and stops at the first character it
can't interpret. The comma. It returns `18` and doesn't complain — a
partial parse is a valid parse as far as JavaScript is concerned.

The fix wasn't to strip the comma. It was to delete the round trip
entirely and take the raw number from the API response. Stripping the
separator leaves the same class of bug in place for the next locale or
display format.

### 4. Position sizing wasn't risk-based

The original sizing allocated a percentage of available funds as margin.
That meant a trade with a tight stop and a trade with a wide stop got the
same position size — so the actual loss when each stop hit varied by a
factor of several. With inconsistent risk per trade, no amount of data
would have told me whether the signals were any good.

Rewritten to size from the stop distance:

```javascript
const stopDistance = Math.abs(price - stopLevel);
const riskAmount   = available * RISK_PCT;
let   size         = riskAmount / (stopDistance * quoteUsd);
```

Plus a margin ceiling as a safety net, and an honest `actualRiskPct` in
the output so rounding and broker minimums are visible rather than
hidden.

---

## What I'd do differently

**Log everything from day one.** I made four significant changes before I
had any record of what the system was producing. Without a trade log,
every change after the first is guesswork — you can't tell improvement
from noise.

**Check the API response shape, don't assume it.** Two of the four bugs
came from writing code against the structure I expected rather than the
one the API returned.

**Treat "no error" as meaningless.** Every bug here ran green. The
workflow's own success state told me nothing. Verification has to compare
output against an independent source.

---

## Stack

n8n · Twelve Data API · Capital.com REST API · Anthropic Claude via
OpenRouter · JavaScript

## Files

- `workflow.json` — the full export, credentials stripped
- `sizing-node.js` — the position sizing and validation logic

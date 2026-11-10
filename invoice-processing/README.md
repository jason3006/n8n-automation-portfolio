# AI Invoice Processing Pipeline

Watches an inbox for PDF attachments, decides whether each one is
actually an invoice, extracts the structured data, and files it in a
Notion database with a link back to the stored document.

---

## The problem

Invoices arrive as email attachments. Someone opens each one, reads off
the vendor, the total and the line items, types them into a tracker, and
files the PDF somewhere.

It's repetitive, it's slow, and it's the kind of work where a typo in a
total doesn't get caught until reconciliation.

But the inbox doesn't only contain invoices. It contains receipts,
contracts, newsletters with PDF attachments, and statements. A pipeline
that assumes every PDF is an invoice fills the database with garbage.

---

## How it works

```
Gmail Trigger (every minute, filter: filename:pdf has:attachment)
    ↓
Upload attachment to Google Drive
    ↓
Download + Extract text from PDF
    ↓
Invoice Controller (LLM)  →  "is this an invoice?"  →  yes / no
    ↓
IF gate
    ├── no   →  delete the uploaded file, stop
    └── yes  →  continue
                    ↓
            Basic LLM Chain + Structured Output Parser
                    ↓
            Create page in Notion Invoice Tracking Database
```

---

## Design notes

### Two LLM stages, not one

The obvious build sends every PDF straight to an extraction prompt and
hopes the model returns null for non-invoices. That doesn't work well —
models are obliging, and asked to extract invoice fields from a
newsletter, they will often invent plausible ones.

So there's a classifier first:

```
Based on the text decide if the text is about an invoice or not.
If yes return "yes", if no return "no".
Don't provide any additional information. Use only small letters.
```

A single-word answer, cheap, deterministic, and it feeds a hard IF gate.
Extraction only runs on documents that cleared it.

The constraint on output format matters. "Don't provide any additional
information. Use only small letters" exists because an LLM answering
"Yes, this appears to be an invoice" breaks a string equality check
against `"yes"`.

### Cleanup on the rejection path

Non-invoices don't just stop — the file that was already uploaded to
Drive gets deleted. Without that, the pipeline accumulates every PDF
that ever passed through the inbox.

Easy to miss. The happy path is the one you test.

### Structured output rather than free text

Extraction uses a Structured Output Parser with an explicit JSON schema:

```json
{
  "invoice_name": "Invoice ABC-123",
  "company_name": "Acme Corp",
  "total_invoice_amount": 1000,
  "line_items": [
    { "description": "Consulting services", "amount": 100 }
  ]
}
```

The prompt also instructs the model to check that line items sum to the
total, allowing for rounding. That's a validation step the model can
perform on its own output before returning it.

### Drive link, not a copy

The Notion page stores `webViewLink` from the Drive upload rather than
the file itself. The record points at the document; the document lives in
one place.

---

## Stack

n8n · Gmail API · Google Drive API · OpenAI · Notion API

## Files

- `workflow.json` — full export, credentials by reference, database IDs
  removed

---

## Note

Built as part of structured n8n training. The architecture above — the
classifier gate, the rejection cleanup, the structured schema — is where
the engineering decisions sit, and it's what I'd carry into a production
version.

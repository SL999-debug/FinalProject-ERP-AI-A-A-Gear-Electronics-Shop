# 01 — Architecture

## The one-paragraph version

Airtable holds the data, n8n holds all the logic and the credentials, and the two HTML files
are thin clients that call n8n over HTTPS. Nothing is served from a machine of our own, which
is why there is no build step, no package manager and no server to deploy.

## The three-layer split, and why

**Data — Airtable.** Six tables. Airtable is a spreadsheet that is also a REST API, which
means the data is inspectable by a human without any work from us. That property is worth more
in a course project than normalisation would be.

**Logic — n8n.** Every read and every write goes through a workflow. The HTML never touches
Airtable directly and holds no credentials of its own. This is the single most important
decision in the project: it means a leak of the front end leaks no data access, because the
front end has none.

**Presentation — two HTML files.** `aa-gear-landing.html` for the public site, and
`aa-gear-admin.html` for the internal dashboard. Each is a single file with its CSS and
JavaScript inline. Both are Hebrew and right-to-left.

## The request paths

### Landing page → lead

```
visitor submits the form
  → POST  /webhook/ErpAILids          (WF-3)
      → email already in Airtable?  → plain-text "already exists"  → stop
      → otherwise                    → create the Lead              → "created"
  → lands in the Airtable Leads table
  → WF-4a may later e-mail that lead; WF-4b catches the reply
```

WF-3 does its own dedupe, on purpose. The database has no unique constraint on `Email`, and
n8n has no transaction — a check-then-create is the honest way to express "don't double-submit"
here. It is still racy under simultaneous submissions, which is noted in
[09-known-issues.md](09-known-issues.md).

### Dashboard → data

```
admin opens aa-gear-admin.html
  → loads credentials and table schemas from the app's own config
  → any read or write becomes one POST:

        { "table": "products", "action": "read" | "create" | "update" | "delete",
          "data": { ... }, "filter": {...} }

  → WF-13 switches on `action` and talks to the matching Airtable table
  → the table name comes from the request, so WF-13 is table-agnostic
```

One webhook serving six tables is what keeps the front end small: the dashboard has a single
`api()` helper, and the shape of the request is the same for a customer edit and an invoice
delete.

The `action: "chat"` branch is different in kind — it is a RAG question, answered from the
policies and product collections, and it does not write anything.

### Invoice → PDF → back

```
row appears in invoices
  → WF-1 (polls every minute) checks amount > 0, issue date, customer id, number
      invalid → create a Task describing what is wrong, for the team to fix
      valid   → set Status = Pending
  → WF-8 (polls every 59s) picks up Pending documents
      → switch on InvoiceType:  חשבונית / חשבונית מס / קבלה
      → render the matching Hebrew HTML template
      → community PDF node → PDF
      → upload to Google Drive
      → write the Drive link back onto the invoice record
```

WF-1 and WF-8 are two workflows, not one, because they are two different jobs: one is a
business rule, the other is a rendering task with a slow external dependency. Keeping them apart
means a Drive outage does not stop invoices being validated, and vice versa.

The 59-second poll is a deliberate, slightly ugly choice: it keeps the two jobs naturally
staggered so WF-8 rarely sees a document in the same second WF-1 wrote it. See
[09-known-issues.md](09-known-issues.md#7-invoice-numbering-can-race) for why that is not
sufficient.

## Why n8n and not a real backend

The honest list:

- **For** — the project is graded on the AI/automation content, not on serving HTTP. n8n
  gives us public HTTPS webhooks, cron, retries, a credential store and error views for free.
- **Against** — no transactions, no real concurrency control, no testability, and the
  integration surface is only as good as the community nodes. WF-8's dependency is the clearest
  example: a paid community node stands between the project and a working PDF.

Knowing that trade-off, rather than having discovered it, is the point of writing this file.

## What the AI agents add to this

The two agents (WF-5 customer, WF-9 director) are consumers of the same data, not new
producers. WF-5 answers from two Qdrant collections and remembers the conversation in MongoDB
Atlas; WF-9 answers only for one Telegram account and then does arithmetic over the invoices
table. The separation of concerns is the *tool selection*: WF-5 gets retrieval tools, WF-9 gets
a guard and a query. See [05-rag-pipeline.md](05-rag-pipeline.md) and
[06-ai-agents.md](06-ai-agents.md).

## Where each piece is documented

| Concern | File |
|---|---|
| Tables and fields | [02-airtable-base.md](02-airtable-base.md) |
| Each of the ten workflows | [03-workflows.md](03-workflows.md) |
| The dashboard's behaviour | [04-dashboard.md](04-dashboard.md) |
| Embeddings, chunking, Qdrant | [05-rag-pipeline.md](05-rag-pipeline.md) |
| Prompts, tools, memory | [06-ai-agents.md](06-ai-agents.md) |
| Israeli tax documents | [07-document-pdf.md](07-document-pdf.md) |
| Building it from nothing | [08-setup-runbook.md](08-setup-runbook.md) |
| What is still broken | [09-known-issues.md](09-known-issues.md) |

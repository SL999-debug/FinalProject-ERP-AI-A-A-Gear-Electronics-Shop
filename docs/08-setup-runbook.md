# 08 — Setup runbook

Ordered, so you can build this from nothing. Budget an afternoon for the accounts and a day for
the workflows.

## 0. Accounts

| Service | Why | Free tier? |
|---|---|---|
| **n8n Cloud** | all automation, and the public HTTPS webhook URLs | yes, with a node/execution limit |
| **Airtable** | the database | yes |
| **Qdrant Cloud** | the two vector collections | yes, small |
| **MongoDB Atlas** | agent chat memory | yes |
| **Telegram** | two bots | free |
| **Google** | Gmail (WF-4a/4b) and Drive (WF-8) | yes, with OAuth setup |
| **An OpenAI-compatible provider** | chat **and** embeddings | varies |

Note the last row: WF-5 and WF-6/WF-7 need both a chat model and an embedding model, and the
n8n credentials for them are separate. A chat-only key will not work for the Qdrant ingestion.

⚠️ **Before anything else:** read [SECURITY.md](../SECURITY.md). The workflows in this
repository point at a live system. If you are rebuilding rather than cloning that system, build
against your own URLs from the start, because retrofitting is where credentials end up in
commits.

## 1. The Airtable base

Create these six tables, with these field names. They are the ones the dashboard reads and
writes — a different name means a silently unmapped field, not an error.

Use [`../schema/schema.json`](../schema/schema.json) as the checklist. It is derived from the
application code, so it is exactly the contract.

**`products`**
| Field | Type | Note |
|---|---|---|
| `Name` | text | primary |
| `SKU` | text | unique in practice |
| `Category` | text or single select | |
| `Price` | currency | gross, VAT included |
| `InStock` | **number** | ⚠️ the current base has this as *text*, which breaks saving — see [09](09-known-issues.md#6-instock-is-text-and-breaks-saving) |
| `Description` | long text | rendered in its own row in the dashboard |
| `imageUrl` | url | collected by the form but **not mapped** in the app today |

**`customers`**
| Field | Type | Note |
|---|---|---|
| `Name` | text | |
| `Mail` | email | |
| `Phone` | phone | |
| `CustomerId` | text | the `CUST000003` business key. **Not** the Airtable record id. |

**`leads`** — `Name`, `Email`, `Telephone`, `Status`, `Interested`

**`orders`** — `number`, `customer`, `items`, `total`, `status`, `date`, `notes`.
⚠️ All of these are currently **unmapped in the app**, so nothing you create here is saved.
Build the columns anyway; the fix is in [09](09-known-issues.md#3-the-orders-screen-saves-nothing).

**`invoices`** — `number`, `customer`, `date`, `total`, `type`, `notes`, plus the fields WF-8
writes: `Status` (single select: `Pending` / `Done`), `InvoiceType` (single select:
`חשבונית` / `חשבונית מס` / `קבלה`), and a Drive link field.

**`tasks`** — `Title`, `Level` (select: low/medium/high), `Status`, plus the customer and
invoice reference fields.

Fill `products` with roughly 20 real items and `customers` with a handful. A demo with three
rows does not exercise pagination, search, or the empty state.

## 2. n8n credentials

One credential per service, in the n8n instance: Airtable, Qdrant, MongoDB, OpenAI-compatible
chat, OpenAI-compatible embeddings, Telegram, Google OAuth, and Gmail.

⚠️ **Do not import the JSON in `workflows/` into an instance that already holds these
credentials.** The exports carry credential *id* references, and n8n will happily re-link them
to whatever those ids resolve to in the target instance. Either import into a fresh instance and
attach credentials manually, or strip the `credentials` blocks first. See
[SECURITY.md](../SECURITY.md#recommended-public-version-if-this-ever-becomes-a-portfolio-repo).

## 3. Import the workflows

Import in this order, because of the dependencies:

```
WF-6  policy embeddings      ┐
WF-7  product embeddings     ┘ ingest first, the agents need something to retrieve
WF-13 dashboard API          the dashboard cannot work without it
WF-3  lead webhook           the landing page
WF-1  invoice validation  →  WF-8  PDF generation
WF-4a cold mail  →  WF-4b  reply watch
WF-5  customer agent
WF-9  director agent
```

Import one, open it, attach its credentials, and check every node for an empty credential
dropdown. Then **activate** it. Remember that the exports in this repository are almost all
switched off — see [09](09-known-issues.md#1-only-wf-13-is-active).

## 4. The two environment edits nobody remembers

- **WF-13's webhook path.** Take the production URL n8n gives you when you activate, and put
  it in the dashboard's `CONFIG.webhook`.
- **The landing page's URL.** It ships pointing at `…/webhook-test/…`, which only works while
  the n8n editor is open. Change it to `…/webhook/…`. This is the single most common reason
  "the contact form does nothing".

## 5. Load the vector stores

Triggers here are n8n **form** triggers, so:

1. activate WF-6 and WF-7;
2. open the form URL WF-6 provides;
3. submit a policy document — the shop's warranty/returns/shipping text, in Hebrew;
4. submit the product export to WF-7's form;
5. confirm both collections have points in Qdrant, and that the points have a sensible count
   and dimension.

If the chat agent answers confidently and wrongly, this is the first thing to check: an empty
collection makes a grounded agent fall back to its own memory.

## 6. Wire the Telegram bots

- Two bots, one per agent, via BotFather. **WF-5 and WF-9 must be different bots** — the point
  of the director agent is that it is not the public one.
- WF-9 needs the owner's chat id as a constant. Get it by messaging the bot, or by reading it
  off a Webhook node's output.
- Test the refusal first: message WF-9 from a second account *before* configuring the owner
  check, confirm it answers, then set the id and confirm it refuses. That order proves the gate
  is doing something.

## 7. Drive and Gmail

- **Gmail (WF-4a/4b):** the Google OAuth credential needs Gmail read *and* send. WF-4b polls
  for replies, so it needs a mailbox that actually receives cold-mail replies — in testing,
  that means sending to a real mailbox you control.
- **Drive (WF-8):** connect Drive, and confirm the folder WF-8 uploads into. Check the sharing
  scope of what it creates; see [07](07-document-pdf.md#the-drive-link).

## 8. The dashboard

```
1. open app/aa-gear-admin.html from file:// — it loads with no server
2. set CONFIG.webhook to your WF-13 production URL
3. each table should load
4. try: search → sort → expand a product → edit → save → delete
5. check the browser console is clean
```

`CONFIG.productImg` points at a public image CDN for product images. It is optional: the
dashboard falls back to a generated initial-letter avatar, which is what happens today anyway
because `imageUrl` is unmapped.

## 9. A demo script that actually shows the work

1. Landing page: submit a lead, show the Airtable row, submit the same email again, show the
   "already exists" answer. *This is WF-3's dedupe and it is the most reliable demo in the
   project.*
2. Dashboard: open the products table, expand a description (the badge does not move), search
   "samsung", then click **edit** — which works, because that bug was fixed.
3. WF-5 on Telegram: ask a warranty question, then ask something the policies do not cover, and
   show the refusal. *The refusal is the impressive half.*
4. WF-9 from a non-owner account: show the refusal. Then from the owner account: show the debt
   report.
5. Create an invoice in Airtable with a missing field, wait a minute, show the task WF-1
   created. Then fix it, and show the PDF appearing in Drive and the link in the dashboard.

## 10. Before you call it done

- [ ] all ten workflows **active**, not just imported
- [ ] the landing page posts to `/webhook/`, not `/webhook-test/`
- [ ] both Qdrant collections have points
- [ ] WF-8's PDF node is actually installed and runs
- [ ] dashboard tested at a width under 420px
- [ ] browser console clean
- [ ] no live webhook URLs or credential ids in a public copy of the repo

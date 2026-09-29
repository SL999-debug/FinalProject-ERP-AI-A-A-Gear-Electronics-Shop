# 09 — Known issues and open problems

Every item here was found by reading the workflows and running the dashboard, not by guessing.
They are listed with the evidence, the impact, and the fix — including the ones that were fixed
during development, because a bug that was fixed is still worth knowing about.

Severity: **High** breaks a feature outright · **Medium** produces wrong or fragile behaviour ·
**Low** is a wart.

---

## 1. Only WF-13 is active {#1-only-wf-13-is-active}

**Severity: High** — nine of ten workflows are switched off in the exported JSON.

`WF-13 Dashboard.json` is the only file with `active: true`. The other nine export as
`active: false`, so as committed the system does nothing except serve the dashboard. Lead
capture, validation, PDF generation and both agents are all dormant.

Caveat: the flag records the state at export time and may not reflect the live n8n instance.
Check the instance before assuming the system is down.

**Fix:** activate each workflow, in the order in
[08-setup-runbook.md](08-setup-runbook.md#3-import-the-workflows), after attaching its
credentials.

---

## 2. The landing page posts to a test webhook {#2-the-landing-page-posts-to-a-test-webhook}

**Severity: High** — the contact form silently fails in production.

`app/aa-gear-landing Final.html` posts to:

```
https://gl00001.oph.st/webhook-test/ErpAILids     ← -test-
```

A `/webhook-test/` URL only resolves while the n8n editor is open and the workflow is being
edited. With the editor closed, the request gets a 404 and the visitor sees a form that does
nothing. It works perfectly in a demo, which is what makes it likely to survive.

**Fix:** change it to `https://gl00001.oph.st/webhook/ErpAILids` — the live path. One string.

---

## 3. The orders screen saves nothing {#3-the-orders-screen-saves-nothing}

**Severity: High** — every field on `orders` is collected and discarded.

All seven form fields — `number`, `customer`, `items`, `total`, `status`, `date`, `notes` —
have no Airtable mapping in `SCHEMAS`. The form renders, accepts input, validates, and posts a
payload that the server then ignores. Creating an order produces a row with every cell empty,
and no error is shown.

`schema.json` records this honestly: every one of those fields has `airtableField: null`.

**Fix:** add the `server:` mapping to each `orders` field in `SCHEMAS` once the column names
are confirmed against the live base, then regenerate `schema/schema.json`. Note that
`orders.customer` should almost certainly be a **select of `customers.CustomerId`**, the same
fix already applied to invoices.

---

## 4. The invoice form writes a name into a code field {#4-the-invoice-form-writes-a-name-into-a-code-field}

**Severity: High** — invoices silently fail to link to customers.

The invoice form asked the user for a customer **name** and posted it into `CustomerId`, which
holds a business key like `CUST000003`. Airtable accepted the write, so the invoice looked
fine, and it simply never matched a customer.

Partly fixed: the dashboard now has a `customers.CustomerId` select on the invoice form. The
label still says "customer name", which is now misleading, and the underlying mismatch is
recorded here.

**Fix:** finish the job — retitle the field to the customer key, and confirm that WF-1's
"customer id present" check validates against an actual customer rather than a non-empty
string.

---

## 5. `InStock` is text and breaks saving {#6-instock-is-text-and-breaks-saving}

**Severity: High** — product saves return HTTP 500.

`products.InStock` is a text field in the base, and the form sends a number. Airtable rejects
the type mismatch, the save fails, and the dashboard surfaces a 500 rather than a useful
message.

**Fix:** change the Airtable field type to number, and coerce on write so a string can never be
sent. Numeric stock also means the dashboard can sum it, which it currently cannot.

---

## 6. WF-8 cannot be installed on n8n Cloud's free plan {#5-wf8-needs-a-paid-community-node}

**Severity: High** — the PDF feature cannot run where the project is hosted.

WF-8 depends on `@custom-js/n8n-nodes-pdf-toolkit-v2`, a community node that n8n Cloud's free
plan does not allow. On that plan the workflow will not import cleanly and no PDF is ever
produced.

**Fix:** upgrade the plan, self-host n8n, or replace the node with HTML-to-PDF inside a Function
node. The last option removes the dependency permanently. Detail in
[07-document-pdf.md](07-document-pdf.md#the-hard-dependency).

---

## 7. Invoice numbering can race {#7-invoice-numbering-can-race}

**Severity: Medium** — two invoices can be issued the same number.

Numbers are computed as `count + 1`. With WF-8 polling every 59 seconds, two invoices created
in the same window are read together, both compute the same next number, and one overwrites
the other. A tax document with a duplicate number is a compliance problem.

**Fix:** derive the number from something unique — the Airtable record id, or a timestamp plus
a short random suffix. Non-sequential numbers are the right trade for uniqueness on a legal
document.

Related: the 59-second poll is a crude attempt to avoid WF-8 reading a row in the same second
WF-1 wrote it. An Airtable view filtered to `Status = Pending` with a `lastModifiedTime()`
guard would fix it properly.

---

## 8. The director agent's authorisation is not security {#8-the-director-agent-authorisation-is-not-security}

**Severity: Medium** — the gate is a capability check, not an access control.

WF-9 compares the Telegram `from.id` against a stored owner id and refuses anyone else. That
stops accidental and casual misuse, which is its value. It does not stop an attacker: a
Telegram chat id is a client-supplied value, and nothing in the workflow authenticates the
sender.

**Fix if this ever handles real money:** put a secret or an approval step between the bot and
the data — a second, unguessable phrase, or a confirmation code sent to the owner's other
channel.

---

## 9. WF-13's webhook is unauthenticated {#9-the-dashboard-webhook-is-unauthenticated}

**Severity: High** — anyone with the URL can write to the base.

`POST /ERPMainSite` has no auth token, header check, or origin check. `create`, `update` and
`delete` are all reachable by anyone who knows the URL, and it is a short, guessable path on a
public host.

This is the main reason the repository is private. See [SECURITY.md](../SECURITY.md).

**Fix:** require a shared secret in a request header and verify it before the switch, with the
secret stored as an n8n environment variable rather than in the JSON. The dashboard would send
it from `CONFIG`. Rotating the URL helps a little; verifying a secret helps properly.

---

## 10. `products.imageUrl` is collected and thrown away {#10-product-image-url-is-collected-and-thrown-away}

**Severity: Medium** — every product shows a generated initial avatar.

The form accepts an image URL and the payload carries it, but there is no `imageUrl` column in
Airtable and no mapping in `SCHEMAS`, so the value is discarded on write. `CONFIG.productImg`
points at a public image CDN, but nothing ever reaches it.

**Fix:** add the column and the `server:` mapping, and have the table render
`row.imageUrl || CONFIG.productImg` with the initial-letter avatar as the final fallback.

---

## 11. Relations are text keys, not Airtable links {#11-relations-are-text-keys-not-airtable-links}

**Severity: Medium** — no referential integrity; a rename orphans records.

`invoices.CustomerId`, and the task→customer and task→invoice references, are plain text. Airtable
cannot enforce that they point at a real row, cannot cascade, and cannot roll up totals
automatically. WF-9's income figure is therefore a query over loose strings.

**Fix:** switch to Airtable linked-record fields. This is a base migration plus a payload
change, so it belongs to a later iteration rather than a hurried fix.

---

## 12. The `LidCreated` / `LidExist` contract is fragile {#12-the-lidcreated-lidexist-contract-is-fragile}

**Severity: Low** — works today, breaks on the next edit.

WF-3 answers with the raw strings `LidCreated` and `LidExist` — missing vowels. The landing
page compares by substring, so it tolerates the typo. Any consumer that compares the string
exactly will fail, and neither string is self-documenting.

**Fix:** pick three stable values — `created`, `exists`, `error` — and change both sides in the
same commit.

---

## 13. Agents are unbounded {#13-agents-are-unbounded}

**Severity: Low** — a cost and abuse risk, not a bug.

WF-5 is reachable by anyone who finds the bot's handle, and each turn costs a chat completion,
an embedding and a vector search. There is no turn limit, no per-user quota, and no cap on the
agent loop's iterations. A scripted conversation loop is an unbounded bill.

**Fix:** cap the agent node's iterations, and keep a per-chat turn counter in MongoDB.

---

## Fixed during development — kept for the record {#fixed-during-development}

### Searching a table killed every button on the page

**Severity: High, now fixed.** The product search box re-rendered the table with `renderTable()`
instead of `reRender()`, so the re-render replaced the table's DOM and every listener with it.
Typing one character made edit, delete, sort, pagination and paging stop responding — while the
list still *looked* correct, which is what made it hard to diagnose.

The fix captures the input reference before the debounce's async boundary, calls `reRender()`,
and restores focus and the caret afterwards. Verified across all six tables at desktop and
mobile widths with no console errors. The code is in
[04-dashboard.md](04-dashboard.md#rerender--the-bug-factory-and-the-fix), and the convention
that a render and its wiring are one operation is now recorded in [AGENTS.md](../AGENTS.md).

### Expanding a product description moved the stock badge

**Severity: Medium, now fixed.** Descriptions lived in a cell that grew on expand, so the
entire row's height changed and the stock badge jumped. The description now lives in a sibling
`rowdetail` row, and the badge geometry is a verified invariant: `51x29` at an unchanged `y`,
desktop and mobile.

### Customer id and customer name were conflated

**Severity: Medium, now fixed.** `customers` has two id-like fields — the Airtable record id
and the `CUST00000x` business key — and the form was writing the display name into the business
key. The dashboard now lists and edits `CustomerId` as its own field. See
[02-airtable-base.md](02-airtable-base.md#the-single-most-confusing-field-in-the-dashboard).

### The empty-products list was not a bug

**Severity: not a bug.** An empty table with an error flag in `Store.errors` renders the error
card instead of the empty state, which is correct. Clearing the flag shows the empty state with
the right 7 columns and a `colspan="7"` detail row. Recorded here so nobody spends an afternoon
on it again.

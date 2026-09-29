# 04 — The custom dashboard

`app/aa-gear-admin.html`, about 108k characters, one file, no dependencies, no build step.
Open it from `file://` and it works.

## Why build a dashboard at all

The workflow template has no admin interface — you would open Airtable and edit rows there.
That is fine for two users and useless for a shop: Airtable's grid is not a form, it has no
validation, no Hebrew-first layout, and no place for the interactions a real back office needs.

So the dashboard is a purpose-built replacement for the Airtable UI, and **WF-13 is the API
underneath it**.

## The shape of it

```
app/aa-gear-admin.html
  <style>   design tokens, dark UI, RTL, the 420px card breakpoint
  <body>    one <div id="app">; the entire UI is generated
  <script>
    CONFIG      webhook URL, product image CDN
    TABLES      the six tables
    SCHEMAS     per-table columns, form fields, tone colours
    Store       the only stateful object: data, search, sort, page, errors
    api()       the single network helper — one POST to WF-13
    render*()   view rendering, always via reRender() so listeners re-attach
    wire*()     event wiring, always paired with a render
```

`dir="rtl"` is on the root, and every string in the interface is Hebrew.

## The three pieces of logic worth explaining

### `api()` — one function, all six tables

Every read and write is one `POST` to WF-13:

```js
{ table, action, data, filter }   // action: read | create | update | delete | chat
```

Because the table travels in the body, WF-13 needs no per-table endpoints and the front end
needs no per-table code. This is the single design decision that keeps the file small.

### `reRender()` — the bug factory, and the fix

Rendering a table does `host.innerHTML = ...`, which **destroys every descendant and its
listeners**. The original search debounce called `renderTable(table)` and nothing else, so
typing one character in the products search box silently removed every button on the page:
edit, delete, sort headers, pagination. The list still looked right, which is what made it
confusing — the DOM was correct and the behaviour was gone.

The fix routes search through the same path as everything else:

```js
var live = q;                                  // capture before the async gap
state.search[table] = live.value;
state.page[table] = 1;
var keep = document.activeElement === live || document.activeElement === document.body;
reRender(table);                                // render AND re-wire, together
if (keep) { var n = $('#q-' + table); if (n) { n.focus(); n.setSelectionRange(n.value.length, n.value.length); } }
```

Two details are load-bearing. `live = q` is captured *before* the debounce's asynchronous
boundary, because `q` is reassigned on every keystroke and the timeout may run after the user
has typed more. And `reRender` — not `renderTable` — is what keeps the controls alive; the
convention throughout the file is that **a render and its wiring are one operation**.

Verified across all six tables, at desktop width and at a mobile width, with no console
errors: search, sort, expand, edit, delete, new, pager, filter chips, and a second search
after a mutation.

### Product descriptions — why a second row

Descriptions are long, and the original implementation put the text in a cell that grew when
expanded. That moved the stock badge vertically, which made the list jump while reading it.

The fix puts the description in its own sibling row, `display: none` until expanded:

```html
<tr class="rowdetail" data-detail hidden><td colspan="7">…description…</td></tr>
```

`colspan` has to match the real column count or the table breaks, and the count is **7** for
`products` — the six data columns plus the expander toggle. The row is a sibling, not a
`<td>`, so the product row's own height no longer depends on the description at all.

Verified as a hard invariant: with a description expanded, the stock badge is still
`51x29` at the same `y` as the collapsed list, desktop and mobile.

## The interface

- **Sidebar** — the six tables with live result counts, dark, RTL-aware.
- **Toolbar** — search, filter chips, new-record, refresh, export.
- **Table** — sortable headers, clickable rows, per-row edit/delete, pagination.
- **Product list** — a card layout below 420px instead of a squashed table.
- **Dialogs** — create/edit forms, driven by the per-table field schema, so a new table needs
  no new form code.
- **Toasts, skeleton loader, preloader** — the unglamorous parts that decide whether it feels
  like a product.
- **Accessibility** — `aria-label` on icon-only controls, focus restored after re-render,
  Enter and Space both activate rows.

## The data model it reads

`SCHEMAS` in the file is the real contract, and it is what `schema/schema.json` was generated
from. Two things in it are bugs being documented rather than bugs being hidden:

- all seven `orders` fields have no Airtable mapping, so saving an order writes an empty record;
- `products.imageUrl` is collected and dropped, so product images are generated initials.

Both are covered in [09-known-issues.md](09-known-issues.md), with the fix sketches.

## The landing page

`app/aa-gear-landing Final.html` is the public-facing site: RTL, Hebrew marketing copy, and a
lead form that POSTs to WF-3. Same design language as the dashboard, no shared build, no shared
CSS file — the duplication is deliberate, because two static files with no build step are
portable and a shared asset pipeline is not.

The form's submission handler is where the `LidCreated`/`LidExist` contract lives, and where
the `/webhook-test/` mistake lives. See
[09-known-issues.md](09-known-issues.md#2-the-landing-page-posts-to-a-test-webhook).

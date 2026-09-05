# Xalterra deployment

This repository (`xalterra/mcp-erpnext`) is Xalterra's maintained fork of Casys
AI's `mcp-erpnext` (see `docs/UPSTREAM.md` for the exact baseline and update
policy). At this stage it is an **unmodified copy of the upstream application**
plus Xalterra-specific documentation and example configuration — no `xal_*`
tools, no behavioural changes, no forked-off write restrictions. The goal right
now is a clean, understood, validated baseline; custom code is deferred until a
live acceptance test against our own ERPNext instance shows a genuine gap (see
"Planned acceptance test" below).

## Target ERPNext instance

- **Site root:** `https://core.apps.xalterra.com`
- This is the API base. Do **not** configure `ERPNEXT_URL` with a `/desk/...`
  path or any other suffix — the Frappe client (`src/api/frappe-client.ts`)
  appends `/api/resource/...` and `/api/method/...` to whatever `ERPNEXT_URL` is
  set to, so a `/desk` suffix would break every request.
- Configuration uses upstream's existing environment variables as-is —
  `ERPNEXT_URL`, `ERPNEXT_API_KEY`, `ERPNEXT_API_SECRET` — no Xalterra-specific
  variables have been introduced. See `.env.example` and
  `docs/environment-variables.md`.
- No credentials exist in this repository. A dedicated, least-privilege ERPNext
  API user will be created and configured separately, after this baseline is
  reviewed.

## Read-first deployment — how it will actually be restricted

Upstream does not currently ship an application-level read/write capability
profile (there is no built-in "read-only mode" flag). Ripping write tools out of
the upstream code to force one would fight future upstream merges for no real
security benefit, so restriction is layered instead, in order of how much it can
be trusted:

1. **ERPNext permissions on the dedicated API user (primary boundary).** Every
   tool call is a normal authenticated Frappe REST/RPC request
   (`src/api/frappe-client.ts`); ERPNext enforces DocType-level permissions
   server-side regardless of what the MCP server exposes. A service user whose
   role has no create/write/submit/cancel/delete permission on any DocType makes
   the corresponding tools fail at the ERPNext layer even though they remain
   registered. This is the boundary that actually matters and the one we control
   operationally, independent of this codebase.
2. **`ERPNEXT_METHOD_ALLOWLIST` left unset.** `erpnext_method_call`
   (`src/tools/operations.ts`) is deny-by-default: with this variable unset, no
   arbitrary Frappe method can be invoked through it at all
   (`src/tools/method-allowlist.ts`). Leave it unset for the read-first
   deployment.
3. **`--categories=` / `ErpNextToolsClient({ categories })`.** Server-side
   category filtering (`server.ts`, `src/client.ts`) can narrow the exposed tool
   surface to only the categories genuinely needed (e.g. `accounting`, `assets`,
   `setup`, `operations`), cutting unrelated categories (`hr`, `crm`,
   `manufacturing`, `kanban`, ...) entirely. Note this filters by _category_,
   not by read/write — a category still includes its create/ update tools, so
   this reduces surface area but is not by itself a read-only guarantee.
4. **Network/auth controls already in the project.** `docker-compose.yml` binds
   to `127.0.0.1` only by default; HTTP mode supports static bearer tokens
   (`MCP_AUTH_TOKEN(S)`) or OAuth JWT validation (`MCP_OAUTH_JWKS_URL` +
   audience/issuer), documented in `docs/http-deployment.md` /
   `docs/environment-variables.md`. These gate _who_ can reach the server, not
   _what_ an authenticated caller can do.

None of the above required a code change — all four are existing upstream
mechanisms used as designed.

### Tools that write, and how they're currently annotated

Every tool in `src/tools/*.ts` carries `annotations.readOnlyHint: true` when
it's a pure read. The tools below do **not** — they are the ones layer 1 above
needs to actually stop, since nothing in this codebase currently gates them by
profile.

**Destructive** (`annotations.destructiveHint: true`):

| Tool                           | Effect                                      |
| ------------------------------ | ------------------------------------------- |
| `erpnext_doc_delete`           | Deletes any document (generic, any DocType) |
| `erpnext_doc_submit`           | Submits any document (generic, any DocType) |
| `erpnext_doc_cancel`           | Cancels any document (generic, any DocType) |
| `erpnext_file_upload`          | Uploads/attaches a file to any document     |
| `erpnext_sales_order_submit`   | Submits a Sales Order                       |
| `erpnext_sales_order_cancel`   | Cancels a Sales Order                       |
| `erpnext_sales_invoice_submit` | Submits a Sales Invoice                     |

**Create/update/assign** (no annotation — writes, but not flagged destructive
upstream):

`erpnext_doc_create`, `erpnext_doc_update`, `erpnext_doc_assign`,
`erpnext_doc_unassign`, `erpnext_method_call` (arbitrary whitelisted Frappe
method — see layer 2 above), `erpnext_journal_entry_create` (creates accounting
transactions), `erpnext_asset_create`, `erpnext_customer_create`,
`erpnext_customer_update`, `erpnext_sales_order_create`,
`erpnext_sales_order_update`, `erpnext_sales_invoice_create`,
`erpnext_quotation_create`, `erpnext_supplier_create`,
`erpnext_purchase_order_create`, `erpnext_company_create`,
`erpnext_item_create`, `erpnext_item_update`, `erpnext_stock_entry_create`,
`erpnext_lead_create`, `erpnext_delivery_note_create`,
`erpnext_leave_application_create`, `erpnext_expense_claim_create`,
`erpnext_kanban_move_card`, `erpnext_work_order_create`, `erpnext_task_create`,
`erpnext_task_update`, `erpnext_project_create`.

`erpnext_journal_entry_create` is called out specifically: it is the one
generic-write tool that posts directly to the general ledger, so the
ERPNext-side permission boundary (layer 1) matters most for it.

## Planned acceptance test

Deferred until a dedicated ERPNext API user and credentials exist (not created
by this baseline work — see `AGENTS.md`/repository instructions). The test will
use **unmodified upstream tools** to check whether Casys's existing
accounting/asset/document tools can already retrieve, for Xalterra's own ERPNext
instance:

- 2025 Trial Balance, Profit and Loss, Balance Sheet at 31 December 2025,
  General Ledger, Chart of Accounts
- the Director's Loan account, its opening balance, and its 2025 movements
- the Range Rover Velar acquisition (~19 September 2025): source transaction,
  fixed asset record, cost, accumulated depreciation, net book value,
  depreciation schedule/entries
- Purchase Invoices, Journal Entries, Payment Entries, and source-document
  drill-down for the above

No figures will be fabricated or inferred to make a statement balance — if
ERPNext's own data is missing or inconsistent for any of the above, that will be
reported as a finding, not patched over. Only a genuine gap surfaced by that
live test — something the unmodified upstream tools cannot already retrieve
cleanly — will justify adding Xalterra-specific (`xal_*`) code, and even then as
a thin wrapper over existing ERPNext reporting/query mechanisms (Trial Balance,
P&L, Balance Sheet, General Ledger must remain ERPNext's own computed output,
never re-derived here).

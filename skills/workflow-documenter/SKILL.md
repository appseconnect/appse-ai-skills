---
name: workflow-documenter
description: >
  Produces clear documentation for an appse ai workflow — a business
  overview the partner can share with their customer, plus a technical
  reference for the partner's own team — by reading the actual saved
  workflow via arise-mcp. Use when a partner asks to "document this
  workflow", "write documentation for workflow {id/name}", "create a
  handover document", "explain what this workflow does", or right after
  workflow-creator has built a workflow. Takes the workflow from the
  conversation (workflow-creator's report) or asks for its ID or name.
  Read-only: never changes the workflow. Asks as little as possible and
  generates the documentation straight away as a Word (.docx) file.
license: Internal — appse ai Partner Accelerator
---

# workflow-documenter — Turn a Built Workflow into Clear Documentation

Reads a saved appse ai workflow and explains it in two layers:

- **Part A — Business Overview:** what the workflow does, in plain business
  language, for the partner's **customer** (operations, finance, IT
  managers). No expressions, IDs, or JSON.
- **Part B — Technical Reference:** exactly how it's built — trigger,
  steps, conditions, field mappings, settings — for the **partner's team**
  to support, test, and change it.

**The failure mode this skill exists to prevent:** documentation that
describes what the workflow was *meant* to do instead of what's actually
saved — or that exposes internal details (credential IDs, org IDs, tool
names) in a document that goes to a customer. Every statement comes from the
saved workflow or the app's real operation details; nothing is invented.

---

## Tone and safety (applies to every document)

- **Describe what's saved, not what was intended.** If the saved workflow
  differs from what the partner expects, document what's saved and flag the
  difference (Step 3) — never paper over it.
- **Never include:** credential IDs, org IDs, connection secrets, internal
  tool or server names, or raw JSON in Part A. Name connections by app
  ("Shopify connection"), not by ID. The workflow ID and link may appear in
  Part B only.
- **No record data.** Never read or quote actual customer/order records
  from workflow runs — describe fields, not their values.
- **Plain, confident, professional language.** Explain *why* each rule
  exists in business terms ("so an order is never created twice"), not
  just *what* it does.
- **Don't invent.** If something can't be determined from the workflow or
  the app's operation details (e.g. who the support contact is), leave a
  clearly marked placeholder — never guess, and don't stop to ask.
- **Neutral branding.** This is the partner's document; use their branding
  only if they provide it. Don't add appse ai branding.

---

## Inputs

| Field | Required | Source | Default |
|---|---|---|---|
| `workflow_id` | ✅ | From workflow-creator's output in the conversation (its "Open it here" link or "Workflow ID" line, earlier in the chat or pasted in); otherwise the partner picks from the latest workflows offered as options, or enters a link, ID, or name (resolved via `list_workflows`) | — |
| `creator_report` | ⬜ | workflow-creator's output, if present in the conversation | Used as extra context for purpose, mapping reasons, safeguards, and assumptions — the saved workflow governs |
| `customer_name` | ⬜ | From the conversation or the partner | Placeholder `[Customer name]` |
| `support_contact` | ⬜ | Partner | Placeholder `[Support contact]` |
| `format` | ⬜ | Only if the partner asks for something else | Word (.docx) — never asked |

---

## Workflow

Follow these steps in order.

### Step 0 — Identify the Workflow

- **First, look in the conversation for workflow-creator's output** —
  either earlier in this chat or pasted in by the partner. Recognise it by
  its report: an **"Open it here:"** link (`…/workflows/{id}/editor`), a
  **"Workflow ID:"** line, and sections like "Mappings I worked out",
  "Safety checks I added", and "Assumptions I'm making". Take the workflow
  ID from the link or ID line and **go straight ahead — don't ask for
  confirmation.** Only if several workflows appear, offer them as options
  (below).
- **Use that report as extra context, not as the source of truth.** Its
  business purpose, proposed mappings (with reasons), safety checks, and
  assumptions help explain *why* the workflow is built the way it is — use
  them in Part A (purpose, safeguards, prerequisites) and in Part B's
  mapping reasons and notes. **The saved workflow always governs:** if the
  report and the saved workflow differ (e.g. a different start date or
  field mapping), document what's saved and flag the difference in "Notes
  for the implementation team".
- **If there's no workflow in the conversation, offer options — don't ask
  an open question.** Call `list_workflows` (most recently changed first)
  and present the latest five as numbered choices, plus a last option to
  enter one directly — using the question tool with selectable options if
  it's available, otherwise a numbered list:
  > "Which workflow should I document?
  > 1. Workflow 24 — updated today
  > 2. Workflow 22 — updated yesterday
  > 3. Workflow 18 — updated 2 days ago
  > 4. Workflow 17 — updated 2 days ago
  > 5. Workflow 16 — updated 3 days ago
  > 6. Another one — paste its link, ID, or name"

  Show workflow **names and when they were last updated**, not IDs. Never
  use a real workflow ID as an example in the question.
- **Given a name:** call `list_workflows` with that name as the search
  term. One match → use it. Several → list them (name, last updated) and
  ask which. None → say so and ask for the ID.
- **Org:** call `list_organizations`; if there's exactly one, use it. Ask
  only if there are several — by org **name**, never ID.

### Step 1 — Read the Workflow and Its Apps (in parallel)

Call, in one parallel batch:
- `get_workflow` for the workflow — the saved nodes, edges, settings,
  status (active/inactive), and name.
- `list_apps` — the display names of the apps involved.

Then, in a second parallel batch, `get_operation_detail` for every app
operation used (trigger, lookups, creates/updates) — for plain-language
operation names and field descriptions. Optionally, `query-docs` on
`/appseconnect/appse-ai-docs` for an operation whose purpose or fields
aren't clear from the operation detail.

Optionally, `list_executions` (metrics only) to state whether the workflow
has run and how the last run ended — never read run payloads.

### Step 2 — Interpret Every Node, in Order

Walk the flow from the trigger along each edge. Branches leaving the same
node run in the order their nodes were added (earlier position in the saved
`nodes` list, lower `idx`) — document them in that order and say so.

Translate each node into business language:

| Node | How to describe it |
|---|---|
| `AppTriggerNode` | "Every {n} minutes, checks {app} for new/updated {records} created since {start date}, up to {limit} at a time." Translate the cron (e.g. `*/3 * * * *` → every 3 minutes). |
| `FilterNode` | The rule in plain words and why — e.g. `is_not_empty` on email → "Orders without a customer email are skipped, so they can't match the wrong customer." Records that don't pass simply stop. |
| `DecisionNode` | "If {condition}, then {true path}; otherwise {false path}." Say when a branch intentionally ends ("if the order already exists, nothing more happens"). |
| `SplitterNode` | "Each {list item — e.g. order line} is handled separately," naming the list (`fields_to_split`). |
| Lookup `AppNode` (get/search) | "Looks up the {record} in {app} by {key}." |
| Create/update `AppNode` | "Creates/updates the {record} in {app}," then its field mappings (Part B). |
| `JsonConverterNode` | "Converts {field} from text into structured data." |

**Translate expressions into plain source descriptions** (Part B mapping
tables; Part A only in summary):

| Expression form | Plain description |
|---|---|
| `{{$payload.x}}` / `{{$('Node').payload.x}}` | "{Field x} from {the record at that step — e.g. the Shopify order}" |
| `substringAfter(id,'gid://shopify/Customer/')` | "The numeric part of the Shopify customer ID" |
| `substringBefore(createdAt,'T')` | "The date part of the order's creation time" |
| `to_number(x)` / `map(&to_number(x), list)` | "{x}, sent as a number" (for each line) |
| `date_add_days(x, `n`)` | "{x} plus {n} days" |
| `{{a}} {{b}}` | "{a} and {b} joined with a space" |
| A fixed value (e.g. `"C"`, `"Person"`, `"$"`) | "Always set to {value}" — and what it means (e.g. "customer type: customer"; "SAP currency code for US dollars") |
| `value[0].X` on a lookup result | "{X} of the first record found" |

If an expression uses a function you can't explain from the docs, show it
as-is in Part B with "(expression)" rather than guessing its meaning.

### Step 3 — Check What You're Documenting (flag, never fix)

While reading, note anything a reader must know — **as observations in
Part B, "Notes for the implementation team"**, never silently corrected and
never omitted:

- The workflow is **inactive** (not yet switched on).
- The trigger's start date is **in the past** (it will process older
  records on first run) or **in the future** (nothing runs until then).
- A field is **blank** or a node has **no connection** attached.
- A `$('…')` reference points to a node name that **doesn't exist** in the
  workflow.
- A write depends on records that must already exist in the target (e.g.
  items for an order) — state it as a prerequisite in Part A.
- No execution history yet → say "not yet run"; never call the workflow
  "tested" unless execution history shows successful runs.

This skill **never changes the workflow.** If something needs fixing, say
what and suggest updating the workflow (e.g. with workflow-creator).

### Step 4 — Assemble the Document

**Title:** *{Workflow purpose in business terms} — Workflow Documentation*
(e.g. "Shopify Orders to SAP Business One Sales Orders").
**Header:** customer name, prepared by (partner), date, version 1.0.

**Part A — Business Overview (for the customer)**
1. **Purpose** — the business problem it solves and the outcome, in 2–3
   sentences.
2. **Systems involved** — each app and its role (e.g. "Shopify — where
   orders are placed"; "SAP Business One — where sales orders are
   created").
3. **When it runs** — schedule and what it picks up.
4. **What happens, step by step** — a short numbered list in plain
   language, following the flow (including branches).
5. **Business rules and safeguards** — what's skipped, what's prevented
   (duplicates, blank data), and why.
6. **What's not handled** — records the workflow skips or doesn't cover
   (e.g. orders without an email; returns), so nobody expects them.
7. **Prerequisites for the customer** — what must be true for it to work
   (e.g. products exist in the ERP with the same SKU; the connection stays
   active).
8. **How to check it's working** — what the customer should see in the
   target system after a new record.
9. **Support** — `[Support contact]` unless the partner provides one.

**Part B — Technical Reference (for the partner's team)**
1. **Workflow details** — name, ID, link
   (`https://workflow.insync.top/workflows/{workflowId}/editor`), status
   (active/inactive), last run status if available.
2. **Trigger** — app, event, schedule (cron and plain), start date, batch
   size.
3. **Flow outline** — the steps as a simple text diagram (e.g.
   `New Shopify Order → Has Email → Find Order → Order exists? → …`),
   including branch order.
4. **Step-by-step reference** — a table: step #, step name (as in the
   workflow), type, app/operation, purpose.
5. **Conditions** — each Filter/Decision: the exact rule (field, operator,
   value) and its plain meaning.
6. **Field mappings** — one table per create/update step: target field |
   source / rule (plain description) | expression (as saved).
7. **Settings and error handling** — per step: on error (stop/continue),
   retry, always output data — and what that means in practice (e.g. "if
   item creation fails, the run stops before the order is created").
8. **Connections used** — by app name only.
9. **Testing checklist** — 3–5 concrete test cases derived from the flow
   (happy path, each skip/branch, duplicate re-run), each with the expected
   result.
10. **Troubleshooting** — likely failures for *this* workflow (e.g.
    "record rejected: code not defined in SAP", "connection expired",
    "duplicate record") and where to look (the workflow's run history).
11. **Notes for the implementation team** — Step 3 observations.
12. **Change log** — version 1.0, date, "Initial documentation".

Include only what applies to this workflow — no generic filler sections.

### Step 5 — Generate the Word Document Straight Away

**Don't ask questions and don't wait for a review in chat** — generate the
Word file as soon as the workflow is identified and read. The only question
this skill ever asks is *which workflow* (Step 0), and only when it can't
tell from the conversation.

- **Anything unknown becomes a placeholder,** never a question: customer
  name → `[Customer name]`, support contact → `[Support contact]`. Step 3
  observations go in "Notes for the implementation team" — don't stop to
  ask about them.
- Use the `docx` skill. Clean, professional layout: title, header block,
  Part A, then Part B, with tables for steps and field mappings. Part A must
  read well on its own, so a partner can share just that part with the
  customer.
- **File name:** `Workflow Documentation - {Workflow purpose} - {YYYY-MM-DD}.docx`,
  saved in the working directory unless the partner names another folder.
  Never overwrite an existing file — add `v2`, `v3`.
- Produce a PDF only if the partner explicitly asks for one (use the `pdf`
  skill, from the same content).
- Report back briefly: the file path, a two- or three-line summary of what
  the workflow does, any Step 3 notes worth a look, and a reminder to fill any
  placeholders before sharing.

---

## Allowed Tools

### arise-mcp — read-only
```
list_organizations   → resolve the org (Step 0)
list_workflows       → resolve a workflow name to its ID (Step 0)
get_workflow         → read the workflow the partner asked to document — only that one
list_apps            → app display names (Step 1)
get_operation_detail → plain names and field descriptions of the operations used (Step 1)
list_executions      → optional, metrics only: whether it has run and the last outcome
```
Never call `create_workflow`, `save_workflow`, or anything that writes.
Never call `get_node_data` or read run payloads — documentation never
contains record data.

### Documentation Access — Context7 (read-only)
`query-docs` with library ID `/appseconnect/appse-ai-docs`, only to explain
an operation, field, or expression function that isn't clear from the
operation detail.

### Local file output — Step 5
`docx` skill → Word version · `pdf` skill → PDF version. Writes go to the
working directory (or a folder the partner names); never overwrite.

---

## Known Limits (update as testing reveals more)

- The workflow link base URL is the test environment
  (`workflow.insync.top`) — same open item as workflow-creator; confirm the
  production URL before partners use this.
- Node types without a confirmed description (e.g. agent/AI nodes, XML or
  Base64 converters) are documented from the docs if available, otherwise
  shown as "(step type not described)" in Part B.
- Not yet tested live.

---

## Output Rules

- Document what's **saved**, not what was intended; flag differences in
  "Notes for the implementation team" — never fix the workflow.
- Read workflow-creator's output from the conversation first (link, ID,
  and its mapping reasons, safeguards, and assumptions as context). If
  there's none, offer the latest workflows as numbered options by name —
  never an open "which workflow?" question.
- Part A is plain business language for the customer — no expressions,
  IDs, or JSON. Part B holds the technical detail.
- Never include credential IDs, org IDs, secrets, internal tool names, or
  record data from runs.
- Every statement traces to the saved workflow, the app's operation details,
  or the docs; anything else is asked or left as a marked placeholder.
- Never call a workflow "tested" unless its run history shows successful
  runs.
- Ask as little as possible: the only question is which workflow, and only
  when the conversation doesn't show it. No confirmation, review, or format
  questions — unknown details become placeholders.
- The final output is always a Word (.docx) file (PDF only on request); the
  run isn't complete until the file path is reported.
- On being asked to stop, stop immediately and report what has and hasn't
  been produced.

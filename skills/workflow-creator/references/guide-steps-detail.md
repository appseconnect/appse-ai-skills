# workflow-creator — Workflow Steps, Tone, Questions & Output Rules (full)

*Moved verbatim from SKILL.md on 2026-09-24 so the core file stays short. Read when a step's condensed rule in SKILL.md isn't enough to decide. SKILL.md holds the condensed rules; this file holds the full detail and examples.*

## Tone

Everything this skill says to the partner should be confident and
encouraging, not hedging or self-doubting, and should never expose internal
implementation details. Two separate rules apply together:

**1. State decision-relevant facts plainly, without hedging.** There's a
difference between two things that can look similar but aren't:
- **Decision-relevant facts** the partner needs in order to choose something
  (e.g. "this will use 2 workflows from your allocation," "this mapping
  wasn't cross-checked against documentation") — always state these,
  plainly and matter-of-factly. These are the partner's information to have.
- **Meta-commentary about what the skill itself can't do** ("I can't check
  remaining balance," "I'm not able to confirm...") — cut this framing.
  State the underlying fact directly instead.

**2. Never name or reference internal tools or systems in partner-facing
text.** This includes `arise-mcp`, `Context7`, "the skill," "the reference
workflow," specific MCP tool names (`save_workflow`, `get_operation_detail`,
etc.), or any other internal implementation detail. A partner doesn't need
to know this machinery exists — describe capabilities and limitations in
plain business language instead. For example:
- Wrong: *"arise-mcp doesn't expose a rename tool, so I couldn't set the
  name directly."*
- Right: *"This workflow is currently named 'Workflow 9' — rename it to
  something clearer next time you're in the appse ai UI."*

This applies everywhere partner-facing text is generated — Step 9's summary,
Step 11's report, and any clarifying question — not just one example. (The
Known Limits, Allowed Tools, and Reference Patterns sections of this file are
internal documentation for whoever maintains this skill, not partner-facing,
and are unaffected by this rule.)

The goal is a partner coming away feeling like they have a capable teammate,
not like they're being warned away from using it, and never feeling like
they're reading a systems log.

---

## Asking Questions

When more than one question needs an answer before proceeding, the goal is
speed without confusion — not "always ask one at a time," which would slow
every run down, but not "batch everything" either, which causes real
confusion when a later question's right answer actually depends on an
earlier step's result.

- **Order matters more than quantity.** Never ask a question whose correct
  answer depends on information a later step will surface — wait for that
  step first, then ask. (Concretely: don't ask which action variant to use
  before Step 6's field check has run, if the field check could reveal a
  reason to prefer one variant over another — as it does for Shopify's
  product actions, where a media-requiring variant only makes sense once
  you know whether the source data actually has media fields.)
- **Only batch genuinely independent questions** — ones where the answer to
  one has no bearing on how to answer another.
- **When batching, number the questions explicitly and ask for numbered
  answers** (e.g. "Q1: ... Q2: ... Q3: ... — reply like '1: yes, 2: b'").
  Never accept a bare, positional reply like "1" against multiple questions
  and assume which one it answers.
- **If a reply doesn't clearly map to all outstanding questions**, say
  explicitly which question(s) were answered and which are still open —
  don't just re-list everything as if nothing was received; that reads as
  ignoring the partner's answer, not as asking a new question.
- **Don't ask again what's genuinely unambiguous.** If a request already
  specifies something clearly and only one real option exists once
  connections are checked (e.g. only one of two app-catalog variants has a
  saved credential), state the resolution and move on rather than asking —
  reserve questions for genuine, live decisions.

---

## Inputs

| Field | Required | Source | Description | Default |
|---|---|---|---|---|
| `requested_scope` | ✅ | User states | What the user actually asked for — one event, or a broader goal (e.g. "the full sales cycle") that may span several | — |
| `org_id` | ✅ | User confirms | Which appse ai organization to build in | none — always ask, every run |
| `entity_type` | ✅ | User states, per workflow | What's being synced (e.g. customer, product, order) | — |
| `source_app` | ✅ | `list_apps`, user confirms if ambiguous | App the trigger fires from, per workflow | — |
| `target_app` | ✅ | `list_apps`, user confirms if ambiguous | App the action writes to, per workflow | — |
| `trigger_operation` | ✅ | `list_operations`, user confirms if ambiguous | The "new/updated {entity_type}" trigger, per workflow | — |
| `action_operation` | ✅ | `list_operations`, user confirms if ambiguous | The "create/update {entity_type}" action, per workflow | — |
| `structural_pattern` | ✅ | Assessed by the skill in Step 0, confirmed with user | The shape this workflow needs — a known Reference Pattern (simple / dedupe-skip / dedupe-create-or-update / find-or-create-parent-then-child / SKU-reconciliation / parallel-branch) or **custom**, composed from Building Blocks for the scenario's rules | — |
| `field_mappings` | ⬜ | `query-docs` (first pass) + `get_operation_detail` (governs); else ask user | Source field → target field mapping, per workflow | No default — do not assume standard fields across arbitrary apps/entities |
| `since_from` | ⬜ | User input, if trigger is polling-based | Trigger start date/time | the current date/time — an earlier date only if the partner deliberately wants a backfill (see Think Like an Integration Expert) |
| `limit` | ⬜ | User input, if trigger is polling-based | Records per request | 10 |

---

## Workflow

Follow these steps in order. Do not skip or reorder.

### Step 0 — Assess, Plan, and Confirm Scope
Before anything else, do two things together:

**a) Decompose by business event**, as before: if `requested_scope` spans
more than one distinct triggering event (e.g. "the full sales cycle" implies
order created, payment received, shipment created, invoice generated — each
a separate event), break it into the explicit list of separate workflows it
requires. Do not build a single workflow that internally handles multiple
unrelated trigger events — this holds even if the user explicitly asks for
them to be combined; explain the reasons (allocation model, envelope shape
only validated per-pattern, operational clarity) and propose the correct
decomposition instead.

**b) Assess structural pattern and performance shape, per workflow**: for
each workflow in the decomposition, map the scenario's rules to Building
Blocks and name the resulting shape — a known Reference Pattern if one
matches, otherwise **custom** (described in plain steps) — and whether the downstream processing for a single trigger
event would be heavy enough (multiple lookups, a reconciliation cascade,
several dependent branches) that splitting it further — even within one
business event — would keep each workflow simpler and faster to run. This is
a qualitative judgment call, not an enforced threshold (no real execution-
time data exists yet to set one — see Known Limits); when genuinely
uncertain, present it as a trade-off in the plan rather than deciding
silently.

**Then present a plan, not just a count**, and let the user decide:
> "This covers **{n} workflow(s)**: {for each — trigger → action, and which
> structural pattern it needs, e.g. 'dedupe-and-update' or 'simple sync'}.
> {If a genuine alternative decomposition exists — e.g. finer-grained for
> performance vs. fewer, larger workflows — present both with trade-offs.}
> This will use {n} workflows from your allocation. Which would you like to
> go with?"

- Wait for the user to confirm the plan (or a modified version of it) before
  proceeding to Step 1 for each workflow in the confirmed set.
- Never create multiple workflows one at a time without having shown this
  plan up front.
- This skill cannot currently check remaining allocation balance — no
  internal tool exposes it. State the count being requested plainly (per
  Tone above — no hedging, and no naming of internal tools). Do not claim to
  know whether the partner has that many remaining.

Repeat Steps 1–11 below for each workflow in the confirmed set.

### Step 1 — Resolve Organization
Call `list_organizations`. If more than one is active, stop and ask which
`org_id` to use. **Never reuse an org_id from a previous run or conversation.**

### Step 2 — Confirm Apps Are Connected
Call `list_apps`, then `list_credentials`. Confirm every app the workflow
will use — `source_app`, `target_app`, and any app needed only for an extra
lookup/search node — has an actual **saved credential** in this org. A
catalog entry with no credential at all is not enough.

**If any app has no saved credential, pause and ask the partner to add it —
do not end the run:**

> "To build this, **{app}** needs to be connected in **{org}**, but I don't
> see a saved connection for it yet. Please add one in the appse ai portal
> (Credentials → Add credential → {app}), then tell me when it's done and
> I'll pick up from here."

Name every missing app in one message, not one at a time. When the partner
says it's done, call `list_credentials` again and confirm the credential now
exists before continuing to Step 3. If it still isn't there, say so plainly
and ask again — never proceed without it, and never build the workflow with
a node that has no credential attached.

Record the credential `id` found for each app here — Step 10 attaches it to
every node for that app.

If an app name matches more than one catalog entry (e.g. a cloud vs. on-prem
variant), and only one of them has a saved credential, state which one will
be used and proceed — this is a resolved fact, not an open question, and
does not need to stop and ask (see Asking Questions). Only stop and ask if
more than one matching variant genuinely has a usable credential.

**Temporary override (added 2026-09-24, revisit once resolved):** do not
block on the `isValidated` flag. Its real meaning is currently unconfirmed —
it has shown `false` even for a credential known to work in practice (a
Shopify credential that has already moved real data in an executed
workflow), across multiple orgs and testers. Treat "credential exists" as
sufficient to proceed for now, and do not mention this override or the
validation flag to the partner — it's an internal, temporary relaxation, not
partner-facing information. Once the platform team confirms what
`isValidated` actually reflects, update this step accordingly — this
override should not become permanent by default; see Known Limits.

### Step 3 — Check for an Existing Matching Workflow (idempotency)
Call `list_workflows`. If a workflow already exists in this org with the same
`source_app`, `target_app`, and `entity_type`, stop and tell the user — ask
whether to reuse/edit it or create a new one anyway. Do not create a duplicate
silently.

### Step 4 — Find Trigger and Action Operations
Call `list_operations` for `source_app` to find the "new/updated {entity_type}"
trigger, and for `target_app` to find the "create/update {entity_type}" action.
If more than one operation looks like a plausible match for either, stop and
ask which is correct — do not guess from naming alone. If the entity type's
plain-English name doesn't return matches, try the app's own likely domain
terminology before giving up (e.g. SAP Business One calls products "Items").
This same "don't guess the right identity, ask" principle applies to
app-level ambiguity too, not just operation-level — e.g. if an app name
matches more than one catalog entry (such as SAP Business One having both a
cloud and an on-prem/DIS variant), stop and confirm which one is meant before
proceeding — unless Step 2 already resolved it because only one had a usable
credential, in which case do not ask again.

If `structural_pattern` requires additional operations beyond the base
trigger/action (e.g. a search/lookup action for dedupe patterns, or an
item-lookup action for SKU reconciliation), find and confirm those here too,
using the same ambiguity rules.

### Step 5 — Check Operation Stage
Both `release` and `preview` stage operations are acceptable to use, with no
extra confirmation needed. **Never use a `dev`-stage operation** — if the
only matching trigger or action is dev-stage, stop and report this rather
than using it. *(Note: no live call in testing so far has ever returned
`dev` as an observed stage value — only `release` and `preview` have been
seen. This rule is currently unconfirmed against real data; verify with the
platform team what the actual set of stage values is, and update this step
if `dev` isn't a real value or is named differently.)*


*(Steps 6–7 in full: see `guide-field-mapping.md`.)*

### Step 8 — Learn the Envelope Structure
Using the `structural_pattern` identified in Step 0, select the matching
Reference Pattern:
- **Simple sync** → call `get_workflow` on the live reference workflow (see
  Reference Patterns above).
- **Any branching pattern** (dedupe-skip, dedupe-create-or-update,
  find-or-create-parent-then-child, SKU-reconciliation, parallel-branch) → read the matching local file directly
  from `references/` — no MCP call needed.

Apply the structure only — never literal field values, credential IDs, or
app-specific mappings from the reference.

- **Custom shape (no reference matches), or a mix of patterns** → compose it
  from Building Blocks: pick the node types for each rule, copy node and
  edge shapes from `references/conventions.md`, and wire the edges
  (`true`/`false` handles on Decision nodes, `default` everywhere else).
  Still read the closest reference file for the envelope details. Say in
  Step 9 that it's a custom flow — don't stop to ask just because no
  reference matches.
- Only stop and ask if a needed node type isn't in Building Blocks.

**Then review the design as an integration expert** (see Think Like an
Integration Expert), whichever route produced it — a reference file only
shows a shape that worked once, not every safeguard this scenario needs.
Add whatever that review calls for before presenting it in Step 9.

### Step 9 — Present Summary and Wait for Confirmation
Before creating this workflow, present (following the Asking Questions and
Tone guidance above):

> I'm about to build a workflow in **{org}**: when **{trigger, in source_app}**
> happens, **{action, in target_app}**, for entity type **{entity_type}**,
> using the **{structural_pattern}** shape, with these field mappings:
> {list of direct mappings, marking any not cross-checked against
> documentation}.
>
> **Mappings I worked out for you — please check:**
> - **{target field}** ← `{expression}` — {one-line reason, e.g. "Customer
>   number is required, so I'm using the Shopify customer's numeric ID"}
>
> {If this is a custom flow built from their rules rather than a known
> pattern, say so and walk through it in plain steps, e.g. "1. New Shopify
> order → 2. look up the customer by email → 3. if found, create the order
> for them; if not, create the customer first, then the order."}
>
> **Safety checks I added:**
> - {plain-language list of what your expert review added, e.g.
>   "Customers without an email are skipped, so they can't match the wrong
>   record", "A customer only counts as found if the email really matches",
>   "It starts from new customers created from now on — existing ones aren't
>   touched"}
>
> If you'd like any of these mapped differently,
> tell me what to use and I'll update it. Otherwise, shall I go ahead?

Wait for explicit confirmation. Do not proceed on an ambiguous or implied yes.

### Step 10 — Build and Save
Call `create_workflow`, then `save_workflow` using the envelope structure
learned in Step 8, the field mappings from Steps 6–7, and this workflow's own
trigger and action. The Step 9 "yes" covers both calls — once the workflow is
created, save it straight away without asking again, then tell the partner
it's saved (Step 11).

**Attach a credential to every app node, including the trigger.** Set
`data.credential_id` on each `AppTriggerNode` and `AppNode` to the
credential `id` recorded in Step 2 for that node's app. (Decision, Filter,
and other non-app nodes don't take one.) The trigger is the easiest to miss
— a trigger with no credential can never run.

**Then verify:** call `get_workflow` on the workflow just saved and check
that every `AppTriggerNode`/`AppNode` has a `credential_id` matching Step 2,
that **no property anywhere in any node is `""` or missing a value —
check recursively, including every field inside arrays and objects** (a
blank `Currency` inside `ItemPrices` counts; fill it via the Step 6 ladder
or ask, never save it blank), **that every `$('<name>')` in any
expression matches the `current_name` of a node that exists in this
workflow and sits earlier on the same path** (list the node names, then
check each reference against the list), **that every
mapping reads from the node that actually passes the record to it (not the
trigger, when a Filter or Decision sits in between), and that the
safeguards from your expert review (listed in Step 9) actually made it into
the saved flow.** If anything is missing, fix it with `save_workflow` and
re-check; report anything still wrong — do not tell the partner the build
is complete.

### Step 11 — Report Back Plainly
**Lead with a clickable link to open the workflow** in the appse ai
editor, built from the workflow ID returned by `create_workflow`:

> **Open it here:** https://workflow.insync.top/workflows/{workflowId}/editor

Use the real ID, as a full clickable URL (not just the ID). If several
workflows were built from Step 0, give one link per workflow, each labelled
with what it does. Give the same link again whenever you later update that
workflow (e.g. after a mapping change).

Then state: workflow name and ID, exact trigger and action used, every field
mapping applied, and which mappings were documentation-confirmed,
unconfirmed, or **proposed by you** (Step 6 rungs 2–4). Repeat the proposed
ones as a short list with their reasons, and close with: _"If any of these
should come from somewhere else, tell me what to use and I'll update the
workflow."_ If the partner then asks for a change, update this same workflow
(read it with `get_workflow`, change only the named fields, `save_workflow`)
rather than creating a new one. The partner's change request is the go-ahead
— save without asking again, then confirm what changed. This is what the user checks against the appse ai
UI once it's reachable — the build itself is safely persisted regardless of
UI availability, so a UI outage delays verification, not the build's
validity (state this plainly, without naming the database or any internal
system). If this was one of several workflows from Step 0, also report
progress against the full set (e.g. "2 of 4 built so far").

If the platform assigns a generic default name (e.g. "Workflow 9") rather
than the descriptive name intended, state this plainly and positively —
e.g. "This workflow is currently named 'Workflow 9' — rename it to something
clearer next time you're in the appse ai UI" — never explain this in terms
of what an internal tool does or doesn't support (see Tone).

---

## Allowed Tools

*(Internal reference for whoever maintains this skill — not partner-facing;
see Tone.)*

### arise-mcp — platform read/write
```
list_organizations
list_apps
list_credentials
list_operations
get_operation_detail
list_workflows
get_workflow → restricted to exactly two targets: (1) the live simple-pattern reference workflow, structure only (Step 8); (2) the workflow this run just created — to verify credentials and mandatory fields (Step 10) and to apply mapping changes the partner asks for (Step 11). Never any other workflow.
create_workflow
save_workflow
```
Note: no tool here exposes the partner's remaining workflow allocation/quota,
a rename operation, or the node/edge envelope schema directly — see Known
Limits.

### Documentation Access — Context7 (read-only, scoped)
```
resolve-library-id   → restricted to resolving "appseconnect/appse-ai-docs" only
query-docs           → restricted to content under:
                          - docs/app_integrations/{app}.md
                          - docs/platform/key_concepts/expressions_mapping/
```
Do not use Context7 to resolve or fetch any other library. This is a
separate access grant from arise-mcp — do not treat the two as interchangeable
or use one to justify expanding the other.

### Local Reference Files (read-only, this skill's own folder only)
```
references/pattern-dedupe-skip-return-request.json
references/pattern-dedupe-create-or-update-customer.json
references/pattern-dedupe-create-or-update-businesspartner-subrecords.json
references/pattern-dedupe-create-or-update-product.json
references/pattern-find-or-create-customer-then-order.json
references/pattern-sku-reconciliation-and-multibranch-order.json
references/pattern-parallel-branch-inventory-notification.json
references/conventions.md
```
No MCP or network access needed for these — plain local file reads, scoped
to this skill's own `references/` folder only. Do not read or infer
structure from any other file outside this folder or outside the arise-mcp/
Context7 tools listed above.

---

## Output Rules

- Always decompose a multi-event request into separate workflows and present
  a plan (not just a count) before building anything — including which
  structural pattern each workflow needs, and any genuine alternative
  decomposition worth considering — and wait for the user to confirm.
  Never build multiple workflows silently one at a time, and hold the
  one-event-per-workflow line even if the user explicitly asks to combine
  events; explain why and propose the correct split instead of complying.
- Always re-resolve `org_id` every run — never assume the last-used org still
  applies.
- Always give a clickable link to open each workflow built or updated
  (`https://workflow.insync.top/workflows/{workflowId}/editor`) at the top of
  the Step 11 report.
- Always review the design as an integration expert before Step 9 (see
  Think Like an Integration Expert) — empty keys, weak matches, how many
  records a write can touch, what the first run picks up, overwrites,
  loops. Add the safeguards yourself and list them under "Safety checks I
  added". References are a starting point, never the whole design.
- Never copy a reference's logic or mappings without judging them against
  this scenario — check references, then decide with your own integration
  expertise; when they disagree, choose the safer design and explain why.
- Always check every app the workflow uses has a saved credential before
  building. If one is missing, ask the partner to add it in the portal, wait,
  re-check with `list_credentials`, then continue — never build without it.
- Always attach the credential to every app node, including the trigger, and
  verify with `get_workflow` after saving.
- Use documentation as a first-pass, not final, source for field
  requirements — the live operation-detail call always governs when they
  disagree.
- If documentation lookup is unavailable, fall back to the live call and
  mark affected mappings as not cross-checked — never stall or fabricate
  what documentation says.
- A shallow/top-level-only result from the live operation-detail call on an
  object- or array-typed parameter does not count as resolved — treat its
  internal shape as still unknown until confirmed by documentation, the
  user, or a confirmed guess.
- Never guess field-reference expression syntax, or Decision/Filter
  condition shape — use the documented forms from Documentation Reference
  and Reference Patterns, never invented syntax or structure.
- Never leave a mandatory field empty or send `""` — **including fields
  nested inside arrays and objects**, which the live operation detail often
  doesn't list. Fill it using the Step 6 ladder (direct field → expression
  function → reference-pattern approach → sensible constant), list every
  proposed mapping with its reason in Steps 9 and 11, and only ask when no
  rung gives a plausible value. Never save a workflow with a blank value
  anywhere in it.
- Never copy or guess company-specific settings (currency, price list,
  warehouse, tax code, posting group, number series) — use a source field,
  else a run-time lookup action if one exists, else ask the partner in one
  batched question. A reference workflow's value for these is never valid
  for another customer.
- Never silently invent a data-shape, app/operation identity, or structural
  pattern that no source resolves — stop and ask, or get explicit
  confirmation. (Proposed field mappings are different: make them, and
  disclose them.)
- Never expand tool access mid-run — if the allowed tools aren't enough, stop
  and say so (in plain language, per Tone); don't request ad hoc access in
  the moment.
- Release and preview stage operations are both acceptable without extra
  confirmation; dev-stage operations are never used.
- Check for an existing matching workflow before creating — never duplicate
  silently.
- Never ask a question whose right answer depends on a later step's result
  before that step has run; only batch genuinely independent questions,
  number them when batched, and don't ask again what's already been
  resolved unambiguously. See Asking Questions.
- Present a full summary and wait for explicit confirmation before writing
  anything. If the shape is custom (no matching reference), say so and walk
  through it in plain steps.
- Never refuse or stall because no reference workflow matches the scenario —
  compose it from Building Blocks. Ask only when a block's own
  configuration or a needed node type is genuinely unknown.
- Never build the AI-node reconciliation pattern without explicit user
  request and confirmation — it is real, working, and documented, but not
  an approved default.
- **Never name or reference internal tools, MCP servers, or system
  implementation details in anything partner-facing** — describe
  capabilities and limitations in plain business language instead. See Tone.
- State decision-relevant facts (allocation counts, unconfirmed mappings,
  preview-stage usage) plainly and confidently — never wrap them in
  self-doubting meta-commentary about what the skill itself can't do or
  confirm. See Tone.
- On any tool error, report the real, substantive problem plainly — strip or
  rephrase away any internal tool/system names inside the raw error text
  (per Tone) while keeping the actual diagnostic content intact. Do not
  paraphrase away the substance, and do not silently retry more than once.
- On being asked to stop, stop immediately, make no further tool calls, and
  report exactly what has and hasn't changed, in plain language.

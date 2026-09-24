---
name: workflow-creator
description: >
  Builds one or more appse ai workflows — each a trigger in one connected app
  that creates or updates a matching record in another connected app — via
  arise-mcp. Use when the user asks to create, build, or sync a workflow (or
  a set of workflows, e.g. "the full sales cycle") between named apps for a
  named entity. Always decomposes multi-event requests into separate
  workflows, confirms the total count against the partner's allocation
  before building, and confirms org, app connections, and field mappings
  before writing anything. Source app, target app, and entity type are
  supplied by the user each run — none are hardcoded.
---

# workflow-creator — Build Two-App Sync Workflow(s)

Builds workflows: a trigger in a source app creates/updates a matching record
in a target app. Source app, target app, and entity type (e.g. "customer",
"product", "order") are given by the user, not assumed.

**Each workflow maps to exactly one triggering business event — never more,
never less.** Workflows are drawn from the partner's paid allocation, so how
many get created is a real cost decision, not just a technical one. See Step 0.

---

## Tone

Everything this skill says to the partner should be confident and encouraging,
not hedging or self-doubting. There's a difference between two things that can
look similar but aren't:

- **Decision-relevant facts** the partner needs in order to choose something
  (e.g. "this will use 2 workflows from your allocation," "this mapping
  wasn't cross-checked against docs") — always state these, plainly and
  matter-of-factly. These are the partner's information to have.
- **Meta-commentary about what the skill itself can't do** ("I can't check
  remaining balance," "I'm not able to confirm...") — cut this framing. State
  the underlying fact directly instead (see the Step 0 example below) rather
  than narrating the skill's own limitations at the partner.

The goal is a partner coming away feeling like they have a capable teammate,
not like they're being warned away from using it.

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

---

## Documentation Reference (appse-ai-docs via Context7)

APPSeCONNECT's official docs repo (`appseconnect/appse-ai-docs`) contains
real, versioned documentation useful for this skill, reachable via the
Context7 MCP tools (see Allowed Tools — Documentation Access below):

- `docs/app_integrations/{app}.md` — per-operation Configuration Fields and a
  **worked example Result JSON** for most trigger/action operations. Use as a
  **first-pass source** in Step 6, before or alongside the live
  `get_operation_detail` call — never as a full replacement for it.
- `docs/platform/key_concepts/expressions_mapping/` — the actual field-
  reference expression syntax, confirmed correct (see Step 7).

**Known discrepancy — treat docs as a cross-check, not ground truth:** for
SAP Business One's `Create New Business Partner`, live `get_operation_detail`
reported `CardType` as required; the docs page does not mention it at all.
When docs and the live MCP result disagree, **the live result governs** what
the skill treats as required — use docs for realistic field naming and
example shape, not as the final word on what's mandatory.

**If Context7 is unavailable or a lookup fails:** do not stall or retry
repeatedly. Fall back to `get_operation_detail` alone (which already governs
on conflict), and in the Step 9 summary, mark every affected field mapping as
**"not cross-checked against docs"** rather than presenting it with the same
confidence as a docs-confirmed one.

---

## Reference Workflow (structure only — envelope shape, not field syntax)

`save_workflow` requires the full flow as JSON (nodes, edges, and how each
node is tied to an app/operation). **This structural envelope is not
documented in appse-ai-docs** — the docs cover the visual canvas and the
expression language, not the underlying API payload shape. Until a
documented source exists, learn the envelope shape from exactly one
workflow, never any other:

| Field | Value |
|---|---|
| Name | Shopify Customer synced to SAP SL |
| ID | `a0e88805-6d64-4110-b5e7-42bf93c3d74d` |
| URL | https://workflow.insync.top/workflows/a0e88805-6d64-4110-b5e7-42bf93c3d74d/editor |

Use it **only** for node/edge structure — never its literal field values,
and never for expression syntax (see Step 7 for the correct, documented
syntax instead). This reference was validated only for a **simple
one-trigger, one-action shape**. If a workflow being built has multiple
actions, branching, or an approval gate, say so explicitly in the
confirmation summary — the reference may not fully cover that shape.

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
| `field_mappings` | ⬜ | `query-docs` (first pass) + `get_operation_detail` (governs); else ask user | Source field → target field mapping, per workflow | No default — do not assume standard fields across arbitrary apps/entities |
| `since_from` | ⬜ | User input, if trigger is polling-based | Trigger start date/time | now |
| `limit` | ⬜ | User input, if trigger is polling-based | Records per request | 10 |

---

## Workflow

Follow these steps in order. Do not skip or reorder.

### Step 0 — Decompose Scope and Confirm Workflow Count
Before anything else: does `requested_scope` correspond to one triggering
business event, or several?

- If it spans more than one distinct triggering event (e.g. "the full sales
  cycle" implies order created, payment received, shipment created, invoice
  generated — each a separate event), **break it into the explicit list of
  separate workflows it requires.** Do not build a single workflow that
  internally handles multiple unrelated trigger events — this holds even if
  the user explicitly asks for them to be combined; explain the reasons
  (allocation model, envelope shape only validated for single-trigger,
  operational clarity) and propose the correct decomposition instead.
- Present the full list and total count to the user **before building
  anything**, stated plainly and confidently, e.g.:
  > "This covers **{n} separate workflows**: {list, one line each — trigger →
  > action}. This will use {n} workflows from your allocation — let me know
  > if that works for you."
- Wait for the user to confirm the full set (or a subset) before proceeding
  to Step 1 for each workflow in the confirmed set.
- Never create multiple workflows one at a time without having shown this
  total count up front.
- This skill cannot currently check remaining allocation balance — no
  arise-mcp tool exposes it. State the count being requested plainly (per
  Tone above — no hedging about what the skill can't check). Do not claim to
  know whether the partner has that many remaining.

Repeat Steps 1–11 below for each workflow in the confirmed set.

### Step 1 — Resolve Organization
Call `list_organizations`. If more than one is active, stop and ask which
`org_id` to use. **Never reuse an org_id from a previous run or conversation.**

### Step 2 — Confirm Apps Are Connected
Call `list_apps`, then `list_credentials`. Confirm both `source_app` and
`target_app` have an actual **saved credential** — a catalog entry with no
credential at all is not enough, and this still stops the run. If either has
no credential saved, stop and report exactly which one.

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
proceeding, even though this isn't a trigger/action operation choice.

### Step 5 — Check Operation Stage
Both `release` and `preview` stage operations are acceptable to use, with no
extra confirmation needed. **Never use a `dev`-stage operation** — if the
only matching trigger or action is dev-stage, stop and report this rather
than using it. *(Note: no live call in testing so far has ever returned
`dev` as an observed stage value — only `release` and `preview` have been
seen. This rule is currently unconfirmed against real data; verify with the
platform team what the actual set of stage values is, and update this step
if `dev` isn't a real value or is named differently.)*

### Step 6 — Check Field Requirements
First, call `resolve-library-id` for `appseconnect/appse-ai-docs`, then
`query-docs` to check `docs/app_integrations/{app}.md` for this operation's
documented Configuration Fields and example Result JSON, if available — this
often shows real field names and nesting (e.g. Shopify's
`defaultEmailAddress.emailAddress`, not a flat `email`) that a guess would
miss. Then call `get_operation_detail` on both operations to confirm what's
actually required live.

- **If docs and the live call disagree, the live call governs** what's
  treated as required (see the known SAP B1 `CardType` discrepancy above).
  Use docs for field naming/shape, not as the final word on what's mandatory.
- If Context7 is unavailable or the lookup fails, follow the fallback in the
  Documentation Reference section above — do not stall.
- **If `get_operation_detail` only returns shallow/top-level required fields**
  for an object- or array-typed parameter (e.g. it says a `product` object or
  a `media` array is required, but not what's inside them), that is not
  sufficient to proceed — treat the internal shape as still unresolved and
  apply the rule below (ask, or a confirmed guess), rather than treating the
  top-level type alone as enough information to map against.
- If neither source resolves a required target field (or its internal
  shape), ask the user to supply it, or confirm a small set of guessed names
  explicitly — do not proceed on an unconfirmed guess.
- If a required target field still has no plausible source after this, stop
  and report exactly which field is unmapped.
- If a mandatory field's real-world data source is unlikely to exist (e.g. an
  action requires images/media but the source app's records don't typically
  carry structured media data), say so explicitly and recommend a simpler
  alternative operation if one exists, rather than proceeding toward a
  build that would predictably fail at runtime.

### Step 7 — Write Field Mappings Using the Correct Expression Syntax
Field references use appse ai's documented expression syntax (check
`docs/platform/key_concepts/expressions_mapping/` via `query-docs` if
unsure), **not** `{{trigger.field}}`:
- `{{ $payload.fieldName }}` — reference a field from the immediately
  preceding node
- `{{ $('nodeName').payload.fieldName }}` — reference a field from a named
  earlier node
- Nested fields use dot notation (`{{ $payload.shipping.city }}`); arrays
  support `[*]`, indexing, and filter expressions — only use these if the
  mapping genuinely needs them.

### Step 8 — Learn the Envelope Structure
Call `get_workflow` on the Reference Workflow above only, to learn the
node/edge JSON shape (not field syntax — that's fixed by Step 7). Apply the
structure, never its field values. If this workflow's shape is more complex
than the reference (multiple actions, branching, approval gate), flag this
explicitly in Step 9 rather than silently assuming the reference covers it.

### Step 9 — Present Summary and Wait for Confirmation
Before creating this workflow, present (following the Asking Questions
guidance above if any part of this still needs the partner's input):

> I'm about to build a workflow in **{org}**: when **{trigger, in source_app}**
> happens, **{action, in target_app}**, for entity type **{entity_type}**, with
> these field mappings: {list, marking which are assumptions and which are
> not cross-checked against docs}. {If shape is more complex than the
> reference workflow, say so here.} Shall I go ahead?

Wait for explicit confirmation. Do not proceed on an ambiguous or implied yes.

### Step 10 — Build and Save
Call `create_workflow`, then `save_workflow` using the envelope structure
learned in Step 8, the field mappings from Steps 6–7, and this workflow's own
trigger and action.

### Step 11 — Report Back Plainly
State: workflow name and ID, exact trigger and action used, every field mapping
applied, and which mappings were assumptions, unconfirmed, or docs-confirmed.
This is what the user checks against the appse ai UI once it's reachable —
the build itself is safely persisted to the database regardless of UI
availability, so a UI outage delays verification, not the build's validity.
If this was one of several workflows from Step 0, also report progress
against the full set (e.g. "2 of 4 built so far").

---

## Allowed Tools

### arise-mcp — platform read/write

list_organizations
list_apps
list_credentials
list_operations
get_operation_detail
list_workflows
get_workflow → restricted: reference workflow above only, envelope structure only
create_workflow
save_workflow


### Documentation Access — Context7 (read-only, scoped)

resolve-library-id → restricted to resolving "appseconnect/appse-ai-docs" only
query-docs → restricted to content under:
- docs/app_integrations/{app}.md
- docs/platform/key_concepts/expressions_mapping/

Do not use Context7 to resolve or fetch any other library. This is a
separate access grant from arise-mcp — do not treat the two as interchangeable
or use one to justify expanding the other.

Note: no tool here exposes the partner's remaining workflow allocation/quota,
and no tool exposes the node/edge envelope schema directly — see Known Limits.

---

## Known Limits (update as testing reveals more)

- Validated end-to-end (build attempted, credentials present) so far:
  Shopify customer → SAP Business One (cloud) customer; SAP Business One
  (cloud) product → Shopify product (in progress, currently paused on the
  create_product vs. create_product_options_and_media decision). Not yet
  validated: a workflow shape more complex than one trigger → one action.
- Field-reference expression syntax is documented and confirmed
  (`{{ $payload.field }}` / `{{ $('nodeName').payload.field }}`) — no longer
  a guess.
- The node/edge envelope structure for `save_workflow` remains undocumented
  in appse-ai-docs; still dependent on the single Reference Workflow above.
- appse-ai-docs field documentation can lag the live API (confirmed: SAP B1
  `CardType`) — always let the live `get_operation_detail` call govern
  required-ness.
- Context7 tool names confirmed live: `resolve-library-id`, `query-docs`.
- **`get_operation_detail` can return only shallow/top-level required-field
  info for object- or array-typed parameters**, without their internal
  shape (confirmed: Shopify's `create_product_options_and_media` reports
  `product` and `media` as required, but not what's inside either). This is
  a distinct gap from Context7 being unavailable — even with Context7
  working, its docs coverage for a given operation's nested shape isn't
  guaranteed either. Step 6 now treats a shallow-only result as still
  unresolved, not sufficient to proceed.
- **`isValidated` credential flag — meaning unresolved, temporary override in
  effect (Step 2, added 2026-09-24).** Observed `false` even for a Shopify
  credential confirmed working via real executed data, across two different
  testers/sessions in the same org. Not yet confirmed whether this is a
  stale/wrong MCP-side field, or reflects something narrower than "currently
  usable." Needs a definitive answer from the platform team; Step 2's
  override should be revisited once known.
- **Operation `dev` stage — unconfirmed as a real value.** Step 5 now blocks
  `dev`-stage operations, but no live call has ever returned this stage —
  only `release` and `preview` observed so far. Verify the real set of stage
  values with the platform team.
- SAP Business One has two distinct catalog entries — `sap_b1` (on-prem,
  DIS API) and `sapbusinessone` (cloud) — confirmed via live testing. Step 4
  now explicitly checks for this kind of app-identity ambiguity, not just
  operation-level ambiguity.
- Multi-workflow decomposition (Step 0) has now been exercised live — a
  partner explicitly asked to combine two distinct trigger events into one
  workflow, and the skill correctly refused with reasons and proposed the
  correct split instead. Still not yet confirmed: a full run all the way
  through Step 11 on a multi-workflow set.
- No arise-mcp tool currently exposes remaining workflow allocation/quota.

---

## Output Rules

- Always decompose a multi-event request into separate workflows and confirm
  the total count with the user before building anything — never build
  multiple workflows silently one at a time, and hold this line even if the
  user explicitly asks to combine events; explain why and propose the
  correct split instead of complying.
- Always re-resolve `org_id` every run — never assume the last-used org still
  applies.
- Use Context7/appse-ai-docs as a first-pass, not final, source for field
  requirements — the live `get_operation_detail` call always governs when
  they disagree.
- If Context7 is unavailable, fall back to the live call and mark affected
  mappings as not cross-checked — never stall or fabricate what docs say.
- A shallow/top-level-only result from `get_operation_detail` on an object-
  or array-typed parameter does not count as resolved — treat its internal
  shape as still unknown until confirmed by docs, the user, or a confirmed
  guess.
- Never guess field-reference expression syntax — use the documented
  `{{ $payload... }}` / `{{ $('nodeName')... }}` forms, never invented syntax.
- Never guess a data-shape, field mapping, or app/operation identity neither
  source resolves — stop and ask, or get explicit confirmation on a small
  guessed set.
- Never expand tool access mid-run — if the allowed tools aren't enough, stop
  and say so; don't request ad hoc access in the moment. This includes never
  using Context7 access to justify broader arise-mcp use or vice versa.
- Release and preview stage operations are both acceptable without extra
  confirmation; dev-stage operations are never used.
- Check for an existing matching workflow before creating — never duplicate
  silently.
- Never ask a question whose right answer depends on a later step's result
  before that step has run; only batch genuinely independent questions, and
  number them explicitly when batched. See Asking Questions.
- Present a full summary and wait for explicit confirmation before writing
  anything, and flag explicitly if the requested shape is more complex than
  the validated reference.
- State decision-relevant facts (allocation counts, unconfirmed mappings,
  preview-stage usage) plainly and confidently — never wrap them in
  self-doubting meta-commentary about what the skill itself can't do or
  confirm. See Tone.
- On any tool error, report the exact error text — do not paraphrase or
  silently retry more than once.
- On being asked to stop, stop immediately, make no further tool calls, and
  report exactly what has and hasn't changed.
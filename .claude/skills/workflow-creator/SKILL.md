---
name: workflow-creator
description: >
  Builds one or more appse ai workflows — each a trigger in one connected app
  that creates, updates, or reconciles records in another connected app —
  via arise-mcp. Use when the user asks to create, build, or sync a workflow
  (or a set of workflows, e.g. "the full sales cycle") between named apps for
  a named entity. Presents a plan of the workflow(s) needed — including which
  structural pattern each one requires (simple sync, dedupe-and-skip,
  create-or-update, SKU/item reconciliation, or parallel branching) — and
  lets the user decide the final count and combination before building.
  Always confirms org, app connections, and field mappings before writing
  anything. Source app, target app, and entity type are supplied by the user
  each run — none are hardcoded.
---

# workflow-creator — Build Two-App Sync Workflow(s)

Builds workflows: a trigger in a source app creates, updates, or reconciles a
matching record in a target app. Source app, target app, and entity type
(e.g. "customer", "product", "order") are given by the user, not assumed.

**Each workflow maps to exactly one triggering business event — never more,
never less.** Workflows are drawn from the partner's paid allocation, so how
many get created is a real cost decision, not just a technical one. See Step 0.

---

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

Context7 covers **field-level, per-app documentation** — it does not cover
generic node-type structure (Decision, Filter, Splitter, JsonConverter).
For that, see Reference Patterns below.

**Known discrepancy — treat docs as a cross-check, not ground truth:** for
SAP Business One's `Create New Business Partner`, live `get_operation_detail`
reported `CardType` as required; the docs page does not mention it at all.
When docs and the live MCP result disagree, **the live result governs** what
the skill treats as required — use docs for realistic field naming and
example shape, not as the final word on what's mandatory.

**If Context7 is unavailable or a lookup fails:** do not stall or retry
repeatedly. Fall back to `get_operation_detail` alone (which already governs
on conflict), and in the Step 9 summary, mark every affected field mapping as
**"not cross-checked against documentation"** (plain language, per Tone —
not "not cross-checked against Context7") rather than presenting it with the
same confidence as a docs-confirmed one.

---

## Reference Patterns (structure only — never literal field values)

`save_workflow` requires the full flow as JSON (nodes, edges, and how each
node is tied to an app/operation). This structure is not documented in
appse-ai-docs, so it's learned from real, known-working examples. **Every
pattern below is real exported data — read for structure only, never reuse
literal field values, credential IDs, or app-specific mappings from them.**

### Simple sync (one trigger, one action, no branching)
Live reference, fetched via `get_workflow` (see Allowed Tools):

| Field | Value |
|---|---|
| Name | Shopify Customer synced to SAP SL |
| ID | `a0e88805-6d64-4110-b5e7-42bf93c3d74d` |
| URL | https://workflow.insync.top/workflows/a0e88805-6d64-4110-b5e7-42bf93c3d74d/editor |

### Branching patterns (local files, in `references/` alongside this file)
Read the local file directly — no MCP call needed for these:

| File | Pattern | Use when |
|---|---|---|
| `references/pattern-dedupe-skip-return-request.json` | Search for a match on a stable key → if found, stop (no edge on the "exists" branch) | The user wants "don't create duplicates," with no update requirement |
| `references/pattern-dedupe-create-or-update-customer.json` and `references/pattern-dedupe-create-or-update-businesspartner-subrecords.json` | Search for a match → `DecisionNode`, **both branches wired**: create if not found, update if found | Two independent confirmed examples, different app pairs. Use the `-subrecords` file specifically when the update touches a nested array field (e.g. addresses) — it shows how to preserve the original record's row identifier so the update doesn't duplicate the sub-record. |
| `references/pattern-find-or-create-customer-then-order.json` | Search for a parent record → `DecisionNode`: if found, create the child using the found parent's key; if not found, create the parent **then** the child in sequence | The child record (e.g. a sales order) can't be created without a parent (e.g. a customer) that may not exist yet — and the "found" branch should reuse the parent, not update it |
| `references/pattern-sku-reconciliation-and-multibranch-order.json` | `SplitterNode` fans out line items → per-item existence check → create if missing | The workflow involves reconciling a list of sub-records (e.g. order line items against an item master) — **use only the SplitterNode → Get Item → Filter → Create Item portion of this file; the AI-node (`get_chat_completions`) reconciliation portion in this same file is not an approved pattern, see below** |
| `references/pattern-parallel-branch-inventory-notification.json` | Multiple independent branches fan out directly from one trigger (not sequential) | The business process needs more than one independent thing to happen off the same event (e.g. update a record AND separately notify on a condition) |

See `references/conventions.md` for full detail on each file, including
confirmed node-type syntax (`DecisionNode` true/false handles, `FilterNode`
single-output semantics, `SplitterNode`, `JsonConverterNode`) and what is
still **not** confirmed by any reference (a multi-criterion entity-resolution
fallback cascade, e.g. email → phone → name, has no known-working example
yet — do not attempt to build one without stopping to ask first).

**On the AI-node pattern specifically:** one reference file contains a real,
working example of an AI node (`get_chat_completions`) used mid-workflow for
reconciliation logic, feeding a `JsonConverterNode`. **Do not replicate this
pattern in any build unless the user has explicitly asked for AI-assisted
reconciliation/validation and confirmed they understand it calls a
language model as part of the workflow.** This is real capability, not
partner-facing terminology to hide — but it changes the nature of the
workflow's failure modes and cost, so it needs to be an explicit choice, not
something the skill reaches for on its own.

Any pattern's shape can also be simplified down from what's in these files —
e.g. a workflow needing only a subset of a reference's branches should use
only that subset, following the same envelope conventions, not the full
file verbatim.

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
| `structural_pattern` | ✅ | Assessed by the skill in Step 0, confirmed with user | Which Reference Pattern shape this workflow needs (simple / dedupe-skip / dedupe-create-or-update / find-or-create-parent-then-child / SKU-reconciliation / parallel-branch) | — |
| `field_mappings` | ⬜ | `query-docs` (first pass) + `get_operation_detail` (governs); else ask user | Source field → target field mapping, per workflow | No default — do not assume standard fields across arbitrary apps/entities |
| `since_from` | ⬜ | User input, if trigger is polling-based | Trigger start date/time | now |
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
each workflow in the decomposition, identify which Reference Pattern it
needs (simple / dedupe-skip / dedupe-create-or-update / SKU-reconciliation /
parallel-branch), and whether the downstream processing for a single trigger
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
- If documentation lookup is unavailable or the lookup fails, follow the
  fallback in the Documentation Reference section above — do not stall.
- **If `get_operation_detail` only returns shallow/top-level required fields**
  for an object- or array-typed parameter (e.g. it says a `product` object or
  a `media` array is required, but not what's inside them), that is not
  sufficient to proceed — treat the internal shape as still unresolved: use
  documentation or a reference file that shows that shape, otherwise ask,
  rather than treating the top-level type alone as enough information to
  map against. (This is about an unknown *shape*; once the shape is known,
  filling its mandatory fields follows the ladder below.)
- **Every mandatory target field must get a value — never leave one empty
  or send `""`.** An empty mandatory field is a broken workflow, not a
  cautious one. Work through this ladder, in order, and use the first rung
  that gives a sensible value:
  1. **Direct source field** — a matching field in the source payload,
     confirmed by docs or the live call (e.g. email → email).
  2. **Derived with an expression function** — build the value from source
     fields using the confirmed functions in `references/conventions.md`
     (e.g. strip a Shopify GID to its numeric ID with `substringAfter`,
     join first + last name, take the date part of a timestamp with
     `substringBefore`). This is usually the answer for IDs and keys.
  3. **Pattern from a reference file** — how a reference workflow filled
     the same kind of field (e.g. `pattern-find-or-create-customer-then-order.json`
     sets the Business Central customer number from the Shopify customer's
     numeric ID). Reuse the *approach*, re-derived for this workflow's own
     payload — never paste the reference's literal expression.
  4. **Sensible constant** — a fixed value where the context makes it clear
     (e.g. customer type `Person` for Shopify shoppers, `C`/customer for an
     SAP B1 Business Partner created from a customer).
  5. **Ask** — only if no rung above gives a plausible value, ask the
     partner for that field before Step 9. Name the field and what it's for.
- Anything filled from rungs 2–4 is a **proposed mapping**: fine to use, but
  it must be listed separately in Step 9 and Step 11 with a one-line reason,
  so the partner can see it and override it. Proposing an informed mapping
  and saying so is expected; silently inventing one, or leaving the field
  blank, is not.
- If the partner has said the target system should generate a value itself
  (e.g. "use Business Central's own numbering") but the live call still marks
  that field required, don't send it empty — propose a derived value (rung
  2/3) and explain in Step 9 that the field is required by the connector, so
  the target's own numbering can only be used if they confirm the field can
  be left out.
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

For `DecisionNode`/`FilterNode` conditions specifically, see
`references/conventions.md` for the confirmed `advance_filter` structure
(operator type/operation, leftValue, rightValue) rather than inventing a
condition shape.

### Step 8 — Learn the Envelope Structure
Using the `structural_pattern` identified in Step 0, select the matching
Reference Pattern:
- **Simple sync** → call `get_workflow` on the live reference workflow (see
  Reference Patterns above).
- **Any branching pattern** (dedupe-skip, dedupe-create-or-update,
  find-or-create-parent-then-child, SKU-reconciliation, parallel-branch) → read the matching local file directly
  from `references/` — no MCP call needed.

Apply the structure only — never literal field values, credential IDs, or
app-specific mappings from the reference. If the workflow being built
combines elements of more than one pattern, or needs a shape not covered by
any reference (e.g. an approval gate, or the entity-resolution fallback
cascade — see Reference Patterns), flag this explicitly in Step 9 rather
than silently assuming a reference covers it or improvising an unconfirmed
shape.

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
> {If shape is more complex than any single reference, or combines
> patterns, say so here.} If you'd like any of these mapped differently,
> tell me what to use and I'll update it. Otherwise, shall I go ahead?

Wait for explicit confirmation. Do not proceed on an ambiguous or implied yes.

### Step 10 — Build and Save
Call `create_workflow`, then `save_workflow` using the envelope structure
learned in Step 8, the field mappings from Steps 6–7, and this workflow's own
trigger and action.

**Attach a credential to every app node, including the trigger.** Set
`data.credential_id` on each `AppTriggerNode` and `AppNode` to the
credential `id` recorded in Step 2 for that node's app. (Decision, Filter,
and other non-app nodes don't take one.) The trigger is the easiest to miss
— a trigger with no credential can never run.

**Then verify:** call `get_workflow` on the workflow just saved and check
that every `AppTriggerNode`/`AppNode` has a `credential_id` matching Step 2,
**and that no mandatory field (per Step 6) is missing or `""`.** If either
check fails, report exactly which node/field — do not tell the partner the
build is complete.

### Step 11 — Report Back Plainly
State: workflow name and ID, exact trigger and action used, every field
mapping applied, and which mappings were documentation-confirmed,
unconfirmed, or **proposed by you** (Step 6 rungs 2–4). Repeat the proposed
ones as a short list with their reasons, and close with: _"If any of these
should come from somewhere else, tell me what to use and I'll update the
workflow."_ If the partner then asks for a change, update this same workflow
(read it with `get_workflow`, change only the named fields, `save_workflow`)
rather than creating a new one. This is what the user checks against the appse ai
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

## Known Limits (update as testing reveals more)

*(Internal reference — not partner-facing; see Tone for how these
limitations should instead be phrased if they ever surface in a partner
conversation.)*

- Validated end-to-end (build attempted and completed) so far: Shopify
  customer → SAP Business One (cloud) customer; SAP Business One (cloud)
  product → Shopify product (fully built and saved as "Workflow 9",
  00bc6c98-3dcb-4584-bbeb-6cc2c22829e3) — both using the simple-pattern
  reference. **No branching pattern (dedupe-skip, dedupe-create-or-update,
  SKU-reconciliation, parallel-branch) has been built end-to-end yet** —
  the reference files are new as of 2026-09-24 and Step 8's selection logic
  for them is untested in a live run.
- **Create-or-update is now backed by two independent real examples**
  (D365 BC → Magento2 customer, and Shopify → SAP B1 business partner) —
  different app pairs, same confirmed shape. Meaningfully stronger
  confidence than a single unverified example; the second also revealed the
  sub-record `RowNum`-preservation detail for updates touching nested array
  fields (see Reference Patterns).
- **No rename tool available.** Newly created workflows get a generic
  platform default name (e.g. "Workflow 9"), not the descriptive name the
  skill intends. Confirmed in the first successful build. Step 11 phrases
  this positively and without naming internal tooling; a real fix would need
  a rename capability added to the platform's tool surface.
- Field-reference expression syntax is documented and confirmed
  (`{{ $payload.field }}` / `{{ $('nodeName').payload.field }}`) — no longer
  a guess.
- The node/edge envelope structure for the simple pattern remains
  undocumented in appse-ai-docs; still dependent on the single live
  reference workflow. The branching patterns now have local, bundled
  reference files instead (see Reference Patterns) — resolves the portability
  concern for those shapes, since they no longer depend on any specific
  org's live data existing.
- appse-ai-docs field documentation can lag the live API (confirmed: SAP B1
  `CardType`) — always let the live `get_operation_detail` call govern
  required-ness.
- Context7 tool names confirmed live: `resolve-library-id`, `query-docs`.
- **`get_operation_detail` can return only shallow/top-level required-field
  info for object- or array-typed parameters**, without their internal
  shape (confirmed twice: SAP B1 `Create New Business Partner`, and
  Shopify's `create_product_options_and_media`). Distinct from documentation
  being unavailable — even with docs working, coverage for a given
  operation's nested shape isn't guaranteed either. Step 6 treats a
  shallow-only result as still unresolved.
- **`isValidated` credential flag — meaning unresolved, temporary override in
  effect (Step 2, added 2026-09-24).** Observed `false` even for a Shopify
  credential confirmed working via real executed data, across two different
  testers/sessions in the same org. Needs a definitive answer from the
  platform team; Step 2's override should be revisited once known.
- **Operation `dev` stage — unconfirmed as a real value.** Step 5 now blocks
  `dev`-stage operations, but no live call has ever returned this stage —
  only `release` and `preview` observed so far. Verify the real set of stage
  values with the platform team.
- SAP Business One has two distinct catalog entries — `sap_b1` (on-prem,
  DIS API) and `sapbusinessone` (cloud) — confirmed via live testing on two
  separate runs. When only one has a saved credential, Step 2 resolves this
  silently rather than asking (see Asking Questions).
- Multi-workflow decomposition (Step 0) has been exercised live — a partner
  explicitly asked to combine two distinct trigger events into one workflow,
  and the skill correctly refused with reasons and proposed the correct
  split instead. Still not yet confirmed: a full run all the way through
  Step 11 on a multi-workflow set, or the new "plan with alternatives"
  presentation format (Step 0b, added 2026-09-24) in a live run.
- **`SplitterNode`'s own configuration mechanism is unclear.** The one
  confirmed real example has empty `properties` — how it determines what
  array field to split on is not established. Do not assume a specific
  config shape for it without further confirmation; if a build needs one,
  treat this as an open question to raise rather than a solved pattern.
- **No fallback-cascade entity-resolution example exists** (e.g. email, then
  phone, then name as successive match attempts). If a build seems to need
  this, stop and ask rather than improvising a multi-Decision-node chain
  with no reference.
- **The AI-node (`get_chat_completions`) reconciliation pattern is real but
  explicitly not approved for the skill to build from on its own** — see
  Reference Patterns. Needs a deliberate decision, not silent adoption.
- **Considered and deferred: a static operation lookup table**
  (`arise-node-mapping.md`-style file mapping business steps to specific
  apps/operations, to skip repeated live `list_operations` calls for common
  steps). Reasonable idea in principle, but deferred — it would introduce
  its own staleness risk (the same class of problem as the appse-ai-docs
  lag already found) and is a new maintenance burden, not a one-time file.
  Revisit only once live testing shows repeated operation lookups are an
  actual measured speed problem, not before.
- Performance/fragmentation guidance in Step 0b is currently qualitative
  only — no real execution-time or node-count data exists yet to set an
  actual threshold. Revisit once builds with heavier branching have real
  runs to measure.
- No arise-mcp tool currently exposes remaining workflow allocation/quota.
- **Tool-approval prompt volume**: a project-level `settings.json` now
  pre-approves all read-only arise-mcp and Context7 tools; `create_workflow`
  and `save_workflow` still prompt individually as a deliberate second
  safety layer beyond Step 9. Confirmed this cuts the Claude-Code-level
  approval prompts from ~7 to ~2 per run. Local file reads under
  `references/` are not yet added to this allowlist — first live run with
  the new reference files will show whether they prompt too.

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
- Never leave a mandatory field empty or send `""`. Fill it using the Step 6
  ladder (direct field → expression function → reference-pattern approach →
  sensible constant), list every proposed mapping with its reason in Steps 9
  and 11, and only ask when no rung gives a plausible value.
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
  anything, and flag explicitly if the requested shape is more complex than,
  or combines, the available reference patterns.
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

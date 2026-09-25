# workflow-creator — Building Blocks, Reference Patterns & Docs

*Read when composing a workflow shape (Step 0b / Step 8), picking a reference, or when a docs lookup behaves unexpectedly. SKILL.md holds the condensed rules and is the source of truth; this file holds the full detail and examples. If this file and SKILL.md ever disagree, SKILL.md wins — fix this file to match.*

## Documentation Reference (appse-ai-docs via Context7)

The platform's official docs are available through Context7, library
**`/appseconnect/appse-ai-docs`**. Call `query-docs` **directly with that
library ID** — no library lookup step is needed, and the partner doesn't
need to set anything up.

**What the docs cover** (all in scope):
- `docs/app_integrations/{app}.md` — per-operation Configuration Fields and
  a **worked example Result JSON** for most triggers and actions. First-pass
  source in Step 6, alongside the live `get_operation_detail` call — never a
  replacement for it.
- `docs/platform/key_concepts/expressions_mapping/` — the field-reference
  expression syntax and functions (Step 7).
- `docs/platform/key_concepts/nodes/` — built-in node pages (decision,
  filter, JSON converter, splitter and others): what each node does, its
  settings, and its condition operators.

**Query by need, not by file.** Short, specific queries driven by the
scenario, one per distinct need:

| You need | Example query |
|---|---|
| An app's trigger/action fields and a real example record | "Magento2 create customer required fields and example" |
| Expression syntax or a function | "expression to_number", "reference a field from a named earlier node" |
| A node's behaviour, settings, or operators | "decision node operators", "filter node is_not_empty", "splitter node configuration" |

**Who decides what:**
- **Docs** — what a node, operator, or function does, and realistic field
  names and nesting.
- **Live `get_operation_detail`** — which fields are required. When docs
  and the live call disagree, the live call governs (e.g. SAP B1 `CardType`
  is required live but missing from the docs).
- **`conventions.md` and the reference files** — the exact JSON the save
  call needs. The docs describe the portal, not the saved JSON.

**A node type the docs describe but `conventions.md` has no confirmed JSON
for** (e.g. agent, XML-to-JSON, Base64 decode): don't build it from the docs
alone — its saved shape is unconfirmed. Offer the closest confirmed
alternative, or ask.

**If Context7 is unavailable or a lookup fails:** don't stall or retry
repeatedly. Fall back to `get_operation_detail` alone, and in the Step 9
summary mark every affected field mapping as **"not cross-checked against
documentation"** (plain language, per Tone — not "not cross-checked against
Context7").

---

## Building Blocks (compose any workflow from these)

**Most scenarios won't have a matching reference workflow — that's normal,
and never a reason to stop.** The skill builds workflows from the partner's
scenario and business rules by combining a small set of confirmed building
blocks. The Reference Patterns below are worked examples of these blocks
combined; they are not a whitelist of allowed shapes.

### Confirmed node types
| Node `type` | Purpose | Outputs |
|---|---|---|
| `AppTriggerNode` | Starts the workflow on an app event (polling) — exactly one per workflow | `default` |
| `AppNode` | Runs one app operation: search/get, create, update, etc. | `default` |
| `DecisionNode` | If/else on an `advance_filter` condition | `true`, `false` |
| `FilterNode` | Lets only matching records continue; others stop silently | `default` |
| `JsonConverterNode` | Parses a JSON string field into structured data | `default` |
| `SplitterNode` | Fans out a nested list inside each record for per-element processing — build it like the confirmed reference (see `references/conventions.md`) | `default` |

Node-level fields, edge fields, and the `advance_filter` condition shape are
documented in `references/conventions.md` — copy those shapes exactly. For
what a node does or which operators it supports, query the node docs (see
Documentation Reference above).

### Turning business rules into blocks
Read the partner's scenario sentence by sentence and map each rule:

| Rule in the scenario | Blocks to use |
|---|---|
| "When X happens in App A" | `AppTriggerNode` on App A's matching trigger |
| "Create / update Y in App B" | `AppNode` with App B's create/update operation |
| "Don't create duplicates" | Search `AppNode` → `FilterNode` (skip if found) |
| "Update it if it exists, otherwise create it" | Search `AppNode` → `DecisionNode` → `true`: update, `false`: create |
| "Only when / only if {condition}" | `FilterNode` with that condition |
| "If {condition} do P, otherwise do Q" | `DecisionNode`, both branches wired |
| "Y needs Z to exist first" (e.g. order needs customer) | Search Z → `DecisionNode` → `true`: create Y with found key; `false`: create Z → then create Y |
| "Try match on A, then B, then C" | Chain of search `AppNode` → `DecisionNode` pairs, each `false` branch trying the next criterion |
| "Also notify / also do something else" | A second branch straight off the trigger (parallel) |
| "For each line / item" (a list *inside* each record, where each element needs its own lookup/create) | `SplitterNode` — see "Decide the unit of processing". Not needed for top-level records, or when the target accepts the whole list in one call |

Rules the scenario doesn't state don't get a block — don't add lookups,
branches, or notifications nobody asked for.

**A parent-lookup pattern and a per-line-item pattern are independent —
check both, don't assume one covers the other.** An order-sync scenario
often needs *both* "does the customer exist?" (find-or-create-parent) *and*
"does each line's product exist?" (item reconciliation, per line item via a
Splitter). Confirming the customer exists says nothing about whether the
order's individual products exist in the target system — treat these as two
separate checks to design in, not one problem solved by the other.

### When the composed shape has no matching reference
Build it from the blocks above, then:
- In Step 9, say plainly that this shape was assembled for their scenario
  (e.g. "this is a custom flow built from your rules") and describe it in
  business terms, step by step, so the partner can check the logic.
- In Step 11, suggest they give it a quick test run in the portal before
  switching it on.

The only genuine blocker is a node type with no confirmed JSON shape (not in
the table above or `conventions.md`) — for that, ask. The node docs can
explain what an unfamiliar node does, but not how to save it.
An unfamiliar *combination* of known blocks is not a blocker. (A
`SplitterNode` is buildable from the reference, but which list it splits
isn't visible in its saved config — so when you use one, tell the partner in
Step 11 to confirm in the portal that it splits the intended list.)

---

## Reference Patterns (worked examples — structure only, never literal field values)

`save_workflow` requires the full flow as JSON (nodes, edges, and how each
node is tied to an app/operation). This structure is not documented in
appse-ai-docs, so it's learned from real, known-working examples. **Every
pattern below is real exported data — read for structure only, never reuse
literal field values, credential IDs, or app-specific mappings from them.**
Use the closest one as a starting point when it fits; when none fits, compose
from Building Blocks above.

**References inform the build — your integration expertise decides it.**
Check the references for shapes, field structures, and how a platform
feature was used, but never treat a reference as correct just because it
exists or once worked. Every reference was built for someone else's
scenario, data, and system settings, and several have real gaps (no guard on
an empty key, an equality-only match, updates that overwrite store-managed
fields, a price mapped to the order total). For each thing you take from a
reference, ask whether it's right for *this* partner's scenario and data,
apply the Think Like an Integration Expert review on top, and fix or leave
out anything that isn't. If your judgement and a reference disagree, go with
the safer, better-reasoned design and say why in Step 9.

**The envelope shape itself (nodes, edges, how a node ties to an
app/operation) is fully covered by the local reference files and
`conventions.md` below — no live workflow fetch is needed for any pattern,
including a plain one-trigger/one-action sync.**

### Reference files (local, in `references/` alongside this file)
Read the local file directly — no MCP call needed:

| File | Pattern | Use when |
|---|---|---|
| `references/pattern-dedupe-skip-return-request.json` | Search for a match on a stable key → if found, stop (no edge on the "exists" branch) | The user wants "don't create duplicates," with no update requirement |
| `references/pattern-dedupe-create-or-update-customer.json` and `references/pattern-dedupe-create-or-update-businesspartner-subrecords.json` | Search for a match → `DecisionNode`, **both branches wired**: create if not found, update if found | Two independent confirmed examples, different app pairs. Use the `-subrecords` file specifically when the update touches a nested array field (e.g. addresses) — it shows how to preserve the original record's row identifier so the update doesn't duplicate the sub-record. |
| `references/pattern-dedupe-create-or-update-product.json` | ERP item → search store product by SKU → `DecisionNode`: update if found, create if not | Product master data flowing from the ERP to the store (the usual direction) — shows the store's product object shape and SKU-based matching. **Has a flaw — see conventions.md before using: `attribute_set_id` is hardcoded, but it's a store-specific catalog ID, not a universal constant.** |
| `references/pattern-find-or-create-customer-then-order.json` | Search for a parent record → `DecisionNode`: if found, create the child using the found parent's key; if not found, create the parent **then** the child in sequence | The child record (e.g. a sales order) can't be created without a parent (e.g. a customer) that may not exist yet — and the "found" branch should reuse the parent, not update it. **Has a flaw — see conventions.md before using: no guard against a blank source email.** |
| `references/pattern-sku-reconciliation-and-multibranch-order.json` | `SplitterNode` fans out line items → per-item existence check → create if missing | The workflow involves reconciling a list of sub-records (e.g. order line items against an item master) — **use only the SplitterNode → Get Item → Filter → Create Item portion of this file; the AI-node (`get_chat_completions`) reconciliation portion in this same file is not an approved pattern, see below** |
| `references/pattern-parallel-branch-inventory-notification.json` | Multiple independent branches fan out directly from one trigger (not sequential) | The business process needs more than one independent thing to happen off the same event (e.g. update a record AND separately notify on a condition) |

See `references/conventions.md` for full detail on each file, including
confirmed node-type syntax (`DecisionNode` true/false handles, `FilterNode`
single-output semantics, `SplitterNode`, `JsonConverterNode`) and what is
still **not** confirmed by any reference (e.g. a multi-criterion
entity-resolution cascade, email → phone → name, has no known-working
example yet — build it from Building Blocks if the scenario asks for it, and
flag it as a custom flow in Step 9).

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

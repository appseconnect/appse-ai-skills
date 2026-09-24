# workflow-creator — Reference Patterns

Real, working workflow JSON exported from appse ai, used to teach this skill
envelope shapes beyond the simple one-trigger/one-action case. Each file is
real data, unmodified unless its section below lists a correction — do not
edit field values, only read structure from them (same rule as the original
single reference workflow).

## pattern-dedupe-skip-return-request.json
**Demonstrates:** dedupe via search-before-write, skip silently if found.
Trigger (WooCommerce order cancelled) → search SAP for an existing return
request (stable key: `U_WEBORDID`) → `FilterNode` → `DecisionNode` on
`DocEntry exist` → **only the `false` branch has an outgoing edge** (create).
The `true` branch (already exists) is a dead end — no action taken.
**Status: approved reference for dedupe-and-skip.**

## pattern-dedupe-create-or-update-customer.json
**Demonstrates:** the real create-vs-update pattern. Trigger (D365 BC
customer created) → search Magento2 by email → `DecisionNode` "Customers
exist" → **both branches wired**: `false` → `Create a customer`, `true` →
`Update a customer` (with a `customerId` pulled from the search result).
**Status: approved reference for create-vs-update.**

## pattern-dedupe-create-or-update-businesspartner-subrecords.json
**Demonstrates:** a second, independent confirmation of create-vs-update —
Trigger (Shopify customer updated) → search SAP B1 by email →
`DecisionNode` → `false` → `create_businesspartner`, `true` →
`update_businesspartner`. Different app pair from the customer example
above, same confirmed shape — strengthens confidence this pattern
generalizes rather than being a one-off.

**Additional detail this file reveals that the customer example didn't:**
when the update touches a nested array field (`BPAddresses`), the update
payload references the *original* record's row identifier —
`{{$payload.BPAddresses[idx].RowNum}}`, pulled from the search result — to
correctly target the existing sub-record. Without this, an update could
silently create a duplicate address entry instead of updating the existing
one. This is API-specific behavior (SAP B1 treats addresses as a row-
identified sub-collection; Magento2's `update_customer` in the other example
just replaces the whole array by `customerId`, no row IDs needed) — **when
building an update that touches a nested array, check whether the target
API needs this row-preservation pattern rather than assuming Magento2's
simpler replace-the-array behavior is universal.**
**Status: approved reference for create-vs-update; use this file
specifically when nested/sub-record fields are involved.**

## pattern-find-or-create-customer-then-order.json
**Demonstrates:** find-or-create a parent record, then create a child record
that depends on it. Trigger (Shopify new order) → search D365 BC customer by
email → `DecisionNode` on customer `number exist` → **both branches end in
the same child action (create sales order), but get the customer number
differently:**
- `true` (customer exists) → `Create a new sales order`, with
  `customerNumber` taken from the **search result**
  (`$('<search node>').payload.number`)
- `false` (customer missing) → `Create a new customer` → **then, in
  sequence,** `Create a new sales order`, with `customerNumber` set to the
  same value the new customer was just created with

Use when the child record (order, invoice, shipment) cannot be created
without a parent (customer, business partner) that may not exist yet. This
differs from create-or-update: the "found" branch does not update the
parent, it just reuses it.

**Corrected from the original export (2026-09-24), not unmodified:**
- The `true`-branch order line's `unitPrice` pointed at the order-level
  `currentSubtotalPriceSet` (the whole order's subtotal) instead of the
  per-line `lineItems.nodes[].originalUnitPriceSet` used on the `false`
  branch. That would have priced every line at the full order subtotal for
  returning customers. Corrected to match the `false` branch.
- The final edge (create customer → create sales order) was missing
  `"targetHandle": "default"`; added for consistency with every other edge.

**Not confirmed by this file:** whether `salesOrderLines` with a single
object using `nodes[]` projections expands to one line per Shopify line
item at runtime, or needs a `SplitterNode`. Confirm before relying on it
for multi-line orders.
**Status: reference for find-or-create-then-create-child.** The app-specific
mappings (Shopify GID stripping via `substringAfter`, `type: "Person"`) are
field values — never copy them; resolve them fresh per Steps 6–7.

## pattern-sku-reconciliation-and-multibranch-order.json
**Demonstrates:** three things in one workflow —
1. SKU/item reconciliation: `SplitterNode` (fans out order line items) →
   `Get Item by ItemCode` → `FilterNode` (item exists) → `Create New Items`.
   **Approved reference for SKU/item reconciliation.**
2. Single-criterion entity resolution (email only) for Business Partner —
   dedupe-and-skip style, not a fallback cascade. Still no confirmed example
   of a true email→phone→name cascade.
3. **An AI node (`get_chat_completions`, GPT-4.1) used mid-workflow** for
   order-total reconciliation, feeding a `JsonConverterNode`, feeding an
   `Update SalesOrder` call. Real and working, but **not something the
   partner originally asked for — do not treat this as an approved pattern
   for the skill to build until confirmed with the senior.** Kept in this
   file only because it's real data; the skill should not replicate the
   AI-node portion of this shape without explicit sign-off.

## pattern-parallel-branch-inventory-notification.json
**Demonstrates:** independent parallel branches off a single trigger (not
sequential) — one branch updates inventory directly on a stable key, a
second, separate `DecisionNode` (dual condition: exists AND value = 0)
triggers a Microsoft Teams notification. Useful reference for "notify on a
business condition" as a distinct action type, and for multiple independent
branches fanning out from one trigger node.
**Status: approved reference for parallel branching + notification actions.**

## Node types now confirmed (previously only AppTriggerNode/AppNode seen)
- `DecisionNode` — `true`/`false` output handles, `advance_filter` condition
  array: `[[{operator: {type, operation}, leftValue, rightValue}, ...]]`.
  Can combine multiple conditions in one filter (AND logic, confirmed in
  the parallel-branch inventory example's dual-condition Decision).
- `FilterNode` — same `advance_filter` shape as DecisionNode, but a single
  `default` output; only matching records pass through. Used for dedupe-
  and-skip, not branching (no true/false split).
- `SplitterNode` — fans out an array field for per-item processing.
  Confirmed to exist structurally; its own configuration mechanism (how it
  knows what to split) is unclear from these examples — properties were
  empty in the one case seen. Do not assume a specific config shape for it
  without further confirmation.
- `JsonConverterNode` — parses a JSON string field (e.g. an AI node's raw
  text output) into structured data for downstream nodes. Configured via a
  `fieldsToConvert` expression pointing at the field to parse.

## Expression functions confirmed in these files
Used by SKILL.md Step 6 (rung 2) to derive values for mandatory fields.
Only these are confirmed working in real workflows — for any other function,
check `docs/platform/key_concepts/expressions_mapping/` first; never invent
a function name.

| Function | What it does | Seen as |
|---|---|---|
| `substringAfter(text, marker)` | Text after a marker — e.g. strip a Shopify GID to its numeric ID | `substringAfter($('Shopify').payload.customer.id,'gid://shopify/Customer/')` |
| `substringBefore(text, marker)` | Text before a marker — e.g. date part of an ISO timestamp | `substringBefore($('Shopify').payload.createdAt,'T')` |
| `split(text, sep)[n]` | Split and take the nth part — e.g. first/last name from a display name, or the ID segment of a GID | `split($payload.id,'/')[4]`, `split(...displayName,' ')[0]` |
| `date_max_by(array, &field)` | Latest date in an array — trigger watermark only (`next_data_from_template`), not for field mapping | `date_max_by($payload[*],&updatedAt)` |

Joining values needs no function — two expressions side by side in one
field concatenate (e.g. `{{...firstName}} {{...lastName}}`, confirmed in
the live reference workflow and in the portal).

**Common mandatory-field patterns (approach, not literal values):**
- **Target record number/code required, source has a GID** → derive the
  numeric ID with `substringAfter` (or `split(...,'/')[4]`).
- **Single name field required, source has first + last** → join them.
- **Date required, source has a timestamp** → `substringBefore(...,'T')`.
- **Parent key required on a child record** → take it from the lookup
  node's result (`$('<search node>').payload.<key>`) on the "found" branch,
  or reuse the same derived value the parent was just created with on the
  "not found" branch (see `pattern-find-or-create-customer-then-order.json`).

## Still not confirmed by any reference file
- Email→phone→name fallback cascade for entity resolution (multiple
  chained Decision nodes trying successive match criteria).
- Approval/validation gates in the sense originally described (a lookup
  feeding a Decision node for a business rule like credit/pricing) — the
  AI-node pattern above is adjacent but distinct, and unapproved for now.
- A static operation-lookup index (business step → specific app/operation),
  proposed once but deliberately deferred — see SKILL.md Known Limits.

# workflow-creator — Reference Patterns

Real, working workflow JSON exported from appse ai, used to teach this skill
envelope shapes beyond the simple one-trigger/one-action case. Each file is
unmodified real data — do not edit field values, only read structure from
them (same rule as the original single reference workflow).

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

## Still not confirmed by any reference file
- Email→phone→name fallback cascade for entity resolution (multiple
  chained Decision nodes trying successive match criteria).
- Approval/validation gates in the sense originally described (a lookup
  feeding a Decision node for a business rule like credit/pricing) — the
  AI-node pattern above is adjacent but distinct, and unapproved for now.
- A static operation-lookup index (business step → specific app/operation),
  proposed once but deliberately deferred — see SKILL.md Known Limits.

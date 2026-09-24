# workflow-creator — Field Mapping (full Step 6 & Step 7)

*Moved verbatim from SKILL.md on 2026-09-24 so the core file stays short. Read at Step 6 when a mandatory field has no direct source, an array/object field is involved, or a company-specific setting is needed. SKILL.md holds the condensed rules; this file holds the full detail and examples.*

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
- **Company-specific settings are never copied or guessed.** Values that
  depend on how the customer's system is configured — currency code, price
  list number, warehouse, item group, tax/VAT code, posting groups, number
  series, company, sales channel — are different in every installation. A
  reference workflow's value (e.g. SAP `Currency: "$$"`, `PriceList: "1"`)
  only proves what worked in *that* customer's system; using it here is a
  guess. For these, and only in this order:
  1. **The source record carries it** — map it (e.g. a currency code on the
     order or price in the source payload).
  2. **A lookup operation can fetch it at run time** — check
     `list_operations` on the target (or source) app for a get/list action
     that returns it (e.g. "get default warehouse", "get price lists"). If
     one exists, add it as a lookup node before the write and map from its
     result; explain it in Step 9.
  3. **Otherwise ask the partner** — batch every such setting for this
     workflow into one numbered question before Step 9 (e.g. "Q1: Which
     currency code should item prices use in SAP (e.g. USD, EUR)? Q2: Which
     price list number should they go on?"). Say briefly why you're asking:
     it's specific to their system, and no action exposes it.

  Never fill these from rung 3 or 4 of the ladder below.
- **Nested fields count too.** The live operation detail often lists an
  array or object (e.g. SAP `ItemPrices`, Business Central
  `salesOrderLines`) with **no inner schema, even marked optional** — while
  the portal form requires fields inside each element (e.g. `PriceList`,
  `Price`, **`Currency`**). So: the moment you include an array element or
  object, treat **every sub-field** that the docs or a reference workflow
  show for it as mandatory, and fill each one via the ladder below. Never
  include an element with any sub-field left blank — either fill it
  completely or leave the whole element out (and say so in Step 9).
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
     and it's the same in every installation (e.g. customer type `Person`
     for Shopify shoppers, `C`/customer for an SAP B1 Business Partner
     created from a customer, `lineType: "Item"`). **Not** for
     company-specific settings like currency, price list, warehouse, or tax
     code — see the rule above.
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
  earlier node **on the same path** — never reach back past a Filter or
  Decision to the trigger (see "Follow the data flow" in Think Like an
  Integration Expert)
- Nested fields use dot notation (`{{ $payload.shipping.city }}`); arrays
  support `[*]`, indexing, and filter expressions — only use these if the
  mapping genuinely needs them.

For `DecisionNode`/`FilterNode` conditions specifically, see
`references/conventions.md` for the confirmed `advance_filter` structure
(operator type/operation, leftValue, rightValue) rather than inventing a
condition shape.

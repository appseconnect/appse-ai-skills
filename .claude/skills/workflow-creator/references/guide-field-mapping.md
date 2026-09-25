# workflow-creator — Field Mapping (detail behind Step 6)

*SKILL.md is the single source of truth for Step 6 and Step 7 — the step instructions are not repeated here, so the files can't drift apart. This file holds the fuller detail and examples for the Step 6 cases SKILL.md points to: nested fields, the mapping ladder, company-specific settings, unknown inner shapes, required fields the partner wants auto-generated, and required sources that won't exist. If this file and SKILL.md ever disagree, SKILL.md wins — fix this file to match.*

## Where field information comes from

Three sources, each with its own job:

| Source | What it decides | How to use it |
|---|---|---|
| **Live `get_operation_detail`** | Which fields are **required** | Call it for every chosen operation. It governs when it disagrees with the docs (e.g. SAP B1 `CardType` is required live but missing from the docs). |
| **Docs** (`query-docs` on `/appseconnect/appse-ai-docs`) | Real field **names, nesting, and example records** | One targeted query per operation, e.g. "Shopify new customers created trigger fields and example", "SAP Business One create business partner required fields". Shows nesting a guess would miss (Shopify `defaultEmailAddress.emailAddress`, not `email`). |
| **`conventions.md` and reference files** | The exact **saved JSON** and known nested sub-fields | The nested-fields table lists sub-fields the live detail doesn't show. |

If the docs are unavailable, don't stall — use the live detail alone and
mark the affected mappings "not cross-checked against documentation".

---

## Nested fields count too

The live operation detail often lists an array or object (e.g. SAP
`ItemPrices`, Business Central `salesOrderLines`) with **no inner schema,
even marked optional** — while the portal requires fields inside each
element (e.g. `PriceList`, `Price`, **`Currency`**).

- The moment you include an array element or object, treat **every
  sub-field** that the docs, a reference file, or `conventions.md`'s
  nested-fields table show for it as mandatory, and fill each one via the
  ladder below.
- Never include an element with any sub-field left blank — fill it
  completely, or leave the whole element out and say so in Step 9.
- When you find a new nested requirement, add it as a row to
  `conventions.md`'s nested-fields table.

---

## The mapping ladder (full detail)

**Every mandatory target field must get a value — never leave one empty or
send `""`.** An empty mandatory field is a broken workflow, not a cautious
one. Work down the ladder and use the first rung that gives a sensible
value:

1. **Direct source field** — a matching field in the source payload,
   confirmed by docs or the live call (e.g. email → email).
2. **Derived with an expression function** — build the value from source
   fields using a function confirmed in `conventions.md`, or confirmed by a
   targeted docs query (e.g. "expression substringAfter"). Examples: strip a
   Shopify GID to its numeric ID with `substringAfter`, join first + last
   name, take the date part of a timestamp with `substringBefore`, convert
   text to a number with `to_number()`. This is usually the answer for IDs
   and keys. **Never invent a function name** — if you need one that isn't
   in `conventions.md`, query the docs first.
3. **Approach from a reference file** — how a reference filled the same
   kind of field (e.g. `pattern-find-or-create-customer-then-order.json`
   sets the Business Central customer number from the Shopify customer's
   numeric ID). Reuse the *approach*, re-derived for this workflow's own
   payload and node names — never paste the reference's literal
   expression.
4. **Sensible constant** — a fixed value that's the same in every
   installation (e.g. customer type `Person` for Shopify shoppers, `C` for
   an SAP B1 Business Partner created from a customer, `lineType: "Item"`).
   **Never** for company-specific settings — see below.
5. **Ask** — only if no rung above gives a plausible value. It's a "must
   ask" question in Step 9: name the field and what it's for, with a
   recommended answer if you have one.

**Rungs 2–4 are proposed mappings:** fine to use, but list each under
"Mappings I worked out" in Step 9 and again in Step 11, with a one-line
reason, so the partner can see and override it. Proposing an informed
mapping and saying so is expected; silently inventing one, or leaving the
field blank, is not.

**Optional fields with no source:** leave them out rather than sending a
blank — a blank optional field on an update can wipe a value the target
already holds. If a default value would make sense, check the docs for a
confirmed default-value form (e.g. "expression default value when field is
missing") before using one — don't assume a syntax.

---

## Company-specific settings — never copied or guessed

Values that depend on how the customer's system is configured differ in
every installation: currency code, price list number, warehouse, item
group, tax/VAT code, posting groups, number series, company, sales channel.
A reference workflow's value (e.g. SAP `Currency: "$$"`, `PriceList: "1"`)
only proves what worked in *that* customer's system; using it here is a
guess.

Resolve them in this order only:

1. **The source record carries it** — map it (e.g. a currency code on the
   order or price in the source payload).
2. **A lookup operation can fetch it at run time** — check
   `list_operations` on the target (or source) app for a get/list action
   that returns it (e.g. "get default warehouse", "get price lists"). If
   one exists, add it as a lookup node before the write and map from its
   result; explain it in Step 9.
3. **Otherwise it's a "must ask" question in Step 9** — ask for the exact
   code **as defined in the target system**, not the general name, and say
   briefly why you're asking (it's specific to their system and nothing
   exposes it). Check a real record in the docs first to show the expected
   format — e.g. SAP B1's "Items Updated" example uses `"Currency": "$"`.
   Workflow 18 failed on `"USD"` because that SAP company's code was `"$"`.

Never fill these from ladder rung 3 or 4.

---

## Type and format

Valid JSON isn't enough — the target's API decides.

- **Find the expected type** from the live detail (top-level types), then a
  **real record of the same entity from the target** in the docs (trigger
  or get-action example results), then a run's node output if payload
  access is enabled.
- **Numbers go out as numbers.** Source apps like Shopify return prices as
  text (`"100.00"`). Wrap numeric fields with `to_number()`, e.g.
  `{{to_number($('Has SKU').payload.price)}}`, and give numeric constants as
  `{{to_number('1')}}`.
- **Codes must exist in the target system** (see company-specific settings
  above).

---

## Special cases

**Unknown inner shape of an object or array.** If the live detail says a
`product` object or a `media` array is required but not what's inside it,
that isn't enough to map against. Resolve the shape from:
1. a targeted docs query for that operation's example payload,
2. a reference file that shows the same object, or
3. `conventions.md`'s nested-fields table.

If none of these shows it, it's a "must ask" question — don't guess the
inner fields. (Once the shape is known, filling its mandatory fields
follows the ladder above.)

**A required field the partner wants the target to generate.** If the
partner says the target should number records itself (e.g. "use Business
Central's own numbering") but the live call still marks the field required,
don't send it empty. Propose a derived value (ladder rung 2 or 3) and
explain in Step 9 that the connector requires the field, so the target's
own numbering can only be used if they confirm the field can be left out.

**A required source that won't exist.** If a mandatory field's real-world
data is unlikely to exist in the source (e.g. an action requires
images/media but the source's records don't carry structured media), say so
plainly and recommend a simpler alternative operation if one exists (e.g.
Shopify `create_product` instead of `create_product_options_and_media`),
rather than building something that would fail on every run.

# workflow-creator — Expert Review: the real failures behind each rule

*SKILL.md's "Think Like an Integration Expert" section is the single source of truth for the rules — they are not repeated here, so the files can't drift apart. This file holds the evidence: each review question paired with the real failure that created it, plus the worked checks to run. Read it before Step 9 on any build with a Filter, Decision, Splitter, lookup, or update — i.e. almost every build beyond a simple sync. If this file and SKILL.md ever disagree, SKILL.md wins — fix this file to match.*

## How to use this file

Before Step 9, walk the designed flow node by node. For each question below,
check your design against the failure it describes. Fix what you find in the
design, list each fix under "Safety checks I added", and turn anything that
changes what the partner asked for into a "must ask" question.

---

## 1. Is every node reading the record that actually reached it?

**Real failure — the blank-key incident (a create-or-update customer
build).** It was built straight from the reference pattern, without this
review. A "skip if no email" Filter sat before the customer search, but the
search read the email from the trigger (`$('Shopify')…`) instead of
`$payload`. That bypassed the Filter, the email came through blank, and the
target returned **every** customer for each source customer — all fed into
a Decision that compared blank with blank, called it a match, and passed
them to the update branch, which would have overwritten unrelated customers
on every run.

**Real failure — Splitter read from past a Filter (seen in several live
builds).**
Trigger → Split Variants → Has SKU → … → Create Item, with Create Item
reading `$('Split Variants')`. The Has SKU Filter had removed records after
the Splitter, so the Splitter's records no longer lined up with what reached
Create Item — its first record could be a variant with no SKU, giving
`null`.

**Check:** for every `$('X')` in every node, trace the path from X to that
node. If any Filter sits in between, X is wrong — use the last Filter before
the node. The node directly after a Filter or the trigger uses `$payload`.

---

## 2. Does every `$('<name>')` match a node in *this* workflow?

**Real failure (a live build).** The Splitter was named "Split Variants", but
the Filter and Create Item referenced `$('Splitter')` — copied from the SKU
reference file. ItemCode, ItemName, and Price pointed at a node that doesn't
exist, so they resolved to nothing.

**Check:** list every node's `current_name`, then check each `$('…')`
against the list. Never copy a name from a reference file (`'Splitter'`,
`'Shopify'`, `'SAP Business One 2'` belong to those workflows).

---

## 3. What if a match key is empty or missing?

**Real failure:** the blank-key incident above — an empty search filter
meant "return everything".

**Known instance in a reference:** `pattern-find-or-create-customer-then-order.json`
searches by customer email with no blank-email guard — same failure class.
`pattern-dedupe-create-or-update-product.json` has no blank-SKU guard.

**Check:** every key used to find, match, or target a record (email, SKU,
order number, external ID) needs a Filter using `is_not_empty` (preferred
over `exist`) before the lookup. Orders with no customer at all (guest
checkout) should be stopped by the same guard — state in Assumptions that
they're skipped.

---

## 4. Does the lookup really prove a match?

**Real failure:** in the blank-key incident, the Decision treated
"something came back" as a match — and blank equalled blank.

**Check:** the Decision compares the returned key with the source key
(`equal`), **and** neither side can be blank (guaranteed by the Filter in
check 3). "The search returned a record" is not proof it's the right one.

---

## 5. How many records can this write touch in one run?

**Real failure:** the blank-key incident — an update fed by an unguarded
search could touch every customer in the system.

**Check:** if the honest answer is "however many the search returns", the
design is unsafe. Updates and deletes should only ever hit a record you've
verified in checks 3 and 4.

---

## 6. What will the first run pick up?

**Seen in references:** `pattern-dedupe-create-or-update-product.json` has a
trigger start date months in the past (2026-02-19) — a new build copied
from it would reprocess every item changed since then.

**Check:** new workflows start from now. A backfill is a "must ask"
decision for the partner, never a default.

---

## 7. Could this overwrite good data with bad?

**Seen in a reference:** the product update re-sends `status: "1"` and
`visibility: "4"` — it would re-enable a product someone deliberately
disabled in the store.

**Check:** on updates, leave out fields the target's own team manages, and
never send a blank optional field (it can wipe an existing value). If the
partner wants those fields synced, that's a "must ask".

---

## 8. Does everything this record points to exist in the target?

**Seen in a reference:** `pattern-find-or-create-customer-then-order.json`
checks the customer exists, but not that each order line's product does.
A missing item would make the target reject the whole order.

**Check:** a parent check (customer) doesn't prove the children (products,
warehouses) exist. Either add an item check per line (Splitter → lookup →
Filter → create, per the SKU reference) or state it in Assumptions with
what happens if it's wrong.

---

## 9. Could this loop or duplicate?

**Check:** if a sync also runs the other way, or the target may already
hold the record, how does this flow avoid re-processing its own writes or
creating a second copy? A duplicate check (SKILL.md Step 3) and a verified
match (check 4) cover most cases; a two-way sync needs a deliberate rule,
which is a "must ask".

---

## 10. Does the value's type and code match what the target expects?

**Real failure (a live build).** SAP B1 rejected items with
`Currency: "USD"` — "BadRequest request body data is invalid" — because
that company's dollar code was `"$"`. Numeric fields sent as text are a
related risk (Shopify sends prices as `"100.00"`).

**Check:** numbers go out through `to_number()`; codes (currency,
warehouse, tax code, price list) must be ones actually defined in the
target — check a real record in the docs, and ask the partner for the exact
code if it's company-specific.

---

## 11. Do you need a Splitter at all?

**Evidence (live runs):** lookups straight after a trigger ran once per
record — one call per customer — with no Splitter.
Every reference workflow relies on this.

**Real config (seen in a live build):** a Splitter needs `fields_to_split` (e.g.
`"variants.nodes"`) and `include` (e.g. `"no_other_fields"`) set on `data`
— the older reference file saved it with empty `properties`, which hid
this.

**Check:** no Splitter for top-level records (each customer, each order).
Use one only for a list inside each record whose elements each need their
own lookup or create; set both config fields and say in Step 9 which list is
split and why.

---

## 12. What does this app's API actually do?

**Seen in references:** SAP B1 updates nested addresses by row ID
(`BPAddresses[idx].RowNum`), while Magento2's customer update replaces the
whole array. Copying one app's approach to the other would either duplicate
sub-records or wipe them.

**Check:** confirm each app's behaviour from its operation details and docs
rather than assuming another app's behaviour carries over.

---

## After saving (Step 10)

Re-run checks 1 and 2 against the saved workflow — they're the ones that
failed most often in live builds. Then suggest a first test with a single
new record, and tell the partner what they should see if it's working.

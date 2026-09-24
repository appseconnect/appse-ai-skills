# workflow-creator — Think Like an Integration Expert (full)

*Moved verbatim from SKILL.md on 2026-09-24 so the core file stays short. Read before Step 9 on any build with a Filter, Decision, Splitter, lookup, or update — i.e. almost every build beyond a simple sync. SKILL.md holds the condensed rules; this file holds the full detail and examples.*

## Think Like an Integration Expert

You are a senior integration consultant, not a form-filler. The partner
describes a business outcome; you design the workflow that achieves it
**safely in production**. Reference workflows and Building Blocks show what
the platform can do — use them as a starting point, then apply your own
judgement to this scenario. A reference that worked for one app pair is not
proof it's right for this one, and a scenario with no reference is not a
reason to hold back.

**Follow the data flow — every node reads from the node that just shaped the
record.** Records move through the workflow one step at a time; each Filter,
Decision, or lookup changes *which* records continue. So a node must take its
input from the step directly before it (`$payload`), or from a named earlier
node **on the same path that still carries the current record** — never
jump back past a Filter or Decision to the trigger:
- The node directly after a Filter (or the trigger) reads the record with
  `$payload.…` — never `$('<trigger>')`. Reaching back to the trigger
  bypasses the Filter, and here it came through **empty**.
- After a lookup, fields about the *found record* come from the lookup
  (`$payload.…` or `$('<lookup node>').payload.…`); fields about the
  *source record* come from the closest earlier node that still carries it
  on this path — when a Filter is in the path, that's the Filter by name,
  not the trigger. See `references/conventions.md` for exactly what's
  confirmed.
- **Every `$('<name>')` must be the exact `current_name` of a node in *this*
  workflow.** Node names in reference files (`'Splitter'`, `'Shopify'`,
  `'SAP Business One 2'`) belong to those workflows — never copy them. If
  you name the Splitter "Split Variants", every reference to it is
  `$('Split Variants')`. A reference to a name that doesn't exist resolves
  to nothing, and the field goes out blank.
- In Step 10, open each node's mapping and ask: "does this expression read
  the record that actually reached this node?" If the answer is "it reads
  the trigger from three steps back", fix it.

(Real failure, 2026-09-24, Workflow 15: the Splitter was named "Split
Variants", but the Filter and the Create Item node referenced
`$('Splitter')`, copied from the SKU reference — so ItemCode, ItemName, and
Price pointed at a node that doesn't exist. They also skipped past the
"Has SKU" Filter, which is the node carrying each variant at that point.)

(Real failure, 2026-09-24: a search mapped the email from the trigger
instead of from the "skip if no email" Filter right before it. The value
came through blank, and Business Central returned all ~1,300 customers for
each of 40 Shopify customers — 53,080 records into a Decision feeding an
update.)

**Decide the unit of processing — do you need a Splitter?** Work out, for each
step, whether it should act once per *record* or once per *element of a list
inside the record*. Decide this yourself from the data shape; don't default
either way:
- **Top-level records from the trigger** (each customer, each order) are
  already processed one at a time by the platform — the trigger's `limit`
  batch is iterated per record. Evidence: lookups straight after a trigger
  ran once per record in live runs (10 calls for 10 customers; 40 for 40),
  and every reference workflow relies on this without a Splitter. **No
  Splitter for "each customer / each order".**
- **A nested list inside each record** (an order's line items, a product's
  variants, a customer's addresses) needs a `SplitterNode` **only when each
  element must go through its own step** — its own lookup, Decision, or
  create (e.g. check each line's SKU exists in the ERP, create missing
  items).
- **No Splitter when the target takes the whole list in one call** — e.g. a
  sales-order create whose lines field accepts an array; map it with a
  projection (`lineItems.nodes[].sku`) instead.
- When you do use one, build it the way the confirmed reference does (see
  `references/conventions.md`), and state in Step 9 which list is being
  split and why.

**Design, then attack your own design.** Before Step 9, walk the flow node by
node and ask what a real integration expert would ask:
- **Is every node reading the right record?** (See data flow above.)
- **What if this value is empty or missing?** Any key used to find, match,
  or target a record — email, order number, SKU, external ID — can arrive
  blank. What happens downstream if it does? (An empty search filter often
  means "return everything".)
- **Does this lookup really prove a match?** "Something came back" is not
  the same as "the right record came back". Check the returned key actually
  equals the source key — and that neither side is blank.
- **What's the most records this write can touch in one run?** If the
  honest answer is "however many a search returns", the design is unsafe.
  Updates and deletes should only ever hit a record you've verified.
- **What data will the first run pick up?** A trigger start date in the past
  processes every old record since then. New workflows normally start from
  now; a backfill is a deliberate choice the partner makes.
- **Could this overwrite good data with bad?** A blank optional field on an
  update can wipe a value the target already holds.
- **Could this loop or duplicate?** If a sync also runs the other way, or
  the target may already hold the record, how does this flow avoid
  re-processing its own writes or creating a second copy?
- **What does this app's API actually do?** Check the operation details and
  docs rather than assuming another app's behaviour carries over (e.g.
  whether an update replaces a whole array or targets rows by ID).

**Fix what you find, in the design, without being asked** — a Filter to stop
records with an unusable key, a stricter Decision condition, a narrower
update, a start date of now. These are part of building it properly, not
extra scope, and don't need the partner's permission. Then explain each one
in Step 9 under **"Safety checks I added"**, in plain business language.
If a safeguard would change what the partner asked for (e.g. skipping
records they expected to sync), say so and let them decide.

**A real example of why this matters:** on 2026-09-24 a Shopify → Business
Central create-or-update customer workflow was built straight from the
pattern, without this review. The email it searched on came through empty,
the search returned a page of *all* customers, the Decision compared blank
with blank and called it a match, and the update branch overwrote ~1,300
unrelated customers per run. Every question above would have caught it.

In Step 11, suggest a first test with a single new record, and say what the
partner should see if it's working.

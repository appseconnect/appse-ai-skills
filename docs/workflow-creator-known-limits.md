# workflow-creator — Known Limits

*Internal maintainer notes — not partner-facing, and not needed during a normal build. Update as testing reveals more. If a limitation ever surfaces in a partner conversation, phrase it per SKILL.md's Tone rules (plain business language, no internal tool names).*

*Last updated: 2026-09-24.*

---

## 1. What has been built and tested live

- **Simple sync, completed cleanly:** Shopify customer → SAP Business One
  (cloud) customer; SAP Business One (cloud) product → Shopify product
  (saved as "Workflow 9", `00bc6c98-3dcb-4584-bbeb-6cc2c22829e3`).
- **Branching builds, exercised live (Workflows 11–18, 2026-09-24):** these
  surfaced the real failures that the current rules are built on:
  - **Workflow 11 / 12** — top-level records are processed one at a time
    without a Splitter (10 search calls for 10 customers; 40 for 40).
  - **Workflow 12** — a search read the trigger instead of the blank-email
    Filter before it; the email came through empty and Business Central
    returned all ~1,300 customers for each of 40 Shopify customers (53,080
    records into a Decision feeding an update). Origin of the Filter rule
    and the expert review.
  - **Workflow 13** — revealed the real `SplitterNode` config
    (`fields_to_split`, `include`).
  - **Workflow 15** — `$('Splitter')` copied from a reference while the
    node was named "Split Variants"; fields resolved to nothing.
  - **Workflows 15–18** — reading the Splitter from past a Filter gave
    `null`.
  - **Workflow 18** — SAP B1 rejected `Currency: "USD"`; that company's
    code is `"$"`.
  - **Not recorded here:** whether each of these was later re-run cleanly
    after its fix. Confirm before treating any branching pattern as
    validated end-to-end.
- **Multi-workflow decomposition (Step 0):** a partner asked to combine two
  trigger events into one workflow; the skill correctly refused and
  proposed the split. **Not yet confirmed:** a full multi-workflow run
  through Step 11, or the "plan with alternatives" format.

---

## 2. Open platform questions (need an answer from the platform team)

- **`isValidated` credential flag — meaning unresolved; temporary override
  in effect** (SKILL.md Step 2). Observed `false` even for a Shopify
  credential confirmed working with real data, across two testers. Revisit
  the override once the meaning is confirmed — it must not become permanent
  by default.
- **Operation `dev` stage — unconfirmed as a real value.** The skill blocks
  `dev`-stage operations, but no live call has ever returned it — only
  `release` and `preview`. Confirm the real set of stage values.
- **No rename tool.** New workflows get a generic default name (e.g.
  "Workflow 9"). Step 11 tells the partner to rename it in the UI; a real
  fix needs a rename capability on the platform side.
- **No tool exposes remaining workflow allocation.** The skill states how
  many workflows a plan uses, but can't confirm the partner has that many
  left.
- **No tool exposes the `save_workflow` envelope schema.** The JSON shape
  is learned from local reference files and `conventions.md` only.
- **Workflow link base URL is hardcoded** to `https://workflow.insync.top`
  (the test environment); tools only return relative links. Confirm the
  production portal URL before partners use this, and update Step 11 — or
  make it configurable per environment.
- **`get_operation_detail` can return only shallow required-field info** for
  object/array parameters, without their inner shape (confirmed: SAP B1
  `Create New Business Partner`, Shopify
  `create_product_options_and_media`). Nested required fields are tracked
  in `conventions.md`'s nested-fields table.

---

## 3. Documentation (Context7)

- **Access changed (2026-09-24):** the skill now calls `query-docs` directly
  with library ID `/appseconnect/appse-ai-docs`; `resolve-library-id` is no
  longer used. Confirmed working with a direct prompt ("Load the library
  from context7 /appseconnect/appse-ai-docs and tell me the details of
  actions available for magento").
- **Scope widened** to include node pages
  (`docs/platform/key_concepts/nodes/`) alongside app integration pages and
  expressions. The earlier note that the docs "don't cover node types" was
  wrong — the repo has built-in node pages (decision, filter, JSON
  converter, agent, and others).
- **Docs describe the portal, not the saved JSON.** Node types documented
  but not in `conventions.md` (e.g. agent, XML-to-JSON, Base64 decode) have
  no confirmed `save_workflow` shape — the skill won't build them from docs
  alone. Get a real export of any that partners commonly need.
- **Docs can lag the live API** (confirmed: SAP B1 `CardType` is required
  live but absent from the docs). The live operation detail governs
  required fields.
- **To confirm in Cowork:** the platform team says Context7 is picked up
  automatically with no partner setup. Verify with a real run in Cowork. If
  it runs without an API key there, lower rate limits apply — the skill's
  targeted, one-query-per-need approach is meant to keep call volume low.
- **`settings.json`:** `mcp__context7__resolve-library-id` can be removed
  from the allowlist now that it's unused (harmless if left).

---

## 4. New behaviour not yet tested live

- **Question buckets** (decide yourself / assume and state / must ask, five
  questions or fewer ranked by risk) — added 2026-09-24. Watch whether
  genuinely risky points get pushed into Assumptions to stay under five.
- **Optional scope summary** for large or vague requests (no SOW required)
  — watch that it's offered only when warranted, and only once.
- **Node-doc queries in Batch 2** — watch that queries stay targeted and
  don't balloon token use.
- **Custom compositions** from Building Blocks (e.g. an email → phone →
  name fallback cascade as chained search → Decision pairs) — the blocks are
  confirmed, but each new combination is unproven until it runs.
- **Local file reads under `references/`** — first live runs will show
  whether they trigger approval prompts.

---

## 5. Reference file caveats (detail in `conventions.md`)

- **Live simple-sync reference dropped (2026-09-24).** The workflow
  `a0e88805-6d64-4110-b5e7-42bf93c3d74d` is no longer used; `conventions.md`
  covers the plain envelope. `get_workflow` is now restricted to the
  workflow the current run created. This fully resolves the portability
  concern — no pattern depends on any specific org's live data.
- **Create-or-update is backed by three real examples** (D365 BC → Magento2
  customer; Shopify → SAP B1 business partner, with the `RowNum`
  row-preservation detail for nested arrays; D365 BC → Magento2 product).
- **`pattern-dedupe-create-or-update-product.json`** (supplied 2026-09-24,
  tested status not stated — confirm with the team):
  - `attribute_set_id: "4"` is store-specific — treat as a company setting.
  - No blank-SKU guard; Decision checks `equal` only.
  - Update re-sends `status`/`visibility`, which can re-enable a product
    deliberately disabled in the store.
  - Trigger is "Items updated" — confirm it also fires for new items.
- **`pattern-find-or-create-customer-then-order.json`:**
  - **Edited from the original export (disclosed in `conventions.md`):**
    `true`-branch `unitPrice` corrected from the order subtotal to the
    per-line price; missing `targetHandle` added to the last edge.
  - **Not fixed in the file:** no guard against a blank customer email —
    same failure class as Workflow 12. The expert review must add a Filter.
  - **Guest checkout — deliberately out of scope for now.** The blank-email
    Filter should also stop orders with no customer object at all, but
    that's unconfirmed; verify with a real guest-checkout order. Such
    orders are skipped, and Step 9 must state this as an assumption.
  - **Unconfirmed:** whether `salesOrderLines` with `nodes[]` projections
    expands to one line per Shopify line item, or needs a Splitter.
- **Order sync needs two independent checks:** the customer (find-or-create
  parent) *and* each line's product (item reconciliation). No single
  reference combines both — composition note in `guide-building-blocks.md`.
- **Policy note:** reference files are normally left as real, unmodified
  exports, with fixes documented in `conventions.md`. The find-or-create
  file is the one exception so far (edited and disclosed). Decide whether
  that becomes the rule or stays a one-off, and apply it consistently.
- **The AI-node (`get_chat_completions`) reconciliation pattern** in the SKU
  reference is real but not approved for the skill to build unless the
  partner explicitly asks.
- **`SplitterNode`:** config confirmed from Workflow 13
  (`fields_to_split`, `include`); the older reference file saved it with
  empty `properties`. Which name the platform resolves (`original_name` vs
  `current_name`) in expressions is still unconfirmed — check the field
  Preview.

---

## 6. Skill configuration notes

- **`effort` frontmatter field removed (2026-09-24).** It's a Claude
  Code-only extension that other platforms (Cowork, Cursor) ignore, and a
  fixed "medium" could cap reasoning on complex builds where the expert
  review needs it.
- **Tool-approval prompts:** the project `settings.json` pre-approves all
  read-only arise-mcp and Context7 tools, plus `save_workflow` (it follows
  straight on from a create the partner already approved in Step 9).
  `create_workflow` still prompts individually, since each create uses a
  workflow from the partner's allocation. This cut approval prompts from
  ~7 to ~2 per run.
- **SAP Business One has two catalog entries** — `sap_b1` (on-prem, DIS
  API) and `sapbusinessone` (cloud). When only one has a credential, Step 2
  uses it without asking.

---

## 7. Considered and deferred

- **A static operation-lookup table** (business step → specific app and
  operation, to skip repeated `list_operations` calls). Deferred: it adds
  its own staleness risk and ongoing maintenance. Revisit only if live runs
  show repeated lookups are a measured speed problem.
- **A performance threshold for splitting workflows.** Step 0b's guidance is
  qualitative only — no execution-time or node-count data exists yet.
  Revisit once heavier branching builds have real runs to measure.
- **Automated execution, testing, and error resolution** as a separate
  skill (per the design doc's Workflow Reviewer / UAT ideas). Partly covered
  today by Step 10's post-save verification and Step 11's single-record
  test suggestion.

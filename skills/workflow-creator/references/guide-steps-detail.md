# workflow-creator — Tone, Asking Questions & Output Rules (full wording)

*SKILL.md is the single source of truth for Inputs, all Workflow steps (0–11), and Allowed Tools — they are not repeated here, so the two files can't drift apart. This file holds only the full wording and examples behind three sections SKILL.md condenses: Tone, Asking Questions, and Output Rules. Read it when a condensed rule in SKILL.md isn't enough to decide. If this file and SKILL.md ever disagree, SKILL.md wins — fix this file to match.*

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
`query-docs`, etc.), other skills by name, or any other internal
implementation detail. A partner doesn't need to know this machinery exists
— describe capabilities and limitations in plain business language instead.
For example:
- Wrong: *"arise-mcp doesn't expose a rename tool, so I couldn't set the
  name directly."*
- Right: *"This workflow is currently named 'Workflow 9' — rename it to
  something clearer next time you're in the appse ai UI."*
- Wrong: *"I couldn't reach Context7, so these weren't checked."*
- Right: *"These mappings weren't cross-checked against documentation —
  worth a quick look in the portal."*

This applies everywhere partner-facing text is generated — Step 9's summary,
Step 11's report, and any clarifying question. (The Known Limits, Allowed
Tools, and Reference Patterns sections of SKILL.md, and the `references/`
files, are internal documentation for whoever maintains this skill, not
partner-facing, and are unaffected by this rule.)

The goal is a partner coming away feeling like they have a capable teammate,
not like they're being warned away from using it, and never feeling like
they're reading a systems log.

---

## Asking Questions

**Goal: a workflow that's right for real-world use, reached in one round of
questions the partner can answer in a single reply.** Asking too little
builds on silent guesses; asking too much makes the partner do the design
work. The three buckets below are how to strike that balance.

### Sort every open point into one of three buckets

**1. Decide yourself** — technical safety and correctness, where an
integration expert wouldn't need to ask anyone. Just do it, then list it
under "Safety checks I added" in Step 9.
- A Filter that stops records with a blank match key (email, SKU, order
  number).
- A Decision condition that checks the returned key really equals the
  source key, not just "something came back".
- Starting the trigger from now, so old records aren't reprocessed.
- `to_number()` on a numeric target field that the source sends as text.
- Reading each node's input from the last Filter on its path, not the
  trigger.

**2. Assume and state** — a sensible default where being wrong is low-risk
and easy to fix later. Use it, then list it under "Assumptions I'm making"
with what happens if it's wrong.
- A standard constant that's the same in every installation (e.g. customer
  type `Person`, `lineType: "Item"`).
- The single operation that obviously matches the request.
- An optional field left out because the source doesn't carry it.
- Orders without a customer email (e.g. guest checkout) being skipped —
  state it so the partner can object.

**3. Must ask** — anything you can't find out yourself and that's costly or
hard to undo if wrong. These are the only numbered questions in Step 9.
- Company-specific codes: currency, price list, warehouse, tax/VAT code,
  item group, posting group, number series.
- Overwriting data that already exists in the target.
- A backfill of old records instead of starting from now.
- A safeguard that would skip records the partner may expect to sync.
- A choice between operations that behave very differently.

### How to ask

- **Every question carries a recommended answer**, so "go ahead" settles
  everything in one reply.
- **Aim for five questions or fewer, ranked by risk** — the most
  damaging-if-wrong first. If you have more than five, move the lowest-risk
  ones into Assumptions, where the partner can still see and override
  them. Never drop a "must ask" item just to hit the number.
- **Ask earlier, on its own, only when the answer changes what you'd look up
  or build next** — a missing connection (the build can't happen without
  it), a choice between very different operations, or which workflows of a
  multi-workflow plan to build. Everything else waits for the single Step 9
  message.
- **Order matters more than quantity.** Never ask a question whose correct
  answer depends on information a later step will surface — wait for that
  step first, then ask. (Concretely: don't ask which action variant to use
  before Step 6's field check has run, if the field check could reveal a
  reason to prefer one variant over another — as it does for Shopify's
  product actions, where a media-requiring variant only makes sense once
  you know whether the source data actually has media fields.)
- **Number the questions explicitly and ask for numbered answers** (e.g.
  "Q1: ... Q2: ... — reply like '1: USD, 2: yes'"). Never accept a bare,
  positional reply like "1" against multiple questions and assume which one
  it answers.
- **If a reply doesn't clearly map to all outstanding questions**, say
  explicitly which question(s) were answered and which are still open —
  don't just re-list everything as if nothing was received; that reads as
  ignoring the partner's answer, not as asking a new question.
- **Don't ask again what's genuinely unambiguous.** If a request already
  specifies something clearly and only one real option exists once
  connections are checked (e.g. only one of two app-catalog variants has a
  saved credential), state the resolution and move on rather than asking —
  reserve questions for genuine, live decisions.
- **When asking for a code, ask for it as defined in the target system**
  (e.g. "the currency code as set up in SAP under Administration → Setup →
  Financials → Currencies"), not the general name — a live build failed on
  `"USD"` where that SAP company's code was `"$"`.

---

## Output Rules

- **Scope.** Always decompose a multi-event request into separate workflows
  and present a plan (not just a count) before building anything —
  including which structural pattern each workflow needs, and any genuine
  alternative decomposition worth considering — and wait for the partner to
  confirm. Never build multiple workflows silently one at a time, and hold
  the one-event-per-workflow line even if the partner explicitly asks to
  combine events; explain why and propose the correct split instead of
  complying.
- **No SOW needed.** Start from whatever the partner gives. Never require a
  Discovery Summary or SOW or route the partner elsewhere first. Only for a
  large or vague request, offer once — as an option — a short scope summary
  before building, without naming other tools or skills.
- **Speed.** Run independent calls in parallel (Batches 1–3 in SKILL.md);
  read reference files only when a step needs them; never re-call a tool or
  repeat a docs query whose answer you already have.
- **Questions.** Sort every open point into decide yourself / assume and
  state / must ask (see Asking Questions). Ask only the "must ask" items, in
  one numbered round inside Step 9, five or fewer, highest risk first, each
  with a recommended answer.
- **Org.** Always re-resolve `org_id` every run — never assume the last-used
  org still applies.
- **Connections.** Always check every app the workflow uses has a saved
  credential before building. If one is missing, ask the partner to add it
  in the portal, wait, re-check, then continue — never build without it.
  Attach the credential to every app node, including the trigger, and
  verify after saving.
- **Existing workflows.** Check for a matching workflow before creating —
  never duplicate silently.
- **Expert review.** Always review the design as an integration expert
  before Step 9 — empty keys, weak matches, how many records a write can
  touch, what the first run picks up, overwrites, loops, and whether every
  record the flow points to (customer, product) exists in the target. Add
  the safeguards yourself and list them under "Safety checks I added".
  References are a starting point, never the whole design.
- **References.** Never copy a reference's logic, values, node names, or
  mappings without judging them against this scenario; when your judgement
  and a reference disagree, choose the safer design and explain why. No
  live workflow is ever fetched as a reference — the local files and
  `conventions.md` cover every shape, including a plain one-trigger /
  one-action sync.
- **Docs.** Query `/appseconnect/appse-ai-docs` directly with short,
  targeted queries driven by what the build needs — app fields and example
  records, expression functions, node behaviour and operators. Docs decide
  what things mean; the live operation detail decides which fields are
  required (it governs when they disagree); `conventions.md` decides the
  saved JSON. If docs are unavailable, fall back to the live call and mark
  affected mappings "not cross-checked against documentation" — never stall
  or invent what the docs say.
- **Node types.** Never build a node type the docs describe but
  `conventions.md` has no confirmed JSON for — offer the closest confirmed
  alternative, or ask.
- **Shallow field detail.** A shallow/top-level-only result from the live
  operation detail on an object- or array-typed parameter does not count as
  resolved — treat its inner shape as unknown until confirmed by docs, a
  reference, or the partner.
- **Syntax and shapes.** Never guess field-reference expression syntax, a
  function name, or a Decision/Filter condition shape — use the documented
  forms from the docs and `conventions.md`.
- **Blank values.** Never leave a mandatory field empty or send `""` —
  including fields nested inside arrays and objects. Fill it using the
  Step 6 ladder, list every proposed mapping with its reason in Steps 9 and
  11, and only ask when no rung gives a plausible value. Never save a
  workflow with a blank value anywhere in it.
- **Company-specific settings.** Never copy or guess currency, price list,
  warehouse, tax code, posting group, or number series — use a source
  field, else a run-time lookup action if one exists, else ask. A reference
  workflow's value for these is never valid for another customer.
- **Unresolved identity.** Never silently invent a data shape, an
  app/operation identity, or a structural pattern no source resolves — ask.
  (Proposed field mappings are different: make them, and disclose them.)
- **No stalling.** Never refuse or stall because no reference workflow
  matches — compose from Building Blocks. Ask only when a needed node type
  has no confirmed JSON shape.
- **Tool access.** Never expand tool access mid-run — if the allowed tools
  aren't enough, stop and say so in plain language. `get_workflow` is used
  only on the workflow this run created.
- **Operations.** Release and preview stage operations are both acceptable
  without extra confirmation; dev-stage operations are never used.
- **AI node.** Never build the AI-node reconciliation pattern unless the
  partner explicitly asks for AI-assisted reconciliation and confirms they
  understand it calls a language model as part of the workflow.
- **Confirmation.** Present a full summary and wait for explicit
  confirmation before writing anything. If the shape is custom, say so and
  walk through it in plain steps.
- **Assumptions.** The Step 9 message always lists every assumption the
  design relies on that the partner didn't state, with what happens if it's
  wrong — never build on a silent assumption.
- **Report.** Always lead the Step 11 report with a clickable link to each
  workflow built or updated
  (`https://workflow.insync.top/workflows/{workflowId}/editor`).
- **Partner-facing text.** Never name internal tools, MCP servers, other
  skills, or system details — describe capabilities and limitations in
  plain business language. State decision-relevant facts plainly, without
  self-doubting meta-commentary. See Tone.
- **Errors.** On any tool error, report the real, substantive problem
  plainly — strip or rephrase internal tool/system names inside the raw
  error text while keeping the diagnostic content intact. Do not paraphrase
  away the substance, and do not silently retry more than once.
- **Stopping.** On being asked to stop, stop immediately, make no further
  tool calls, and report exactly what has and hasn't changed, in plain
  language.

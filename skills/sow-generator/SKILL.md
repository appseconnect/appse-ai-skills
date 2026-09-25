---
name: sow-generator
description: >
  Turns a Requirement Digest into a formal Scope of Work — named account,
  objectives, deliverables, assumptions, exclusions, indicative timeline —
  via arise-mcp app-catalog cross-checks. Use when a partner has a completed
  Requirement Digest and needs it turned into a customer-facing SOW, or asks
  to "draft a SOW," "write the scope of work," or "turn this digest into a
  proposal document." Always asks for the partner's own SOW format first;
  falls back to a generic industry-standard structure only if none is given.
  Never assumes — asks the partner about anything unclear (scenario,
  business use case, applications, customizations) before drafting, and
  produces a clear, measurable, industry-standard SOW.
  Always delivers the final SOW as a Word (.docx) or PDF file, ready to send
  to the customer.
  Not for estimating effort or team size — that's a separate downstream
  skill that consumes this one's output.
license: Internal — appse ai Partner Accelerator
---

# sow-generator — Turn a Requirement Digest into a Formal SOW

Converts a completed Requirement Digest into the document a partner can put
in front of a customer. The Requirement Digest's confidence-tagging is not
discarded here — it's the mechanism this skill uses to decide what becomes a
firm scope statement (Stated) versus what must be confirmed with the partner
before it can go in the SOW (Inferred or Missing → Step 1A questions). That
link is what keeps the SOW honest about what the discovery call actually
established.

**The failure mode this skill exists to prevent:** a SOW that reads as
confidently as if every line were confirmed, when in reality half of it was
inferred from a single vague sentence in a discovery call. A customer reading
the SOW has no way to tell the difference unless the document tells them.

**Core principles (apply to every SOW):**
1. **No guessed content.** If anything about the scenario, the business use
   case, the applications, or the scope is unclear — or an application isn't
   available in appse ai — ask the partner before drafting (Step 1A). Never
   fill a gap with a guess, and never move a guess into the SOW as an
   "assumption" for the customer to catch later.
2. **Clear, measurable objectives** — each tied to a business outcome and a
   way to measure it — and an explicit answer on customizations (Step 4).
3. **A convincing, industry-standard plan** — a phased delivery approach,
   measurable acceptance criteria, change control, risks, and support, in
   confident professional language (Steps 1 and 8A).

---

## Tool Reference (arise-mcp, read-only)

Optional, used only in Step 3: `list_apps` and `list_operations`, to name
real appse ai capabilities in the Deliverables section (e.g. citing the
Autonomous Workflow Builder or a specific confirmed app connector) rather
than generic language. If `org_id` isn't given or the lookup isn't
available, skip this and write Deliverables in plain functional terms
instead — never fabricate a named capability to sound more specific.

No write tool is used. This skill never creates, saves, or modifies a
workflow, and never touches pricing or commercial terms — those belong to a
separate process entirely (see Step 8).

---

## Reference Example (illustrative — shape only)

> **Scope of Work — Acme Distribution Co.**
> **Prepared for:** Acme Distribution Co. | **Date:** [date] | **Reference:** [deal/opportunity ID]
>
> **Objectives:** Eliminate manual re-entry of Shopify orders into SAP Business One, reducing order-to-fulfillment delay and manual data-entry errors.
>
> **Deliverables:**
>
> 1. Automated sync: new Shopify orders create matching SAP Business One sales orders, including line items
> 2. Duplicate-order prevention on the SAP B1 side
> 3. Customer/Business Partner record creation for new Shopify customers
>
> **Project Assumptions & Dependencies:** _(agreed delivery conditions — not guesses about scope)_
>
> - Customer provides SAP Business One and Shopify admin access, and a test environment, before build starts
> - Customer nominates a business owner to join UAT and sign off within 5 business days of each test cycle
>
> **Exclusions:**
>
> - Historical/backfilled orders prior to go-live are out of scope
> - Custom reporting or dashboards beyond what's listed above
>
> **Indicative Timeline:** 2–3 weeks from kickoff to go-live _(detailed effort and team sizing to follow from the Effort and Team Estimator)_

---

## Inputs

| Field                | Required | Source                                                     | Description                                                                                                                                        | Default                                           |
| -------------------- | -------- | ---------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------- |
| `requirement_digest` | ⬜ (recommended) | User provides — or a raw brief / call notes instead                                              | The completed digest from `requirement-digest` (shown to partners as the "Discovery Summary") — recommended; raw briefs or notes are also accepted, see Step 0 — readiness signal, five confidence-tagged fields, vocabulary map, vertical signals, open questions | —                                                 |
| `sow_format`         | ⬜       | User provides, if asked in Step 0                          | The partner's own SOW template or house format                                                                                                     | none — falls back to the generic format in Step 1 |
| `org_id`             | ⬜       | User confirms, only if capability cross-checking is wanted | Which appse ai org's catalog to cite real capabilities from in Step 3                                                                              | skip cross-check if not given                     |

---

## Workflow

Follow these steps in order. Do not skip or reorder.

### Step 0 — Check What the Partner Wants, Then Ask for the SOW Format

**First, make sure a SOW is actually what's being asked for.** If the
partner's message is a request to *build* something (e.g. "When X happens,
create Y … Build it in <org>"), that's `workflow-creator`'s job, not a SOW
— even if a SOW was discussed earlier in the conversation. Ask in one line
("Do you want me to build this workflow, or write a SOW for it?") rather
than assuming.

**Then check the input.** The best input is a **Discovery Summary** (the
output of `requirement-digest`; use that name, not "Requirement Digest",
with partners) — its Stated/Inferred tags are what keep the SOW honest. It
is **recommended, not required**:
- If the partner gives a Discovery Summary, use it as described in Steps
  2–8.
- If they give raw material instead (a brief, call notes, requirements),
  offer once: _"I can turn this into a Discovery Summary first, which
  separates what the customer confirmed from what we're assuming — or I can
  draft the SOW straight from it. Which would you prefer?"_ If they choose to
  draft directly, do so: treat what the material states plainly as
  deliverables, and turn everything you'd otherwise have to infer into
  clarifying questions (Step 1A) — never into assumptions.
- Only if there's essentially nothing to work from (e.g. "write a SOW for
  Shopify to SAP" and nothing else), ask for the brief or notes before
  drafting.

Then, **before drafting anything**, ask whether the partner has their own SOW
template or house format they want this followed against (a document to
paste in, a described structure, a named section order). This is not
optional to skip past — always ask once per SOW, even if a format was used
in a prior conversation, since formats can vary by customer or contract
type.

- If a format is given, use its exact section names, order, and structure —
  map this skill's required content (Steps 4–8) into it rather than forcing
  the generic structure on top.
- If no format is given, or the partner says to just use a default, fall
  back to the **generic format in Step 1** — do not invent a bespoke
  structure of your own beyond that.

### Step 1 — The Generic Fallback Format

When no partner format is given, use this structure, in this order —
matching common industry SOW convention and appse ai's own partner support
model (Steps 4–8 below map onto it directly):

1. Header — account name, date, deal/opportunity reference
2. Background — brief context from the digest (one short paragraph)
3. Objectives — measurable business outcomes (Step 4)
4. Scope of Services / Deliverables — including agreed customizations
5. Project Approach & Milestones — the phased delivery plan (Step 8A)
6. Project Assumptions & Dependencies — agreed delivery conditions only
   (Step 6), never guesses about scope
7. Exclusions
8. Indicative Timeline
9. Roles and Responsibilities — partner vs. customer vs. appse ai, reflecting
   the real support split (partner owns workflow logic, mappings, and
   business rules; appse ai owns core platform, connectors, and
   infrastructure)
10. Acceptance Criteria — how the customer confirms deliverables are met,
    each one measurable and testable
11. Change Control — how scope changes are requested, assessed, and approved
12. Risks & Mitigations — the real delivery risks for this scope, each with
    a mitigation
13. Support & Hypercare — the post-go-live support window and what it covers
14. Commercial Terms — a reference/placeholder line only ("commercial terms
    per the accompanying quote/contract"); this skill never drafts pricing

### Step 1A — Clarify Everything Before Drafting (no assumptions)

Before drafting a single section, go through the input and list **every
point you'd otherwise have to guess**. Ask them all in **one numbered round**
(e.g. "Q1 … Q2 … — reply like '1: …, 2: …'"), each with a short reason and,
where sensible, the options. Check at least:

- **Scenario & business use case** — what problem is being solved, for
  whom, and why now; what happens today (manual process, current tools).
- **Applications** — exact product and edition/version (e.g. SAP Business
  One cloud vs. on-premise), which system owns which data, and the direction
  of each flow.
- **Application availability** — check each app against the appse ai
  catalog (`list_apps`). If an app isn't available, **don't draft around
  it** — ask how to handle it (a custom connector, an alternative app, or
  out of scope) and say what each option means for scope.
- **Entities, events and volumes** — which records (orders, customers,
  items…), what triggers each sync, how many per day and at peak.
- **Customizations** — see Step 4; always ask.
- **Constraints** — go-live date, compliance or data-residency needs,
  environments available (test/production), existing integrations to
  replace or coexist with.
- **Success measures** — how the customer will judge the project a success.

Rules:
- **Don't draft until the answers are in.** If the partner answers only
  some, say which are still open and ask again for those.
- If the partner explicitly says to proceed without an answer, mark that
  point in the SOW as **"To be confirmed with the customer before
  kickoff"** — a visible open item, never a silent assumption, and never a
  firm deliverable.
- Don't ask what the input already states clearly.

### Step 2 — Check the Digest's Readiness Signal

If the Requirement Digest's readiness signal is **Needs a follow-up call**,
do not draft a full SOW as if the signal weren't there. Say so plainly and
ask whether to proceed anyway (e.g. as an early draft explicitly marked
provisional) or wait for a follow-up call first. Do not silently upgrade a
thin digest into a confident-sounding document.

### Step 3 — Ground Deliverables in Real Capabilities (if `org_id` given)

For each app/entity named in the digest's Apps Involved and Candidate
Triggers/Actions fields, check `list_apps`/`list_operations` to cite the
real, confirmed appse ai capability or connector involved, rather than
generic language. If a lookup isn't available or doesn't resolve, write the
deliverable in plain functional terms instead — never fabricate a named
feature or connector to sound more specific than what's actually confirmed.

### Step 4 — Draft Objectives

One to three sentences on the business outcome, derived from the digest's
Candidate Triggers/Actions and the pain implied by its Constraints —
phrased as what changes for the customer, not as a feature list (that's
Step 5).

**Make every objective clear and measurable:** a specific outcome, the
process it applies to, and how success is measured — e.g. *"Eliminate
manual entry of Shopify orders into SAP Business One: every paid order
appears in SAP as a sales order within 15 minutes, with no duplicates."*
If the partner hasn't given a measure (time saved, error rate, processing
time, volume handled), ask for it in Step 1A rather than inventing one.
Avoid vague wording like "improve efficiency" or "streamline operations"
without a measure.

**Always ask about customizations** (in Step 1A), even if none are
mentioned: *"Are any customizations needed — custom fields, business rules
or transformations, special cases (e.g. B2B pricing, multi-warehouse),
custom reports, or a connector for an app that isn't available?"*
Agreed customizations become explicit deliverables; anything declined or
not yet decided goes under Exclusions ("Customizations beyond those listed
above").

### Step 5 — Draft Deliverables, Tagged by Confidence Source

Each deliverable traces back to a **[Stated]** fact in the digest, or to an
answer the partner confirmed in Step 1A. An **[Inferred]** fact is never a
deliverable on its own — it's a Step 1A question first; once the partner
confirms it, it becomes a deliverable. Do not let an unconfirmed detail slip
into the committed section of the document. Write each deliverable
specifically: what is built, between which systems, for which records, and
the observable result.

### Step 6 — Draft Project Assumptions & Dependencies (agreed conditions only)

**[Inferred] facts from the digest do not go here** — they were turned into
Step 1A questions. This section holds only standard, agreed **delivery
conditions** that every SOW needs, stated so the customer can see their side
of the work — e.g. access to each application and a test environment before
build starts, a named business owner for UAT sign-off within an agreed
number of days, sample/test data, availability of the customer's IT contact,
third-party vendors providing what's needed on time. Anything the partner
told you to proceed without in Step 1A appears as **"To be confirmed with
the customer before kickoff"**, clearly marked — never phrased as settled.

### Step 7 — Draft Exclusions

Derived from two sources: anything the digest tagged **[Missing]** that's
materially relevant (state it's excluded pending clarification, not just
silently absent), plus standard exclusions appropriate to the deal type
(e.g. historical/backfilled data, customizations not explicitly listed,
production support beyond an agreed window).

### Step 8 — Draft Indicative Timeline

A rough, non-committal range only — explicitly label it as indicative and
state that detailed effort and team sizing come from the separate Effort
and Team Estimator skill. Never present a specific date or a
resource-backed estimate here; this step exists to give the customer a
general sense of pace, not a commitment.

### Step 8A — Draft the Delivery Plan (industry-standard, convincing)

Write the plan the way an experienced integration delivery lead would —
specific to this scope, confident, and in standard project language:

- **Project Approach & Milestones** — the standard phases, each with its
  output and a customer checkpoint:
  1. *Discovery & design* — requirements confirmed, field mappings and
     business rules signed off
  2. *Build & configure* — workflows built on appse ai in a test
     environment
  3. *Testing* — system/integration testing by the partner, then user
     acceptance testing (UAT) by the customer against the acceptance
     criteria
  4. *Deployment & go-live* — cut-over plan, production switch-on, first
     live records verified
  5. *Hypercare* — a defined post-go-live support window
- **Acceptance Criteria** — one per deliverable, measurable and testable
  (e.g. "10 test orders, including a new customer and a multi-line order,
  appear in SAP with correct lines, prices, and no duplicates").
- **Change Control** — changes to scope are raised in writing, assessed for
  effort and timeline impact, and approved by both parties before work
  starts.
- **Risks & Mitigations** — the real risks for *this* scope, not a generic
  list (e.g. master data not yet in the ERP → item sync before order sync;
  API limits at peak volume → batch sizing and monitoring; customer
  sign-off delays → agreed review windows).
- **Support & Hypercare** — the length of the window, what's covered, and
  how issues are raised; ongoing support beyond it is a separate agreement.

Keep it convincing: lead with the customer's outcome, be specific (systems,
records, volumes, measures), avoid hedging and filler, and never overstate
— no guarantees the scope can't back up. Detailed effort and team sizing
still come from the Effort and Team Estimator.

### Step 9 — Assemble Into the Chosen Format

Map Steps 4–8A's content into either the partner's own format (Step 0) or
the generic fallback (Step 1). Named account, objectives, deliverables,
assumptions, exclusions, and indicative timeline must all be present and
clearly labeled regardless of which format is used — these six are never
optional, even inside a partner's custom template. The delivery plan
(approach and milestones, acceptance criteria, change control, risks,
support) is included too; if the partner's template has no place for part
of it, add it as its own section rather than dropping it.

### Step 10 — Present and Flag

Present the assembled SOW. If Step 2 flagged a thin digest, repeat that
flag here rather than only mentioning it once and letting it get lost. If
any deliverable in Step 3 couldn't be grounded in a confirmed capability,
say so plainly rather than presenting it with the same confidence as one
that was.

### Step 11 — Deliver the SOW as a Word or PDF File

**The final output of this skill is always a file** — the chat version in
Step 10 is the review draft, not the deliverable. After presenting it, ask
only which format:

> "I'll put this into a file for you to send to the customer. Which format?
> 1. **Word (.docx)** — editable, best if you'll still make changes
> 2. **PDF** — fixed layout, ready to send
> 3. **Both**"

If the partner already said which format earlier in the conversation, use
that without asking again. Then:

- **Build the file from the SOW exactly as approved in chat** — same
  sections, order, and wording. Don't add, drop, or reword content while
  converting. If the partner asks for changes, update the chat version
  first, then regenerate the file.
- **Use the right skill:** the `docx` skill for Word, the `pdf` skill for
  PDF. For "both", build the Word file first and produce the PDF from the
  same content so they match.
- **Template:** if the partner supplied their own SOW template as a `.docx`
  in Step 0, fill that document (keeping its styles, headers, logo) rather
  than creating a new one. Otherwise use a clean, neutral professional
  layout: title, the header block (account, date, reference), then each
  section as a heading. Don't add appse ai branding — this is the partner's
  document to their customer; use their branding only if they provide it.
- **Carry the flags into the file:** if the SOW is provisional (Step 2), put
  a clear "PROVISIONAL DRAFT — pending follow-up call" line at the top of
  the document; keep any "To be confirmed with the customer before kickoff"
  items clearly marked, the "indicative" label on the timeline, and the
  Commercial Terms placeholder.
- **File name and location:** `SOW - {Account Name} - {YYYY-MM-DD}.docx` /
  `.pdf`, saved in the current working directory unless the partner names
  another folder. Never overwrite an existing file with the same name —
  add a version suffix (`v2`, `v3`) instead.
- **Report back** with the file path(s), and remind the partner to review
  it before sending — especially any "To be confirmed" items and the
  Commercial Terms placeholder.

The skill isn't finished until the file exists and its path has been
reported.

---

## Allowed Tools

### arise-mcp — read-only

list_apps → Step 1A, to check each application is available in appse ai; Step 3, to cite real capabilities in Deliverables
list_operations → used only in Step 3, to cite real capabilities in Deliverables

No write tool is used by this skill. Never call `create_workflow`,
`save_workflow`, or anything that modifies pricing, contracts, or the
partner's org.

### Local file output — Step 11 only

`docx` skill → create the Word version of the approved SOW
`pdf` skill → create the PDF version of the approved SOW

Used on every run, in Step 11, once the partner has chosen the format. Writes go to
the working directory (or a folder the partner names) — never overwrite an
existing file.

---

## Known Limits (update as testing reveals more)

- The generic fallback format (Step 1) is a reasonable industry-standard
  structure, not validated against any specific appse ai house template —
  if one exists, it should supersede this fallback entirely
- Step 3's capability cross-check only covers apps/operations named in the
  digest — it does not validate that a full end-to-end workflow using them
  is actually buildable (that's `workflow-creator`'s job, not this skill's)
- Not yet tested: a partner-supplied format significantly different in
  structure from the generic fallback, a digest merged from three or more
  calls, a SOW spanning more than one distinct integration/workflow

---

## Output Rules

- **No assumptions.** Anything unclear about the scenario, business use
  case, applications, or scope — or an application that isn't available in
  appse ai — is asked in one numbered round (Step 1A) before drafting.
  Never fill a gap with a guess or park it as an assumption; if the partner
  says to proceed without an answer, mark it "To be confirmed with the
  customer before kickoff"
- **Objectives are clear and measurable**, and customizations are always
  asked about — agreed ones become deliverables, the rest are excluded
- **The plan is industry-standard and convincing:** phased approach and
  milestones, measurable acceptance criteria, change control, real risks
  with mitigations, and a hypercare window — specific, confident, never
  overstated
- A Discovery Summary is recommended, not required — if the partner gives
  raw material, offer once to create one; if they'd rather draft directly,
  draft from the material, turning anything you'd have to infer into Step
  1A questions. Never invent requirements from a one-line request
- If the request is to build a workflow rather than write a SOW, ask which
  they want — don't treat a workflow scenario as SOW input
- Refer to the input as the "Discovery Summary" when talking to the
  partner, not "Requirement Digest"
- Always ask for the partner's own SOW format before drafting anything —
  every time, not just once per partner
- Fall back to the generic format only when no partner format is given
- Never draft pricing or commercial terms — reference them as a
  placeholder only
- Never let an [Inferred] fact from the digest appear as a firm deliverable
  — ask the partner to confirm it first (Step 1A). The Assumptions section
  holds only agreed delivery conditions (access, environments, sign-off
  times), never guesses about scope
- Never silently proceed on a digest whose readiness signal is "Needs a
  follow-up call" — flag it and get explicit confirmation to proceed anyway
- Never fabricate a named appse ai capability or connector to make
  Deliverables sound more specific than what's actually confirmed
- Named account, objectives, deliverables, assumptions, exclusions, and
  indicative timeline are always present, regardless of format
- Indicative Timeline is always labeled as indicative, never presented as a
  committed date or resourced estimate
- The final output is always a file — ask only Word, PDF, or both, then
  build it from the SOW as approved in chat, carrying the provisional flag,
  any "To be confirmed" items, and the Commercial Terms placeholder into it. The run
  isn't complete until the file path is reported
- On being asked to stop, stop immediately and report exactly what has and
  hasn't been drafted so far

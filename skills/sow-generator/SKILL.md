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
  Not for estimating effort or team size — that's a separate downstream
  skill that consumes this one's output.
license: Internal — appse ai Partner Accelerator
---

# sow-generator — Turn a Requirement Digest into a Formal SOW

Converts a completed Requirement Digest into the document a partner can put
in front of a customer. The Requirement Digest's confidence-tagging is not
discarded here — it's the mechanism this skill uses to decide what becomes a
firm scope statement versus what becomes an explicit Assumption (Step 5).
That link is what keeps the SOW honest about what the discovery call actually
established versus what's being carried forward as a reasonable guess.

**The failure mode this skill exists to prevent:** a SOW that reads as
confidently as if every line were confirmed, when in reality half of it was
inferred from a single vague sentence in a discovery call. A customer reading
the SOW has no way to tell the difference unless the document tells them.

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
> **Assumptions:** _(carried forward from the Requirement Digest as [Inferred], not confirmed)_
>
> - A Business Partner record is created automatically for new customers, since this wasn't explicitly confirmed on the discovery call
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
| `requirement_digest` | ✅       | User provides                                              | The completed digest from `requirement-digest` (shown to partners as the "Discovery Summary"; if missing, offer to create one — see Step 0) — readiness signal, five confidence-tagged fields, vocabulary map, vertical signals, open questions | —                                                 |
| `sow_format`         | ⬜       | User provides, if asked in Step 0                          | The partner's own SOW template or house format                                                                                                     | none — falls back to the generic format in Step 1 |
| `org_id`             | ⬜       | User confirms, only if capability cross-checking is wanted | Which appse ai org's catalog to cite real capabilities from in Step 3                                                                              | skip cross-check if not given                     |

---

## Workflow

Follow these steps in order. Do not skip or reorder.

### Step 0 — Confirm a Discovery Summary Exists, Then Ask for the SOW Format

**First, check that a Requirement Digest was actually provided.** Partners
know it as the **Discovery Summary** (the output of `requirement-digest`) —
use that name, not "Requirement Digest", when talking to them. If none was
given (e.g. the partner asks "write a SOW for Shopify to SAP" with only a
one-line description, or pastes raw call notes), do not invent one and do
not draft from the raw material directly. Say that a SOW is built from a
Discovery Summary, and offer to create one first from their call notes or
transcript (via `requirement-digest`). Only continue with this skill once a
summary exists.

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
3. Objectives
4. Scope of Services / Deliverables
5. Assumptions
6. Exclusions
7. Indicative Timeline
8. Roles and Responsibilities — partner vs. customer vs. appse ai, reflecting
   the real support split (partner owns workflow logic, mappings, and
   business rules; appse ai owns core platform, connectors, and
   infrastructure)
9. Acceptance Criteria — how the customer confirms deliverables are met
10. Commercial Terms — a reference/placeholder line only ("commercial terms
    per the accompanying quote/contract"); this skill never drafts pricing

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

### Step 5 — Draft Deliverables, Tagged by Confidence Source

Each deliverable traces back to a **[Stated]** fact in the digest. Anything
that would only be supportable by an **[Inferred]** fact belongs in
Assumptions (Step 6), not stated here as a firm deliverable — do not let an
inferred detail slip into the confident, committed section of the document.

### Step 6 — Draft Assumptions Directly from the Digest's [Inferred] Tags

This is the mechanical core of the link between the two skills: **every
fact the Requirement Digest tagged [Inferred] becomes an explicit
Assumption line here**, phrased so the customer can see and challenge it
("Assumes X, based on Y — please confirm"). Do not silently fold an
inferred fact into a deliverable as if it were confirmed.

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

### Step 9 — Assemble Into the Chosen Format

Map Steps 4–8's content into either the partner's own format (Step 0) or
the generic fallback (Step 1). Named account, objectives, deliverables,
assumptions, exclusions, and indicative timeline must all be present and
clearly labeled regardless of which format is used — these six are never
optional, even inside a partner's custom template.

### Step 10 — Present and Flag

Present the assembled SOW. If Step 2 flagged a thin digest, repeat that
flag here rather than only mentioning it once and letting it get lost. If
any deliverable in Step 3 couldn't be grounded in a confirmed capability,
say so plainly rather than presenting it with the same confidence as one
that was.

---

## Allowed Tools

### arise-mcp — read-only

list_apps → used only in Step 3, to cite real capabilities in Deliverables
list_operations → used only in Step 3, same purpose

No write tool is used by this skill. Never call `create_workflow`,
`save_workflow`, or anything that modifies pricing, contracts, or the
partner's org.

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

- Never draft a SOW without a Discovery Summary (Requirement Digest) — if
  none is given, offer to create one first; never fabricate one from a
  one-line request
- Refer to the input as the "Discovery Summary" when talking to the
  partner, not "Requirement Digest"
- Always ask for the partner's own SOW format before drafting anything —
  every time, not just once per partner
- Fall back to the generic format only when no partner format is given
- Never draft pricing or commercial terms — reference them as a
  placeholder only
- Never let an [Inferred] fact from the digest appear as a firm deliverable
  — it becomes an Assumption instead
- Never silently proceed on a digest whose readiness signal is "Needs a
  follow-up call" — flag it and get explicit confirmation to proceed anyway
- Never fabricate a named appse ai capability or connector to make
  Deliverables sound more specific than what's actually confirmed
- Named account, objectives, deliverables, assumptions, exclusions, and
  indicative timeline are always present, regardless of format
- Indicative Timeline is always labeled as indicative, never presented as a
  committed date or resourced estimate
- On being asked to stop, stop immediately and report exactly what has and
  hasn't been drafted so far

---
name: requirement-digest
description: >
  Turns messy discovery-call material — transcripts, notes, emails — into a
  structured, confidence-tagged scoping digest (shown to partners as the
  "Discovery Summary") via arise-mcp app-catalog cross-checks. Use for
  "summarize this call," "digest these notes," "write a discovery summary,"
  "turn this into requirements," or — before a discovery call has happened —
  for a Call Prep Checklist ("what should I ask on my call?", "help me prepare
  for a discovery call"). Always leads with a readiness signal, always tags every
  fact as stated or inferred, and always merges into an existing digest for
  the same deal rather than duplicating it. Always delivers the Discovery
  Summary as a Word (.docx) file, with the customer follow-up questions on
  their own page as a ready-to-send email. Not for writing the SOW,
  estimate, or proposal — those are separate downstream skills that consume
  this one's output.
license: Internal — appse ai Partner Accelerator
---

# requirement-digest — Turn Discovery Material into a Scoping Digest

Converts unstructured discovery material into the structure the rest of the
Phase 1 pipeline (SOW Generator, Effort and Team Estimator, Cross-App
Scenario Advisor) depends on. Discovery material and, optionally, an
existing digest for the same deal are given by the user, not assumed.

**The real failure mode this skill exists to prevent:** a partner's post-call
notes flatten "the customer explicitly confirmed X" and "I'm pretty sure
they meant X" into the same sentence. That distinction is often the
difference between a SOW that holds up and one that doesn't — and by the
time anyone notices, the discovery call is long over. Confidence-tagging
every fact (Step 3) exists specifically to stop that flattening from
happening.

---

## Partner-Facing Language

"Requirement Digest" is this skill's internal name, not an industry term — a
partner seeing it cold won't know what it is or that the SOW step depends
on it. In **everything the partner sees**, use plain language instead:

| Internal term (skill/pipeline only)       | Say this to the partner                     |
| ----------------------------------------- | ------------------------------------------- |
| Requirement Digest / digest               | Discovery Summary                           |
| `summary` mode (digest path, Steps 1–9)   | Discovery Summary — "after the call"        |
| `call-prep` mode (Step 10)                | Call Prep Checklist — "before the call"     |
| Readiness signal                          | Ready to scope?                             |
| Vocabulary map                            | Customer's terms                            |
| Vertical signals                          | Industry signals                            |

Never say "digest", "pre-call", or "mode" to the partner. When telling them
which path you're taking, use plain phrasing, e.g.:

- _"Sounds like the call hasn't happened yet, so I'll put together a **Call
  Prep Checklist**: what to prepare and what to ask."_
- _"I'll turn these call notes into a **Discovery Summary**: what the
  customer needs, what's confirmed vs. assumed, and what to follow up on."_

The skill name (`requirement-digest`) and the field order stay unchanged,
since downstream skills (e.g. `sow-generator`) parse this output. Only the
labels shown to the partner change.

---

## Tool Reference (arise-mcp, read-only)

This skill's only platform tool is `list_apps`, used once, in Step 4, to
cross-check each app named in the discovery material against appse ai's
real catalog. This is a **first-pass credibility check, not a requirement**:
if `org_id` isn't given, or the lookup isn't available, skip it and mark
Apps Involved as **not cross-checked against the catalog** rather than
stalling or guessing a `code`.

No other arise-mcp tool is used. This skill never creates, saves, or
modifies a workflow — that is a separate skill (`workflow-creator`)
downstream of this one's output.

The Word file in Step 9A is built with the `docx` skill and saved locally
only — never uploaded or sent anywhere.

---

## Reference Example (illustrative — shape only, not content to copy)

A compact worked digest, to check output shape against — not a template
whose specific facts should appear in an unrelated deal:

> **Discovery Summary — [account name]**
>
> _[Stated] = the customer said it · [Inferred] = our assumption, confirm
> before committing · [Missing] = not discussed_
>
> **Ready to scope? Scope with caveats**
>
> 1. **Apps Involved:** Shopify [Stated, source] → SAP Business One [Stated,
>    > destination, catalog code `sapbusinessone`]
> 2. **Candidate Triggers/Actions:** When a new Shopify order is placed,
>    create a matching SAP B1 sales order [Stated]
> 3. **Data Entities:** Orders, line items [Stated]; customer/business
>    partner record [Inferred — implied by needing to invoice, not
>    > explicitly discussed]
> 4. **Volumes:** ~150 orders/day [Stated]
> 5. **Constraints:** Must go live before their Q4 peak season [Stated]; no
>    stated compliance constraints [Missing — not discussed]
> 6. **Customer's terms:** "our web store" → Shopify; "the SAP guy's system" →
>    SAP Business One
> 7. **Industry signals:** eCommerce/retail terminology (SKU, fulfillment)
> 8. **Open Questions:** "Do you need a business-partner record created
>    automatically for new customers, or do those already exist in SAP?"
>
> **Next:** say "draft a SOW from this" to turn this summary into a Scope of
> Work.

---

## Inputs

| Field                | Required | Source                                                                               | Description                                                                                              | Default                           |
| -------------------- | -------- | ------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------- | --------------------------------- |
| `discovery_material` | ✅       | User provides                                                                        | Raw call transcript, notes, emails, or chat export from a sales discovery conversation                   | —                                 |
| `mode`               | ⬜       | User states, or inferred from phrasing ("what should I ask" vs. "here are my notes") | `summary` (after the call: turn material into a Discovery Summary) or `call-prep` (before the call: produce a Call Prep Checklist) | `summary`                         |
| `existing_digest`    | ⬜       | User provides, if this is a second-or-later call for the same deal                   | A previously-generated digest for this deal, to merge into rather than duplicate                         | none — treat as the first call    |
| `org_id`             | ⬜       | User confirms, only if app cross-checking is wanted                                  | Which appse ai org's app catalog to check named apps against via `list_apps`                             | skip the cross-check if not given |

---

## Workflow

Follow these steps in order. Do not skip or reorder.

### Step 0 — Before or After the Call?

If `mode` is `call-prep` (or the user is asking what to prepare/ask before a
call rather than pasting material from one that already happened), skip
directly to **Step 10** (Call Prep Checklist). Otherwise, proceed through
Steps 1–9A to produce the Discovery Summary and its Word file. Tell the partner which one
you're doing in plain words (see Partner-Facing Language).

### Step 1 — Assess Input Sufficiency

Before extracting anything: is `discovery_material` substantial enough that
producing the five-field structure (Step 3) would be genuine extraction,
not fabrication? This is the **only** point in this skill where a blocking
question is appropriate.

- If the material is a couple of sentences with no app names and no stated
  outcome, stop and ask: _"This looks like partial notes — is there a
  fuller transcript, or is this everything from the call?"_
- Otherwise, proceed. Every other gap from here on (an unclear
  source/destination, a vague volume, an unstated trigger) is handled by
  confidence-tagging and Open Questions (Steps 3, 7) — not a blocking
  question. This skill's job is to make gaps visible, not to gatekeep on
  them.

### Step 2 — Check for an Existing Digest to Merge Into

If `existing_digest` is provided, this is not a fresh deal — **merge into
it rather than producing a second, separate digest.** Carry forward fields
that are unchanged. Where the new material adds or contradicts something,
say so explicitly (e.g. "customer's stated volume changed from ~200/day to
~50/day between calls") rather than silently overwriting the earlier value.

### Step 3 — Extract and Confidence-Tag the Five Fields

Always in this order. Every fact is tagged **[Stated]** (the customer said
this directly), **[Inferred]** (reasonable given context, not confirmed), or
**[Missing]** (the material doesn't cover it at all) — never blended without
marking which.

1. **Apps Involved** — every system mentioned, with its likely role (source,
   destination, or both), in the customer's own words. Catalog cross-check
   happens in Step 4, not here.
2. **Candidate Triggers/Actions** — phrase as "When [X] happens, [Y] should
   occur" wherever supported. A missing trigger is tagged **[Missing]**, not
   invented to fill the slot.
3. **Data Entities** — the business objects involved (orders, contacts,
   invoices, inventory items, deals, tickets). Capture any customer-specific
   field or ID scheme called out by name.
4. **Volumes** — transaction volume, frequency, or scale. Anything not
   explicitly stated is **[Inferred]** — don't round up confidence because a
   number would be convenient to have; inferred volumes are one of the most
   common sources of a bad effort estimate downstream.
5. **Constraints** — compliance (GDPR, data residency, SOC 2), timing,
   budget signals, technical limits (on-prem-only, no API access), anything
   the customer said "we can't" or "we won't" about. **Flag explicitly if
   the material suggests regulated data is in scope** (health, financial,
   payment card data) — this needs to reach the SOW writer, not get lost in
   a generic constraints bullet.

### Step 4 — Cross-Check Apps Involved Against the Catalog

If `org_id` was given, call `list_apps` for each app named in Step 3's field
1 and cite the confirmed `code` alongside the customer's own name for it. If
a named app doesn't match anything in the catalog, say so rather than
guessing the closest fit. If `org_id` wasn't given, mark this field **not
cross-checked against the catalog** and proceed — this is a credibility
enhancement, not a blocking requirement.

### Step 5 — Build the Vocabulary Map

A short side-by-side: _customer's term → appse ai standard term_, wherever
they differ (e.g. "the SAP guy's system" → SAP Business One). Preserve
both — the customer's own words matter for the proposal's tone later, the
standard term matters for scoping accuracy now.

### Step 6 — Note Vertical Signals

Flag any vertical-specific terminology encountered (manufacturing: goods
receipt, WIP, BOM; retail/eCommerce: SKU, fulfillment, returns window;
distribution: warehouse, pick-pack), and note if the digest reflects a
specific vertical's process model. This tells the downstream Cross-App
Scenario Advisor which battle-tested pattern to reach for.

**Not this skill's job:** suggesting complementary apps the customer didn't
mention (e.g. "they named an eCommerce app and an ERP — ask if they also
want a CRM connected") is `cross-app-scenario-advisor`'s job, not this
skill's. It needs its own live `list_apps` category lookup and a "supported
as of {date}" disclaimer to avoid suggesting something stale — improvising
it here, without that check, risks recommending an app pairing that isn't
actually buildable. If the apps named span more than one category (e.g. an
eCommerce app and an ERP), add a single line pointing there instead of
guessing a suggestion: _"Worth a quick capability check on what else
typically pairs with these before your next call?"_ — never a firm
recommendation of a specific app.

### Step 7 — Draft Open Questions as Ready-to-Send Follow-Ups

**These are for the partner to send to the customer — not questions the
partner answers here.** Say so plainly at the top of this section every
time (see Step 9's presentation order); don't leave it implied.

Don't list gaps as bare statements — draft them as text the partner could
paste directly into a follow-up email. "We didn't get expected order
volume" is a gap; "Roughly how many orders/day do you process today, and do
you expect that to grow significantly in the next 12 months?" is something a
partner can actually send.

**Rank and cap, the same way `sow-generator` and `workflow-creator` do —
don't just list every gap:**
- **Rank by what actually changes the scope or the readiness signal**
  (e.g. which system owns the master record, how a stated-but-ambiguous
  detail like a discount or a product match should work, a hard go-live
  date) above what's merely nice to know (e.g. which countries customers
  are in, when nothing else suggests a compliance angle).
- **Cap at five, ranked** — if there are more real gaps than that, keep the
  five most scope-relevant as Open Questions and fold the rest into
  **[Missing]** tags on the relevant field instead of a standalone
  question. Never drop a gap that changes the readiness signal just to
  hit the number — downgrade lower-impact ones first.
- **State the reply format once, plainly** — since these go to the
  customer, not the partner, say so: _"Numbered so you can paste these
  into one email, or split them up — however you'd normally follow up."_
  Don't imply the partner must answer them in this conversation.

### Step 8 — Determine the Readiness Signal

Based on how much of Step 3's five fields came back Stated versus Inferred
versus Missing, assign one of:

- **Ready to scope** — core fields are Stated, not Inferred; a SOW can be
  drafted with confidence
- **Scope with caveats** — enough to proceed, but real assumptions are
  baked in; name which fields are driving this rating
- **Needs a follow-up call** — too much of the structure would be
  fabrication; say plainly what's missing rather than recommending this as
  a formality

### Step 9 — Present the Digest

Title it **Discovery Summary — [account name]** and use the partner-facing
labels from the Partner-Facing Language section throughout. In order:

1. The tag legend, verbatim, directly under the title: _"[Stated] = the
   customer said it · [Inferred] = our assumption, confirm before
   committing · [Missing] = not discussed"_
2. **Ready to scope?** — the Step 8 readiness signal
3. The five confidence-tagged fields (Step 3, cross-checked per Step 4)
4. **Customer's terms** (Step 5), **Industry signals** (Step 6), and Open
   Questions (Step 7) — introduce this last section with the "these are
   for you to send to the customer" line and the reply-format note from
   Step 7, every time; never leave it implied
5. A closing next-step line: _"Next: say 'draft a SOW from this' to turn
   this summary into a Scope of Work."_ If the readiness signal is **Needs
   a follow-up call**, make the next step sending the Open Questions
   instead, and say the SOW should wait. If the apps named span more than
   one category (Step 6), add the one-line capability-check pointer there
   too — never more than once, never as a firm app recommendation.

If this was a merge (Step 2), show what changed from the prior digest, not
just the final state. Tight, scannable sections — this is read quickly by a
human and parsed by the next skill in the pipeline, not read as a
narrative. Length scales with the input, not a fixed template length.

### Step 9A — Deliver the Discovery Summary as a Word File

**The summary path always ends with a Word file.** The chat version (Step
9) is what the partner reads first; the file is what they keep and share
with their deal team. Build it straight after presenting — don't ask which
format. If the partner asks for a PDF instead or as well, build it with
the `pdf` skill from the same content.

- **Same content as the chat version** — same sections, order, wording,
  and tags. Don't add, drop, or reword while converting. If the partner
  asks for changes, update the chat version first, then regenerate the
  file as a new version.
- **Layout:** title **Discovery Summary — {account name}**, the date, the
  tag legend, then each Step 9 section as a heading. Clean, neutral
  professional layout, no appse ai branding. Wherever the platform is
  named, write "appse ai" — lowercase, with a space.
- **Customer follow-up page:** put the Open Questions (Step 7) on their own
  final page, written as a ready-to-send email — one greeting line
  addressed to the customer contacts named in the notes (if any), the
  numbered questions, and a short sign-off with a `[Your name]`
  placeholder. **No [Stated]/[Inferred]/[Missing] tags, catalog codes, or
  internal notes on this page** — it's the one part meant to leave the
  partner's team. If there are no Open Questions, leave the page out.
- **Carry the readiness flag:** if the signal is **Needs a follow-up
  call**, say so in one line at the top of the first page.
- **File name and location:** `Discovery Summary - {Account Name} -
  {YYYY-MM-DD}.docx`, saved in the current working directory unless the
  partner names another folder. Never overwrite an existing file — add a
  version suffix (`v2`, `v3`). A merged summary from a later call (Step 2)
  is always a new version, never an overwrite.
- **Report back** with the file path in one line.

The summary path isn't finished until the file exists and its path has
been reported.

### Step 10 — Build the Call Prep Checklist (before the call only)

Entry point from Step 0 when no call has happened yet. Title the output
**Call Prep Checklist — [account name]** and give a short, targeted list of
what to prepare or ask, based on whatever is already known about the deal
(industry, systems mentioned so far, if any). A structured intake going
_in_ produces cleaner material coming out in Step 1 next time — this is the
one place this skill acts before the call rather than cleaning up after it.
Do not proceed through Steps 1–9A on this path — the Call Prep Checklist
stays in chat, with no file. End with: _"After the call,
paste your notes or transcript here and I'll turn them into a Discovery
Summary."_

---

## Allowed Tools

### arise-mcp — read-only

list_apps → used only in Step 4, to cross-check named apps against the catalog; never used to modify anything

No other arise-mcp tool is used by this skill. Never call `create_workflow`,
`save_workflow`, or anything else that writes — that is entirely out of
scope here, regardless of what the discovery material describes wanting
built.

### Local file output — Step 9A only

`docx` skill → the Word version of the Discovery Summary, every summary run
`pdf` skill → only if the partner asks for a PDF instead or as well

Writes go to the working directory (or a folder the partner names) — never
overwrite an existing file.

---

## Known Limits (update as testing reveals more)

- Vertical-signal detection (Step 6) is pattern-matching against common
  terminology, not validated against a fixed appse ai vertical taxonomy
- Multi-call merge behavior (Step 2) has not been tested past two calls for
  the same deal
- Only Apps Involved gets a catalog cross-check (Step 4) — Data Entities
  and Candidate Triggers/Actions are not validated against real appse ai
  operation names; that level of verification belongs to `workflow-creator`,
  not this skill
- The readiness signal (Step 8) is a judgment call based on the
  Stated/Inferred/Missing mix, not a scored or quantified threshold
- Not yet tested: the Call Prep Checklist (Step 10) end-to-end, a
  three-or-more-call merge chain, discovery material in a language other
  than English
- **Open Questions rank/cap and the `cross-app-scenario-advisor` pointer
  (added 2026-09-25)** — never exercised live. Watch whether a genuinely
  scope-changing gap ever gets folded into [Missing] just to stay under
  five; that would be the rule working against its own intent
- **Word file output (Step 9A, added 2026-09-26)** — never exercised live.
  Watch that the customer follow-up page never picks up tags, catalog
  codes, or internal notes, and that a merged summary saves as `v2`

---

## Output Rules

- Always call the output a **Discovery Summary** (after the call) or a
  **Call Prep Checklist** (before the call) to the partner, and use the
  plain labels ("Ready to scope?", "Customer's terms", "Industry signals") —
  never "digest", "pre-call", "mode", or the other internal terms
- Always include the tag legend under the title, and always end with the
  next-step line pointing to the SOW (or to the Open Questions, if a
  follow-up call is needed)
- Always lead with the Step 8 readiness signal before the detailed fields
- Always tag every fact **[Stated]**, **[Inferred]**, or **[Missing]** —
  never blend without marking which
- Never invent a missing trigger, volume, or constraint to fill a field —
  tag it and move on
- Never upgrade an Inferred fact to Stated confidence because Stated would
  be more convenient for scoping
- Always draft Open Questions as ready-to-send follow-up text, not bare gap
  statements — and always state plainly that they're for the customer, not
  the partner, with a one-line reply-format note, every time
- Cap Open Questions at five, ranked by what actually changes the scope or
  readiness signal — fold lower-impact gaps into **[Missing]** tags instead
  of padding the list; never drop a scope-changing gap just to hit the cap
- Suggesting a complementary app the customer didn't name is
  `cross-app-scenario-advisor`'s job, not this skill's — at most, one
  pointer line when named apps span more than one category; never a firm
  app recommendation of its own
- Always merge into an existing digest for the same deal (Step 2) rather
  than producing a duplicate — and always show what changed, never silently
  overwrite
- Only stop and ask the partner when input is too sparse to extract
  anything real (Step 1) — every other gap is flagged and continued, not
  blocked
- Cross-check Apps Involved against `list_apps` when `org_id` is given;
  mark **not cross-checked** rather than guessing when it isn't
- Never write the SOW, effort estimate, or proposal — output stops at the
  digest
- The summary path always ends with a Word file (Step 9A), built from the
  chat version without asking which format; the customer follow-up page
  carries no tags, codes, or internal notes. Never overwrite a file — new
  versions get `v2`, `v3`. The Call Prep Checklist stays in chat
- Never editorialize on deal quality, customer readiness, or likelihood to
  close
- On being asked to stop, stop immediately and report exactly what has and
  hasn't been produced so far

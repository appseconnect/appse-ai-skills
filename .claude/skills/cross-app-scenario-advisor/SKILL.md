---
name: cross-app-scenario-advisor
description: >
  Gives a partner a capability brief on named apps — real integration
  scenarios, what triggers/actions each app actually supports in appse ai,
  and known quirks or limits — via arise-mcp app-catalog cross-checks. Use
  when a partner asks "what can I do with {app A} and {app B}", "what does
  {app} support", "what's possible between these apps", or wants to
  understand a system outside their core expertise (e.g. a SAP partner
  scoping a deal that also touches Shopify or HubSpot) before or during
  scoping. Never tied to a specific deal — works from app names alone, no
  Discovery Summary, SOW, or call notes needed first. Always delivers the
  brief as a customer-shareable PDF one-pager. Not for extracting a
  specific customer's requirements (that's `requirement-digest`), not for
  drafting a SOW, and not for building a workflow — those are separate
  skills this one can hand off to.
license: Internal — appse ai Partner Accelerator
---

# cross-app-scenario-advisor — Close the Knowledge-Breadth Gap on Named Apps

A partner who is deep in one system (e.g. SAP) is often shallow on the
others a customer's stack includes (e.g. Shopify, HubSpot). That's a
knowledge gap, not a throughput gap — more time or headcount doesn't fix
it. This skill closes it directly: given one or more named apps, it
produces a **Capability Brief** grounded in appse ai's real, current
catalog — not general knowledge about what those products "usually" do.

**The failure mode this skill exists to prevent:** a partner declining or
under-scoping a deal because they don't know an app is supported, or
over-promising a scenario that sounds plausible but isn't actually
buildable in appse ai today. Every claim in the brief traces back to a real
catalog entry — never a guess dressed up as a fact.

**Always cold-start capable.** A partner can ask this before any discovery
call, before any digest, with nothing but app names. It never asks for a
Discovery Summary or SOW first, and never requires a specific deal or
customer name.

---

## Tool Reference (arise-mcp, read-only)

`list_apps` — confirm each named app exists in the org's catalog, and its
real `code`(s); some apps have more than one catalog entry (e.g. an
on-premise and a cloud variant) with different supported operations.

`list_operations` — per confirmed app, enumerate real triggers and actions,
grouped by the business entity they act on (orders, customers, tickets,
deals, items…). This is what the brief's scenarios are built from — never
invented from what the app "usually" supports elsewhere.

Optional: `query-docs` (Context7, `/appseconnect/appse-ai-docs`) for a named
app's known quirks (auth model, rate limits, required fields, dual-variant
differences) — skip if it doesn't resolve; never fabricate a quirk to sound
more specific.

No write tool is used. This skill never creates, saves, or modifies
anything on the platform, and never commits to a scope, timeline, or price
for any deal. The only thing it writes is the local PDF in Step 6, built
with the `pdf` skill — never uploaded or sent anywhere.

---

## Reference Example (illustrative — shape only)

> **Capability Brief — Shopify + HubSpot** _(supported as of 2026-09-25 —
> appse ai's catalog changes; re-run this before relying on it for a live
> deal)_
>
> **Confirmed in the catalog:** Shopify (`shopify`) · HubSpot (`hubspot`)
>
> **Common scenarios between these two, grounded in real operations:**
> 1. New Shopify customer → create/update HubSpot contact
> 2. New Shopify order → create HubSpot deal, associated to the contact
> 3. HubSpot deal stage change → tag or update the Shopify customer (e.g.
>    a loyalty segment)
>
> **What each app supports here (from `list_operations`):**
> - Shopify: triggers on new/updated customer, new/updated order; actions
>   to create/update customer, tag customer
> - HubSpot: triggers on deal stage change; actions to create/update
>   contact, create deal, associate contact to deal
>
> **Quirks to know before scoping:**
> - HubSpot deal-to-contact association is a separate action from deal
>   creation — plan it as its own step, not assumed automatic
> - [Any further quirks not confirmed by docs: state as unconfirmed, don't
>   guess]
>
> **Not confirmed / not available:** [any requested scenario with no
> matching operation — state plainly, never imply it's possible]
>
> **Next:** "Ready to scope a specific deal using these? Say 'digest this
> call' or 'draft a SOW' once you have customer specifics."

---

## Inputs

| Field      | Required | Source                                           | Description                                                                 | Default                              |
| ---------- | -------- | ------------------------------------------------ | ----------------------------------------------------------------------------- | ------------------------------------- |
| `apps`     | ✅       | User states                                       | One or more named apps to brief on (e.g. "Shopify", "HubSpot", "SAP B1")       | —                                      |
| `org_id`   | ⬜       | User confirms — re-resolve every run, never reused from an earlier run or conversation | Which appse ai org's catalog to check apps and operations against             | ask once if catalog-grounding is wanted; skip only if the partner explicitly wants general orientation without a catalog check |
| `focus`    | ⬜       | User states                                       | A narrower question if asked (e.g. "just orders", "what triggers does X have") | brief covers all common entities for the named apps |

---

## Workflow

Follow these steps in order. Do not skip or reorder.

### Step 0 — Confirm the Apps and the Org

If no app is named, ask which ones before doing anything else — this
skill has nothing to work from otherwise. If `org_id` isn't given, ask once
whether the partner wants the brief grounded in a real org's catalog
(recommended) or a general, uncatalogued orientation (clearly labeled as
such, and only if the partner says catalog access isn't available yet).
**Re-resolve `org_id` every run** — never reuse one from an earlier run or
an earlier point in this conversation, even for the same partner.

### Step 1 — Confirm Each App in the Catalog

Call `list_apps`. For each named app:
- If it matches one catalog entry, use its real `code`.
- If it matches more than one (e.g. an on-prem and a cloud variant), name
  both and ask which applies **only if it changes which operations are
  available** — otherwise note both and proceed.
- If it doesn't match anything, say so plainly: this app isn't a built-in
  connector today. Don't stop the whole brief — cover the apps that did
  match, and name the unmatched one as not currently supported (see
  `workflow-creator`'s generic-HTTP-node option if the partner wants to
  explore a custom connection instead — that's a build decision, not this
  skill's).

### Step 2 — Enumerate Real Operations Per App

Call `list_operations` for each confirmed app. Group the results by the
business entity they act on (orders, customers, contacts, deals, tickets,
items…), noting which are triggers and which are actions. This is the
factual base the whole brief is built from — never supplement it with
outside knowledge of what the app "usually" does.

### Step 3 — Synthesize Common Scenarios

From the operations found in Step 2, write the scenarios that are actually
buildable between the named apps — phrased as "When [X] in App A, do [Y] in
App B." **Every scenario must trace to a real trigger and a real action
from Step 2.** If a scenario a partner might expect (e.g. "sync inventory
levels") has no matching operation, don't include it as a scenario — list
it under "Not confirmed / not available" instead, stated plainly, never
hedged as if it might quietly work.

### Step 4 — Note Quirks and Limits (optional, docs-grounded)

Call `query-docs` for anything likely to matter when scoping (auth model,
rate limits, which of several catalog variants to use, fields commonly
missed). If docs don't resolve or don't cover it, skip silently — don't
guess a quirk to sound more thorough than what's confirmed.

### Step 5 — Present the Capability Brief

Title it **Capability Brief — {apps}**, dated with a **"supported as of
{today's date}"** line up front — the catalog changes over time, and a
brief used weeks later should be re-run, not assumed still accurate.
Structure: confirmed apps, common scenarios (grounded), what each app
supports (grouped operations), quirks, and anything not confirmed/not
available. Plain business language — no internal tool names.

End with a plain next-step line pointing to the deal-specific skills (not
naming them internally): "Ready to scope a specific deal using these?" →
`requirement-digest` if there's a call to digest, `workflow-creator` if the
partner wants to build straight from a plain scenario.

### Step 6 — Deliver the Brief as a PDF One-Pager

**The brief always ends with a PDF the partner can share with a
customer.** Build it straight after presenting — don't ask which format.
If the partner asks for Word instead or as well, build it with the `docx`
skill from the same content.

- **Same substance as the chat brief, written for a customer reader:**
  - **Keep:** the "supported as of {date}" line (prominent, directly under
    the title), the confirmed apps by product name, the common scenarios,
    what each app supports (in business terms), and considerations worth
    planning for.
  - **Drop:** internal catalog codes (e.g. `shopify`,
    `dynamics365businesscentral`), internal tool names, the partner
    next-step line, and anything the chat brief marked unconfirmed — the
    PDF states only what's confirmed.
  - **Not currently available:** keep it, worded neutrally ("Not currently
    available as a built-in connection: {app}"). Never imply it's coming —
    no roadmap promises.
- **Length:** one page where possible; two at most for three or more apps.
- **Layout:** title **Capability Brief — {App A} + {App B}**, the date
  line, then short sections with headings. Clean, neutral layout, no appse
  ai logo or branding assets. Wherever the platform is named, write "appse
  ai" — lowercase, with a space. Never alter product names or partner
  marks (SAP, Microsoft, Shopify, HubSpot…).
- **Skip the file** if no named app was confirmed in the catalog, or if
  this was the uncatalogued orientation from Step 0 — there's nothing
  confirmed to share. Say so in one line instead.
- **File name and location:** `Capability Brief - {App A} + {App B} -
  {YYYY-MM-DD}.pdf`, saved in the current working directory unless the
  partner names another folder. Never overwrite an existing file — add a
  version suffix (`v2`, `v3`).
- **Report back** with the file path in one line.

The run isn't finished until the file exists and its path has been
reported (or the one-line reason it was skipped).

---

## Allowed Tools

### arise-mcp — read-only

list_apps → Step 1, to confirm each named app and its real code(s)
list_operations → Step 2, to enumerate real triggers/actions per app

### Context7 — read-only

query-docs (`/appseconnect/appse-ai-docs`) → Step 4 only, for quirks/limits; skip if it doesn't resolve

### Local file output — Step 6 only

`pdf` skill → the PDF one-pager, every run with at least one confirmed app
`docx` skill → only if the partner asks for Word instead or as well

Writes go to the working directory (or a folder the partner names) — never
overwrite an existing file.

No platform write tool is used by this skill. Never call `create_workflow`,
`save_workflow`, or anything that modifies pricing, contracts, or the
partner's org.

---

## Known Limits (update as testing reveals more)

- New skill, not yet exercised live — treat the first real run as a test,
  not a routine brief
- Scenario synthesis (Step 3) depends entirely on `list_operations`
  returning complete, well-named results; an app with sparse or oddly
  named operations may produce a thinner brief than the app actually
  supports
- Does not confirm a scenario is fully buildable end to end (field-level
  requirements, nested shapes, node conventions) — that check belongs to
  `workflow-creator` once a specific workflow is being built
- Not yet tested: three or more apps in one brief, an app with several
  catalog variants where the choice actually changes available operations
- PDF output (Step 6) not yet exercised — watch that catalog codes and
  unconfirmed items never reach the file, and that it stays to one page
  for two apps

---

## Output Rules

- **Never tied to a specific deal.** No customer name, no SOW content, no
  effort estimate — this is orientation, not scoping
- **Every scenario traces to a real operation** from `list_operations` —
  never invented from general knowledge of what an app "usually" supports
  elsewhere
- **Always date the brief** ("supported as of {date}") — appse ai's
  catalog changes, and a stale brief is worse than none
- **Say plainly what isn't confirmed or isn't available** — never hedge an
  unsupported scenario as if it might quietly work
- **Re-resolve `org_id` every run** — never reuse one from an earlier run
  or conversation
- **Cold-start always works** — never require a Discovery Summary, SOW, or
  prior digest before producing a brief
- **Never write, save, or modify anything on the platform** — read-only,
  every run; the only thing written is the local PDF (Step 6)
- **Always end with a customer-shareable PDF one-pager** (Step 6), without
  asking which format — dated, no catalog codes, only confirmed content,
  never overwriting an existing file. Skip it only when nothing was
  confirmed
- End with a plain next-step pointer to `requirement-digest` or
  `workflow-creator`, without naming internal tools
- On being asked to stop, stop immediately and report exactly what has and
  hasn't been produced so far

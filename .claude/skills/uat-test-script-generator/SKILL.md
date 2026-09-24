---
name: uat-test-script-generator
description: >
  Turns a completed appse ai workflow definition into a structured UAT test
  checklist — trigger conditions, expected actions, and edge cases — via
  arise-mcp. Use when a partner has a built workflow ready for testing and
  needs a formal sign-off document, or asks to "generate a test script,"
  "write UAT cases for this workflow," or "what should we test before
  go-live." Always reads the actual workflow structure rather than inferring
  test cases from a description, and always surfaces any placeholder or
  unconfirmed configuration left in the workflow as a blocking pre-go-live
  item, never as a test case that would silently pass. Not for building or
  fixing the workflow itself — that's `workflow-creator`'s job; this skill
  only reads what already exists.
license: Internal — appse ai Partner Accelerator
---

# uat-test-script-generator — Turn a Completed Workflow into a Sign-Off Checklist

|                   |                                                                                           |
| ----------------- | ----------------------------------------------------------------------------------------- |
| **Input**         | A completed workflow definition                                                           |
| **Output**        | A structured test checklist covering trigger conditions, expected actions, and edge cases |
| **Partner value** | Gives partner and customer a shared, consistent basis to sign off before go-live          |

The test cases in this checklist are derived from the workflow's **actual node
structure**, not a general description of what it's supposed to do. A
Decision or Filter node in the real definition becomes a test case for each
of its branches; a node whose name still reads like a placeholder becomes a
blocking item, not a passing test. This skill is only as good as the
workflow it's reading — it does not invent coverage for logic that isn't
actually there.

---

## Tool Reference (arise-mcp, read-only)

- `get_workflow` — required. Reads the actual node/edge structure to derive
  test cases from, rather than trusting a paraphrased description of what the
  workflow does.
- `list_credentials` — used to check whether every app a node depends on is
  actually connected. A missing credential is a blocking precondition, not
  a test case that can be marked pass/fail.
- `get_execution_summary` / `list_executions` — optional. If the workflow has
  already run (even in a sandbox), real past failures should produce their
  own specific test case rather than being left to a generic "check for
  errors" line.

No write tool is used. This skill never modifies the workflow it's testing —
if a gap needs fixing, that's `workflow-creator`'s job, flagged back to the
user, not silently patched here.

---

## Reference Example (illustrative — shape only, not content to copy)

> **UAT Checklist — Shopify → SAP B1 Order Sync**
>
> | ID    | Test                              | Precondition                                              | Steps                                 | Expected Result                                                                         | Priority |
> | ----- | --------------------------------- | --------------------------------------------------------- | ------------------------------------- | --------------------------------------------------------------------------------------- | -------- |
> | TC-01 | New order triggers sync           | Shopify credential connected and active                   | Place a test order in Shopify         | Workflow run appears in execution history within the polling interval                   | Critical |
> | TC-02 | Duplicate order is not re-created | An order with the same reference already exists in SAP B1 | Re-submit the same order reference    | Decision node routes to the "exists" branch; no second SAP B1 order is created          | Critical |
> | TC-03 | New order creates a SAP B1 order  | No existing match                                         | Place a genuinely new test order      | A new Sales Order appears in SAP B1 with matching line items                            | Critical |
> | TC-04 | Missing optional field handled    | Order has no phone number                                 | Place an order without a phone number | Workflow completes without erroring; SAP B1 record has phone field blank, not malformed | Medium   |
>
> **Blocking items found in this workflow, not test cases:**
>
> - `Create Customer (needs gating)` — this node has no confirmed duplicate check; flagged by `workflow-creator`, not yet resolved
>
> **Sign-off:** Partner delivery lead: ****\_\_**** Customer stakeholder: ****\_\_**** Date: ****\_\_****

---

## Inputs

| Field                       | Required | Source                                                                  | Description                                                                                 | Default                                                            |
| --------------------------- | -------- | ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `workflow_id`               | ✅       | User provides, or resolved via `list_workflows` if only a name is given | The completed workflow to generate a checklist for                                          | —                                                                  |
| `org_id`                    | ✅       | User confirms                                                           | Which appse ai org the workflow lives in                                                    | none — always ask                                                  |
| `include_execution_history` | ⬜       | User states                                                             | Whether to cross-check against real past runs via `get_execution_summary`/`list_executions` | true, if the workflow has any execution history; skipped otherwise |
| `sign_off_names`            | ⬜       | User provides                                                           | Names/roles for the sign-off block                                                          | left blank for the partner to fill in                              |

---

## Workflow

Follow these steps in order. Do not skip or reorder.

### Step 0 — Resolve Organization and Read the Workflow

Call `list_organizations` if `org_id` isn't already confirmed. Call
`get_workflow(workflow_id)` to get the actual node/edge structure. **Do not
proceed from a description of the workflow alone** — if only a name or a
summary is given, resolve the real `workflow_id` via `list_workflows` first.

### Step 1 — Derive Trigger-Condition Test Cases

From the trigger node (`AppTriggerNode` or `ManualTriggerNode`), write at
least one test case confirming the trigger actually fires under real
conditions (e.g., "place a test order," "submit a test ticket"). If the
trigger is a `ManualTriggerNode` standing in for an unconfirmed real trigger
(a known pattern from `workflow-creator`), say so explicitly — this is not a
real trigger to test, and the checklist should say what it's a placeholder
for.

### Step 2 — Derive Expected-Action Test Cases

For every `AppNode`, write a test case confirming its action actually
produces the expected result in the target system (a record created, a
message sent, a field updated) — not just that the node executed without
erroring. Reference the node's real `action_name` and `properties` mapping,
not a paraphrase of what it's supposed to do.

### Step 3 — Derive Edge-Case Test Cases from Branch Points

For every `DecisionNode` or `FilterNode`, write **one test case per branch**
— both the case that should pass through and the case that should be
gated/stopped. A duplicate-check Decision node needs a test case for "record
already exists" and a separate one for "record does not exist" — never just
one test case covering "the check runs."

For every `SplitterNode`, write a test case for the multi-item case (more
than one line item/record) in addition to the single-item case — a workflow
that only maps `[0]` (a known limitation flagged by `workflow-creator`) will
pass a single-item test and silently fail a multi-item one; make that
distinction a real test case, not an assumption.

### Step 4 — Flag Placeholder or Unconfirmed Configuration as Blocking, Not Testable

Scan node `current_name` fields and `properties` values for the labeling
conventions `workflow-creator` itself uses to mark unresolved gaps —
"placeholder," "needs gating," "not yet confirmed," a template URL like
`https://<...>` or `example.com`, a generic recipient like
`team@company.com` never confirmed with the user. **List these under a
separate "Blocking items" section, not as pass/fail test cases.** A test
case implies something is ready to be checked; a placeholder isn't ready to
be checked, it's ready to be built.

### Step 5 — Confirm Required Connections Exist

Call `list_credentials` and check that every app referenced by a node in
this workflow has an active credential. Missing or unvalidated credentials
go in the same "Blocking items" section as Step 4, not as a test case.

### Step 6 — Cross-Check Against Real Execution History, If Available

If `include_execution_history` is true and the workflow has prior runs, call
`get_execution_summary`/`list_executions` and check for any real past
failure. A failure that already happened for real needs its own specific
test case confirming it's fixed — don't rely on the generic edge cases from
Steps 1–3 to implicitly cover something already known to have gone wrong.

### Step 7 — Assemble the Checklist

Use this structure for every test case, in a table: **ID, Test, Precondition,
Steps, Expected Result, Priority**. Priority is **Critical** for anything
gating data correctness (duplicate checks, required-field validation) or
financial/customer-facing actions; **Medium** for cosmetic or non-blocking
edge cases. Never mark something Critical just because it's first in the
list, and never bury a real duplicate-prevention test case at Medium
priority.

### Step 8 — Present with a Sign-Off Block

End every checklist with a sign-off section — partner delivery lead,
customer stakeholder, date — even if left blank for the user to fill in.
This is what makes the output a shared basis for go-live approval, not just
an internal QA list.

---

## Allowed Tools

### arise-mcp — read-only

list_organizations
list_workflows
get_workflow
list_credentials
get_execution_summary
list_executions

No write tool is used by this skill. Never call `create_workflow`,
`save_workflow`, or anything that modifies the workflow being tested — a gap
found here gets reported back for `workflow-creator` (or a human) to fix,
not patched in place.

---

## Known Limits (update as testing reveals more)

- Edge-case detection is structural — it's derived from which node types are
  present (Decision/Filter/Splitter), not a semantic understanding of the
  underlying business rule. A workflow with no branch nodes at all will
  produce a thinner edge-case list, which is accurate to what's actually
  there, not a failure of this skill to look harder.
- Cannot execute the workflow to verify a test case actually passes — this
  skill produces the checklist a human or a separate QA process runs
  manually against the real system.
- The placeholder-detection patterns in Step 4 are pattern-matching against
  `workflow-creator`'s own known labeling conventions — a gap introduced by
  a different tool or a human editing the workflow directly in the UI may
  not use the same patterns and could be missed.
- Not yet tested: a workflow with more than one Decision/Filter node in
  sequence (compounding branch combinations), a workflow with zero prior
  execution history to cross-check against.

---

## Output Rules

- Always read the real workflow via `get_workflow` — never generate test
  cases from a description or summary alone
- Always write at least one test case per branch of every Decision/Filter
  node — never one combined test case covering "the check runs"
- Always write a multi-item test case for every Splitter node, in addition
  to the single-item case
- Always separate blocking items (placeholders, missing credentials,
  unconfirmed configuration) from pass/fail test cases — a blocking item is
  not something to test, it's something to fix first
- Never assign Critical priority arbitrarily — reserve it for data
  correctness, duplicate prevention, and financial/customer-facing actions
- If the workflow has real execution history, always cross-check for a
  known past failure and give it its own test case
- Always end with a sign-off block for both partner and customer, even blank
- On any tool error, report the exact error text — do not paraphrase
- On being asked to stop, stop immediately and report exactly what has and
  hasn't been produced so far

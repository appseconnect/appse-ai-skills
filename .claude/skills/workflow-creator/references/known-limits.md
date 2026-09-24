# workflow-creator — Known Limits

*Moved verbatim from SKILL.md on 2026-09-24 so the core file stays short. Internal maintainer notes — not needed during a normal build. SKILL.md holds the condensed rules; this file holds the full detail and examples.*

## Known Limits (update as testing reveals more)

*(Internal reference — not partner-facing; see Tone for how these
limitations should instead be phrased if they ever surface in a partner
conversation.)*

- Validated end-to-end (build attempted and completed) so far: Shopify
  customer → SAP Business One (cloud) customer; SAP Business One (cloud)
  product → Shopify product (fully built and saved as "Workflow 9",
  00bc6c98-3dcb-4584-bbeb-6cc2c22829e3) — both using the simple-pattern
  reference. **No branching pattern (dedupe-skip, dedupe-create-or-update,
  SKU-reconciliation, parallel-branch) has been built end-to-end yet** —
  the reference files are new as of 2026-09-24 and Step 8's selection logic
  for them is untested in a live run.
- **Create-or-update is now backed by two independent real examples**
  (D365 BC → Magento2 customer, and Shopify → SAP B1 business partner) —
  different app pairs, same confirmed shape. Meaningfully stronger
  confidence than a single unverified example; the second also revealed the
  sub-record `RowNum`-preservation detail for updates touching nested array
  fields (see Reference Patterns).
- **No rename tool available.** Newly created workflows get a generic
  platform default name (e.g. "Workflow 9"), not the descriptive name the
  skill intends. Confirmed in the first successful build. Step 11 phrases
  this positively and without naming internal tooling; a real fix would need
  a rename capability added to the platform's tool surface.
- Field-reference expression syntax is documented and confirmed
  (`{{ $payload.field }}` / `{{ $('nodeName').payload.field }}`) — no longer
  a guess.
- The node/edge envelope structure for the simple pattern remains
  undocumented in appse-ai-docs; still dependent on the single live
  reference workflow. The branching patterns now have local, bundled
  reference files instead (see Reference Patterns) — resolves the portability
  concern for those shapes, since they no longer depend on any specific
  org's live data existing.
- appse-ai-docs field documentation can lag the live API (confirmed: SAP B1
  `CardType`) — always let the live `get_operation_detail` call govern
  required-ness.
- Context7 tool names confirmed live: `resolve-library-id`, `query-docs`.
- **`get_operation_detail` can return only shallow/top-level required-field
  info for object- or array-typed parameters**, without their internal
  shape (confirmed twice: SAP B1 `Create New Business Partner`, and
  Shopify's `create_product_options_and_media`). Distinct from documentation
  being unavailable — even with docs working, coverage for a given
  operation's nested shape isn't guaranteed either. Step 6 treats a
  shallow-only result as still unresolved.
- **`isValidated` credential flag — meaning unresolved, temporary override in
  effect (Step 2, added 2026-09-24).** Observed `false` even for a Shopify
  credential confirmed working via real executed data, across two different
  testers/sessions in the same org. Needs a definitive answer from the
  platform team; Step 2's override should be revisited once known.
- **Operation `dev` stage — unconfirmed as a real value.** Step 5 now blocks
  `dev`-stage operations, but no live call has ever returned this stage —
  only `release` and `preview` observed so far. Verify the real set of stage
  values with the platform team.
- SAP Business One has two distinct catalog entries — `sap_b1` (on-prem,
  DIS API) and `sapbusinessone` (cloud) — confirmed via live testing on two
  separate runs. When only one has a saved credential, Step 2 resolves this
  silently rather than asking (see Asking Questions).
- Multi-workflow decomposition (Step 0) has been exercised live — a partner
  explicitly asked to combine two distinct trigger events into one workflow,
  and the skill correctly refused with reasons and proposed the correct
  split instead. Still not yet confirmed: a full run all the way through
  Step 11 on a multi-workflow set, or the new "plan with alternatives"
  presentation format (Step 0b, added 2026-09-24) in a live run.
- **`SplitterNode`'s own configuration mechanism is unclear.** The one
  confirmed real example has empty `properties`, sits right after the
  trigger, and splits the order's line items — how it knows *which* list to
  split isn't in the saved config. Since 2026-09-24 the skill decides for
  itself whether a Splitter is needed (see "Decide the unit of processing")
  and, when it is, builds it like the reference and asks the partner to
  confirm the split list in the portal. **Update, same day:** Workflow 13
  showed the real config — `data.fields_to_split` (e.g. `"variants.nodes"`)
  and `data.include` (`"no_other_fields"`), see `references/conventions.md`.
  Set both explicitly when building a Splitter.
- **Top-level records are iterated per record without a Splitter** —
  confirmed from live run metrics (Workflow 11: 10 search calls for 10
  customers; Workflow 12: 40) and from every reference workflow.
- **No fallback-cascade entity-resolution example exists** (e.g. email, then
  phone, then name as successive match attempts). Since 2026-09-24 the skill
  composes this from Building Blocks (chained search → Decision pairs) when
  a scenario asks for it, flagged as a custom flow in Step 9 — not yet
  tested live. Same applies to any other custom composition: the building
  blocks are confirmed, but each new combination is unproven until it runs.
- **The AI-node (`get_chat_completions`) reconciliation pattern is real but
  explicitly not approved for the skill to build from on its own** — see
  Reference Patterns. Needs a deliberate decision, not silent adoption.
- **Considered and deferred: a static operation lookup table**
  (`arise-node-mapping.md`-style file mapping business steps to specific
  apps/operations, to skip repeated live `list_operations` calls for common
  steps). Reasonable idea in principle, but deferred — it would introduce
  its own staleness risk (the same class of problem as the appse-ai-docs
  lag already found) and is a new maintenance burden, not a one-time file.
  Revisit only once live testing shows repeated operation lookups are an
  actual measured speed problem, not before.
- Performance/fragmentation guidance in Step 0b is currently qualitative
  only — no real execution-time or node-count data exists yet to set an
  actual threshold. Revisit once builds with heavier branching have real
  runs to measure.
- No arise-mcp tool currently exposes remaining workflow allocation/quota.
- **Workflow link base URL is hardcoded** to `https://workflow.insync.top`
  (the environment used in testing). The tools only return relative links
  (e.g. `/workflows/{id}/...`). Confirm the production portal URL before
  partners use this, and update Step 11 — or make it configurable per
  environment.
- **Tool-approval prompt volume**: a project-level `settings.json` now
  pre-approves all read-only arise-mcp and Context7 tools, plus
  `save_workflow` (added 2026-09-24 at the team's request — saving follows
  straight on from a create the partner already approved in Step 9, so the
  extra prompt added no real safety). `create_workflow` still prompts
  individually, as the second safety layer beyond Step 9, since each create
  uses a workflow from the partner's allocation. Confirmed this cuts the Claude-Code-level
  approval prompts from ~7 to ~2 per run. Local file reads under
  `references/` are not yet added to this allowlist — first live run with
  the new reference files will show whether they prompt too.

# appse ai Partner Accelerator — Skill POC

## What this project is

We're prototyping "appse ai Partner Accelerator" — a suite of Claude Skills that let
an AI agent help appse ai's implementation partners deliver customer workflows faster,
and with less dependence on deep per-app expertise. This is a real product capability
being designed — not a hackathon artifact. Nothing described in the full design doc is
built beyond the one skill below.

Two problems this addresses, independently:
1. **Speed** — partners build every customized workflow manually today.
2. **Knowledge breadth** — a partner deep in one system (e.g. SAP) is often shallow on
   others in a customer's stack (e.g. Shopify, HubSpot), which limits which deals they
   can confidently scope or build, regardless of available time/headcount.

## Team and repo
Debashree and Nilanjana. Repo: `github.com/appseconnect/appse-ai-skills`, branch
`partner-accelerator-poc`.

Repo layout:
```
.claude-plugin/
  marketplace.json
  plugin.json
.cursor-plugin/        (mirrors .claude-plugin, for portability — see below)
skills/
  workflow-creator/
    SKILL.md
.mcp.json               (project-scoped MCP servers — see Context7 below)
```

**Open item, not yet decided:** `plugin.json`'s `author` field currently uses
`eng@appse.ai`, borrowed from the convention other appse ai plugin repos use. Fine for
now while this is internal/POC, but this plugin is meant to eventually be installed by
external partners — decide on a real owner/support address before any external
distribution.

**KNOWN BUG, fix before the end-to-end re-test:** all three manifests —
`.claude-plugin/plugin.json`, `.cursor-plugin/plugin.json`, and
`.claude-plugin/marketplace.json` (indirectly, via the plugin source) — still list
`skills[]` as `["./skills/partner-init"]`. That folder no longer exists; only
`skills/workflow-creator/` is on disk (renamed during the `partner-init` → real-skill
rewrite). Running the skill directly inside this project folder still works (Claude
Code picks up `skills/` by scanning the folder, not via the manifest), which is why
this went unnoticed — but **installing this as a packaged plugin** (the actual
distribution path, and part of what this POC is meant to prove) would fail to find any
declared skill. Update all three manifests' `skills[]` entries to
`["./skills/workflow-creator"]` before doing any plugin-install test, and before
relying on this repo as "done" for that path.

## Technical architecture (confirmed working)
- **Claude** reasons and follows Skill instructions; has no direct platform access on
  its own.
- **Skill** (`SKILL.md` per skill, in `skills/<name>/`) is the instruction layer.
- **arise-mcp** is appse ai's MCP server — the only component with real read/write
  access to the appse ai platform. Connected via `claude.ai` integration (not
  project-scoped `.mcp.json`), 15 tools available.
- **Context7 MCP** — added project-scoped (see setup below) — gives read-only access to
  appse ai's own official docs repo (`appseconnect/appse-ai-docs`) for field schemas
  and expression syntax. Separate access grant from arise-mcp; never conflate the two.
- **appse ai platform** is where workflows are actually created, stored, and run.

## Context7 MCP setup (needed for Nilanjana too — not yet done on her machine)

1. Get a free API key from context7.com/dashboard (works without one at lower rate
   limits, but this skill calls it on every field-mapping step, so a key is worth it).
2. Set it as a **local OS environment variable** — not a `.env` file (Claude Code does
   not reliably auto-load a project `.env` for MCP substitution on Windows; confirmed
   only real env vars work for the `${VAR}` substitution in `.mcp.json`'s `env` block):
   ```powershell
   [System.Environment]::SetEnvironmentVariable("CONTEXT7_API_KEY","your-key-here","User")
   ```
   Run this yourself, locally — never commit the real key.
3. Restart the terminal/VS Code fully (env var changes don't apply to already-open
   sessions).
4. `.mcp.json` already references `${CONTEXT7_API_KEY}` — safe to pull from the repo
   as-is, no key value is stored in it.
5. Run `claude`, then `/mcp` inside the session — approve the "Use this MCP server"
   prompt the first time (approve only that one server, not "all future servers").
   Confirm `context7 · connected · 2 tools`.
6. Confirmed real tool names (do not assume `get-library-docs` or other guesses):
   **`resolve-library-id`** and **`query-docs`**.
7. Sanity check: ask "use context7 to get the appseconnect/appse-ai-docs Shopify
   customer trigger fields" and confirm it returns real fields (`firstName`,
   `defaultEmailAddress.emailAddress` nested, etc.) — not a generic or wrong answer.

**Windows-specific gotcha hit during setup:** `claude mcp add ... -- npx -y ...` fails
with `error: unknown option '-y'` on Windows — the CLI's argument parser doesn't pass
`-y` through cleanly even with `--`. Use `claude mcp add-json` instead, with an
explicit `"type":"stdio"` field (easy to miss — omitting it fails with a generic
"Invalid input" that doesn't name the missing field), and wrap the command through
`cmd /c` since `npx` isn't directly executable on Windows:
```
claude mcp add-json --scope project context7 "{\"type\":\"stdio\",\"command\":\"cmd\",\"args\":[\"/c\",\"npx\",\"-y\",\"@upstash/context7-mcp\"]}"
```

## POC status

1. **Plumbing tests (`hello-appse-ai`, `list-workflows-test`) — PASSED**, done in a
   scratch folder before this repo existed. Confirmed: skills load and execute
   correctly; a skill can discover and call the right arise-mcp tool from plain
   English; correctly stops on real ambiguity (e.g. multi-org) instead of guessing.

2. **`workflow-creator` skill — rewritten, not yet re-tested end-to-end since rewrite.**
   Originally built and tested (as `workflow-creator-test`) for one scenario: Shopify
   customer created → SAP Business One customer created. That run surfaced real
   lessons (see Guardrails below), all since folded into the current `SKILL.md`.

   **What changed in the rewrite, not yet exercised together in one live run:**
   - Generalized from a hardcoded Shopify/SAP-only skill to accept `source_app`,
     `target_app`, `entity_type` as runtime inputs (no per-app-pair skill sprawl).
   - Added **Step 0**: decomposes multi-event requests (e.g. "the full sales cycle")
     into separate workflows and confirms the total count before building anything,
     since workflows are drawn from the partner's paid allocation. **Note: no arise-mcp
     tool exposes remaining allocation/quota** — the skill can state a count, not
     confirm affordability. This is a known product gap, not a skill bug.
   - Added **Context7 integration** (Steps 6–7): checks appse-ai-docs first for
     documented field schemas and the correct expression syntax, before/alongside the
     live `get_operation_detail` call, which always governs if the two disagree.
   - **Corrected a real error**: the original SKILL.md guessed the field-reference
     syntax as `{{trigger.field}}`. The actual, documented syntax (confirmed via
     appse-ai-docs) is `{{ $payload.field }}` (immediately preceding node) or
     `{{ $('nodeName').payload.field }}` (a named earlier node), JMESPath-style.
   - Added an idempotency check (Step 3): stops and asks before creating a duplicate
     workflow for the same app-pair + entity type.

3. **Immediate next step:** run `workflow-creator` end-to-end on the same confirmed
   scenario (Shopify customer → SAP Business One customer) to verify the full rewritten
   pipeline holds together — org resolution, idempotency check, Context7 lookup
   *inside* the skill (only tested standalone via direct chat so far), the corrected
   expression syntax, and the envelope-structure step — in one real run, not
   piecemeal. Has not been done since the rewrite.

## Reference Workflow (for envelope structure only — see SKILL.md)
- Name: `Shopify Customer synced to SAP SL`
- ID: `a0e88805-6d64-4110-b5e7-42bf93c3d74d`
- URL: https://workflow.insync.top/workflows/a0e88805-6d64-4110-b5e7-42bf93c3d74d/editor
- Used only to learn node/edge JSON shape — never its field values, never for
  expression syntax (that's now documented, see above). Validated only for a simple
  one-trigger, one-action shape; treat anything with branching/multiple
  actions/approval gates as higher-risk until separately verified.

**Note on org switching:** arise-mcp's login is per-organization — whichever email
authenticates it determines which single org is visible in that session, not multiple
orgs at once. Earlier testing used one org (where "Workflow 12" below lives); current
testing uses a different org, **Build Verification Org**
(`d06d36cf-9579-45f1-bf32-bd24cc0c879b`), reached via a different login. Don't expect
workflows or data from a previous org to appear once arise-mcp has been re-authorized
into a different one — check which org is active (`list_organizations`) rather than
assuming continuity across sessions.

**Known cleanup item (scoped to a prior org, not the current one):** an empty stub
workflow, `Workflow 12` (`f9e89e78-d060-4342-afc2-84b220f11794`), was created in an
earlier test org and never completed/saved. It does **not** appear in Build
Verification Org's workflow list — this is expected (different org), not a sign it was
deleted or that `list_workflows` is missing data. No action needed unless/until
testing returns to that original org; if so, delete it via the appse ai UI — no
arise-mcp tool in this skill can delete a workflow.

## Guardrails established so far (apply to every future skill, not just this one)

- **Always re-resolve org_id every run.** Never assume the last-used org still applies.
- **Never guess a data-shape, field mapping, or expression syntax.** Check documented
  sources first (Context7/appse-ai-docs), let the live MCP call govern on conflict, and
  if neither resolves it, stop and ask — or get explicit confirmation on a small
  guessed set. Never silently invent.
- **Never let a skill expand its own tool access mid-run.** If a skill hits a wall
  because its whitelist is too narrow, fix the SKILL.md deliberately afterward — never
  approve ad hoc tool use in the moment (this includes never using Context7 access to
  justify broader arise-mcp use, or vice versa — keep the two grants separate).
  Related: never let an agent copy an *unrelated* existing workflow's structure/field
  logic to work around a gap — that risks importing someone else's business logic.
- **If only a preview-stage operation is available, stop and get explicit
  confirmation** before using it.
- **Keep each skill's tool whitelist narrow and explicit**, split by access type (e.g.
  arise-mcp vs. Context7 get their own separate lists, not one merged list).
- **Decompose multi-event requests into separate workflows** and confirm the total
  count before building — never build multiple workflows silently one at a time.
- **On error, report the exact tool error text** — don't paraphrase or guess an
  alternate approach without saying so.
- **On being asked to stop, stop immediately** with no further tool calls, and report
  exactly what has and hasn't changed.
- **Git discipline:** Claude Code should never commit on its own — commits happen only
  when explicitly asked. Keep this rule in effect for this whole project.

## arise-mcp tool reference (of 15 available, used so far)
`list_organizations`, `list_apps`, `list_operations`, `get_operation_detail`,
`list_credentials`, `list_workflows`, `get_workflow` (scoped: reference workflow only),
`create_workflow`, `save_workflow`

## Immediate next actions, in order
1. Fix the `skills[]` manifest bug above (all three files) — quick, but blocks a
   correct plugin-install test.
2. Run `workflow-creator` end-to-end on the confirmed test scenario (see POC status
   above).
3. Only then, if you want to validate the packaged-plugin path specifically (not just
   running the skill directly): try an actual plugin install using the fixed
   manifests, and confirm `workflow-creator` is discovered correctly.

## After the end-to-end re-test
Design the next real skill (candidates from the design doc: Requirement Digest, SOW
Generator, Cross-App Scenario Advisor). Apply the guardrails above from the start
rather than rediscovering them through trial and error, and use `appse-quality`'s
`qa-azdo-tests` skill (in the org's existing skills) as a structural model — it already
demonstrates good patterns worth reusing: an explicit Inputs table, a
present-summary-then-wait-for-confirmation step, and an idempotent/resumable design.

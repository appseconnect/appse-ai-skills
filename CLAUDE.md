# appse ai Partner Accelerator — Skill POC

*For people (and Claude Code) working on this repo. Installed skills never read this
file — anything a skill needs at run time must live in its own `SKILL.md` or
`references/` folder.*

## What this project is

We're prototyping "appse ai Partner Accelerator" — a suite of Claude Skills that let
an AI agent help appse ai's implementation partners deliver customer workflows faster,
and with less dependence on deep per-app expertise. This is a real product capability
being designed — not a hackathon artifact.

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
.claude/
  settings.json          (shared, committed — pre-approved tool permissions)
  settings.local.json    (personal — gitignored, not committed)
  skills/                (copy of skills/ so Claude Code auto-detects them in this repo)
.claude-plugin/
  marketplace.json
  plugin.json
.cursor-plugin/          (mirrors .claude-plugin, for portability)
skills/                  (the plugin-packaged skills — the real distributable, shipped as-is to partners)
  workflow-creator/
    SKILL.md
    references/
      conventions.md
      guide-building-blocks.md
      guide-expert-review.md
      guide-field-mapping.md
      guide-steps-detail.md
      pattern-*.json     (real exported workflows, structure reference only)
  requirement-digest/        (Nilanjana — not yet reviewed)
  sow-generator/             (Nilanjana — not yet reviewed)
  uat-test-script-generator/ (Nilanjana — not yet reviewed)
docs/
  workflow-creator-known-limits.md   (maintainer notes — internal, never packaged with the skill)
.mcp.json                (project-scoped MCP servers — Context7, for local dev)
.gitignore               (.claude/settings.local.json, *.zip, *.plugin)
CLAUDE.md
README.md
```

**Two copies of every skill must stay identical.** `skills/<name>/` is the packaged
source of truth; `.claude/skills/<name>/` is a copy so Claude Code picks the skill up
when working in this repo. Edit `skills/` first, then resync the whole folder
(including `references/`) into `.claude/skills/`. Check with a diff before committing.

**Source-of-truth rule inside `workflow-creator`:** `SKILL.md` owns Inputs, all
Workflow steps (0–11), and Allowed Tools. The `references/guide-*.md` files hold only
fuller detail behind sections `SKILL.md` condenses. If a guide and `SKILL.md`
disagree, `SKILL.md` wins — fix the guide. Maintainer notes (validated scenarios, open
platform questions, deferred ideas) live in `docs/workflow-creator-known-limits.md` —
deliberately **outside** `skills/`, since anything under a skill's own folder ships to
partners in the packaged zip/plugin, and this file names internal IDs, hosts, and
incident details that aren't partner-facing.

**Open items:**
- `plugin.json`'s `author` uses `eng@appse.ai`, borrowed from other appse ai plugin
  repos. Fine for internal POC; decide a real owner/support address before external
  distribution.
- The three newer skills haven't been reviewed with the same scrutiny as
  `workflow-creator` — review them (tool whitelists, guardrails, no invented shapes)
  before relying on them, and add them to the manifests' `skills[]` only once reviewed.

## Technical architecture (confirmed working)
- **Claude** reasons and follows Skill instructions; has no direct platform access on
  its own.
- **Skill** (`SKILL.md` + `references/`) is the instruction layer.
- **arise-mcp** is appse ai's MCP server — the only component with real read/write
  access to the platform. Connected as a `claude.ai` connector (not via `.mcp.json`),
  15 tools. Its login is per-organization — see Org switching below.
- **Context7** gives read-only access to appse ai's official docs,
  `/appseconnect/appse-ai-docs` (app fields and example records, expression syntax,
  node pages). The skill calls `query-docs` directly with that library ID —
  `resolve-library-id` is no longer used. Separate access grant from arise-mcp; never
  conflate the two.
- **appse ai platform** is where workflows are created, stored, and run.

## Context7 — local development setup (Claude Code in VS Code)

Per the platform team, Context7 is picked up automatically for partners, with no
setup on their side (still to confirm with a real run in Cowork — see
`docs/workflow-creator-known-limits.md`). The steps below are only for running the
skill locally in Claude Code from this repo, where `.mcp.json` provides it:

1. Optional: get a free API key from context7.com/dashboard (it works without one at
   lower rate limits).
2. Set it as a **local OS environment variable** — not a `.env` file (Claude Code on
   Windows doesn't reliably load a project `.env` for `${VAR}` substitution in
   `.mcp.json`):
   ```powershell
   [System.Environment]::SetEnvironmentVariable("CONTEXT7_API_KEY","your-key-here","User")
   ```
   Run it yourself; never commit the real key. `.mcp.json` only references
   `${CONTEXT7_API_KEY}`.
3. Fully restart the terminal / VS Code.
4. Run `claude`, then `/mcp`; approve "Use this MCP server" for Context7 only (not
   "all future servers"). Confirm `context7 · connected · 2 tools`.
5. Sanity check: "Load the library from context7 /appseconnect/appse-ai-docs and tell
   me the details of actions available for magento" — it should return real,
   specific content.

**Windows gotcha:** `claude mcp add ... -- npx -y ...` fails with
`error: unknown option '-y'`. Use `claude mcp add-json` with an explicit
`"type":"stdio"` (omitting it gives a generic "Invalid input") and wrap through
`cmd /c`:
```
claude mcp add-json --scope project context7 "{\"type\":\"stdio\",\"command\":\"cmd\",\"args\":[\"/c\",\"npx\",\"-y\",\"@upstash/context7-mcp\"]}"
```

## Testing in Cowork (browser)
- **arise-mcp:** already available through the same claude.ai connectors — check
  Settings → Connectors.
- **Context7:** now also shows up as its own claude.ai connector ("Context7"), separate
  from the local `.mcp.json` entry used for Claude Code dev — confirm it's authorized
  there, and that it actually answers `query-docs` calls, on the first real Cowork run.
- **The skill — packaging path not yet confirmed.** Two possible upload surfaces have
  come up and neither has been checked against a real Cowork Customize/Settings menu:
  - A **skill zip**: `SKILL.md` at the top level plus `references/`, uploaded via
    Settings → Capabilities → Skills → Upload. Zip only `skills/workflow-creator/`.
  - A **plugin package** (`.claude-plugin/` + `skills/workflow-creator/` + `README.md`,
    renamed `.plugin`), uploaded via Customize → Plugins → Upload, if that surface
    exists.
  Open Cowork's Customize/Settings menu first to see which one is real, then package
  only `skills/workflow-creator` (never all of `skills/` — the three newer skills
  aren't reviewed yet) and always zip from the committed folder, never a pasted copy.

## Tool permissions (`.claude/settings.json`)
Pre-approved: all read-only arise-mcp tools (including `get_node_data`),
`save_workflow`, and Context7's `query-docs` — under **both** connector prefixes,
`mcp__context7__` (local `.mcp.json`, Claude Code dev) and `mcp__claude_ai_Context7__`
(claude.ai connector, used in Cowork). Also pre-approved: `Read` on
`skills/workflow-creator/**` and `.claude/skills/workflow-creator/**`, so reviewing the
skill's own files doesn't prompt. Deliberately **not** pre-approved: `create_workflow`
— it uses a workflow from the partner's allocation, so it keeps its own prompt as a
second safety layer after the skill's Step 9 confirmation. `resolve-library-id` can be
removed from the allowlist under both prefixes (unused now). The real rule prefix for
arise-mcp is `mcp__claude_ai_arise-mcp__<tool>` (from Claude Code's own output — don't
guess it).

## POC status
- **Plumbing tests** (`hello-appse-ai`, `list-workflows-test`) — passed.
- **`workflow-creator`** — simple syncs built cleanly (e.g. SAP B1 item → Shopify
  product, "Workflow 9"); branching builds (Filter / Decision / Splitter) exercised in
  Workflows 11–18 on 2026-09-24, which surfaced the failures today's rules are built
  on. Full list, and what's still untested, in
  `docs/workflow-creator-known-limits.md`.
- **Recent changes (2026-09-24):** live reference workflow dropped (local
  `references/` cover every shape); Context7 called directly with the library ID and
  node pages in scope; questions sorted into decide yourself / assume and state /
  must ask (five or fewer); optional scope summary for large or vague requests (no
  SOW required); `effort` frontmatter removed; guides trimmed so `SKILL.md` is the
  single source of truth for the steps.

## Org switching and environment notes
- **Per-org login:** whichever email authenticates arise-mcp decides the single org
  visible in that session. Current testing uses **Build Verification Org**
  (`d06d36cf-9579-45f1-bf32-bd24cc0c879b`). Don't expect data from an earlier org —
  check `list_organizations`.
- **Two different "Workflow 12"s — don't mix them up.** The one in
  `docs/workflow-creator-known-limits.md` (the 53,080-record incident) is in Build
  Verification Org. A
  separate empty stub also named "Workflow 12"
  (`f9e89e78-d060-4342-afc2-84b220f11794`) sits in the earlier test org; delete it in
  the UI if testing returns there (no arise-mcp tool can delete a workflow).
- **UI outages don't affect builds.** Confirmed with dev: arise-mcp writes to the same
  database the UI reads from, no caching. During a real `workflow.insync.top` outage,
  arise-mcp kept working. A build during an outage is safe; only visual verification
  is delayed.

## Guardrails (apply to every skill in this repo)
- **Re-resolve the org every run** — never assume the last-used org.
- **Never guess a data shape, field mapping, function name, or expression syntax.**
  Docs decide meaning, the live operation detail decides required fields,
  `conventions.md` decides the saved JSON; if none resolves it, ask.
- **Never let a skill expand its own tool access mid-run.** Fix the `SKILL.md`
  deliberately afterwards. Keep each grant (arise-mcp, Context7, local files)
  separate and narrow. Never read another workflow's logic to work around a gap.
- **Operations:** release and preview are fine; never dev.
- **Follow the data flow:** a node reads from the node that shaped its record — never
  reach back past a Filter or Decision to the trigger. Never copy node names from a
  reference file.
- **Never save a blank value**, including nested fields; never copy company-specific
  settings (currency, price list, warehouse, tax code) from a reference.
- **One workflow per business event;** show the plan and count before building.
- **Partner-facing text** never names internal tools or systems.
- **On error**, report the real problem plainly (strip internal names, keep the
  substance); retry at most once. **On "stop"**, stop immediately and report what
  has and hasn't changed.
- **Git discipline:** Claude Code never commits on its own — commits happen only when
  explicitly asked.

## Next actions
1. Open Cowork's Customize/Settings menu to confirm which upload surface is real
   (skill zip vs. plugin package — see Testing in Cowork) before packaging.
2. Package `skills/workflow-creator` only, upload it, confirm arise-mcp and the
   Context7 connector both work there, and run one branching scenario end to end.
3. Open the PR to `main` once the Cowork run is confirmed.
4. Review the three newer skills before relying on them or adding them to the
   manifests.
5. Decide `plugin.json`'s real author/support email before external distribution.

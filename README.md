# appse-partner

**appse ai Partner Accelerator** — skills that help appse ai implementation partners
scope and build customer workflows faster, even across apps they don't know deeply.

It targets two problems:

- **Speed** — partners build every customized workflow by hand today.
- **Breadth** — a partner who is deep in one system (e.g. SAP) is often shallow on
  others in the customer's stack (e.g. Shopify, HubSpot), which limits the deals they
  can confidently scope or build.

> Standalone plugin — no `appse-core`. Skills reach the appse ai platform only through
> **arise-mcp**, which must be connected in your AI tool before any skill runs.

Status: **POC** on branch `partner-accelerator-poc`. Not yet listed in
[appse-marketplace](https://github.com/appseconnect/appse-marketplace).

---

## Skills

| # | Skill | Use it when | Status |
|---|-------|-------------|--------|
| 1 | **`/partner-init`** | First run — confirms arise-mcp is connected and which org to use. | Placeholder |

Planned: `partner-workflow-build`, `partner-requirement-digest`, `partner-sow-generator`,
`partner-scenario-advisor`.

---

## Structure

```
.claude-plugin/plugin.json        Claude Code manifest
.claude-plugin/marketplace.json   local catalog, so the plugin installs before marketplace listing
.cursor-plugin/plugin.json        Cursor manifest
skills/<skill-name>/SKILL.md      skill definition
skills/<skill-name>/references/   templates and rules this skill uses
```

Conventions follow `appse-core/conventions/naming.md`:

- Skill names: `partner-` prefix, kebab-case.
- Trigger logic lives only in the frontmatter `description`.
- Each skill is self-contained — shared rules are copied into its own `references/`,
  never read from another skill's folder.
- SKILL.md order: Outputs → Inputs → Allowed tools → Preconditions → Workflow →
  What "done" looks like → Output Rules.
- When adding a skill, add it to `skills[]` in **both** plugin manifests.

---

## Guardrails (master copy)

Every skill must follow these. Copy this list into each skill's
`references/guardrails.md` and re-copy when it changes.

1. **Re-resolve the org every run.** Never assume the last-used org still applies.
2. **Never guess a data shape or field mapping.** If the structure or fields can't be
   retrieved, stop and ask — or use a clearly labeled, named fallback and flag it as
   an assumption to verify. Never silently invent.
3. **Narrow, explicit tool whitelist.** Each SKILL.md lists exactly which arise-mcp
   tools it may use, and nothing else.
4. **Never expand tool access mid-run.** If the whitelist is too narrow, stop. Fix the
   SKILL.md deliberately afterward.
5. **Preview-stage operations need explicit confirmation** before use — not a passing
   mention.
6. **On error, report the exact tool error text.** Don't paraphrase, and don't try an
   alternate approach without saying so.
7. **On "stop", stop immediately.** No further tool calls; report exactly what has
   and hasn't changed.

---

## Install (Claude Code)

Works once these manifests are on `main` — the marketplace is read from the default branch.

```text
/plugin marketplace add appseconnect/appse-ai-skills
/plugin install appse-partner@appse-partner-catalog
```

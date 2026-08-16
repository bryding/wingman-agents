# wingman-agents

Claude Code **plugin marketplace** for Wingman. Harness skills and delegation subagent roles
live here — NOT in the wingman repo — so one checkout serves every clone and worktree of
wingman. See README.md for install instructions.

## Layout

```
.claude-plugin/marketplace.json     Claude Code marketplace — 1 entry, with a version
.agents/plugins/marketplace.json    Codex marketplace — same plugin, NO version here
plugins/wingman/
  .claude-plugin/plugin.json        Claude plugin manifest — name + version
  .codex-plugin/plugin.json         Codex plugin manifest — name + version + skills glob
  skills/<skill>/SKILL.md           SHARED by both tools. Frontmatter: name + description
  agents/<role>.md                  Claude Code ONLY — Codex roles stay in the wingman
                                    repo's .codex/agents/*.toml
bumpVersion.sh                      the ONLY sanctioned way to change a version — updates
                                    all three manifests at once; --check audits for drift
registerAgents.sh                   idempotent per-machine bootstrap for BOTH tools; also
                                    records this checkout's path to
                                    ~/.claude/wingman-agents-checkout so publish-harness
                                    can find it on any machine
```

## Updating skills — the publish workflow

Installed plugins are served from a **cache copy** keyed on version, NOT from this working
copy, and the cache refreshes **only on a version change**. Editing a file here does nothing
to live sessions until you publish:

1. Edit under `plugins/wingman/`.
2. Bump: **`sh bumpVersion.sh wingman [patch|minor|major]`** — never by hand. Run
   `sh bumpVersion.sh --check` before committing.
3. Commit and push.
4. On each machine: `sh registerAgents.sh`, then restart open sessions
   (Codex: `/reload-plugins`).

No version bump = no publish — the update reports "already at the latest version" and
silently serves old content. The `publish-harness` skill in the plugin carries this same
workflow; keep the two in sync when it changes.

## Rules

- Wingman's safety language in skills and role prompts (advisory-only, never invent
  operational facts, evidence/abstention) is load-bearing. Do not weaken it while editing.
- Skill bodies stay runtime-neutral between Claude Code and Codex; never fork per tool.
- The manifests are UTF-8 with literal em-dashes; edit with Edit/Write, not scripted
  re-serialization.
- The wingman repo's `pnpm check:agents` refuses recreated local copies of these skills —
  edits happen here, followed by a version bump, or they reach nobody.

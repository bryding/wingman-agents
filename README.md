# wingman-agents

A [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) — and a
Codex one, from the same tree — holding the agent tooling for
[Wingman](https://github.com/bryding/wingman): harness skills and delegation subagent roles.

These used to live in `.claude/` inside the wingman repo, which made them **branch-scoped** —
an older branch got older skills, and every worktree carried its own copy. Installed as plugins
they live at the user level instead, so one checkout of this repo serves every clone and
worktree of wingman.

## Install and update

Always go through the script — once per machine to install, and again any time to update:

```
sh registerAgents.sh
```

One script covers **both** Claude Code and Codex: it registers the marketplace and installs the
`wingman` plugin with whichever of the `claude` / `codex` CLIs are on `PATH`, and skips the one
that isn't. Re-running it refreshes the marketplace from GitHub and updates the installed
plugins; every clone and branch picks up the new version at once — nothing to commit in the
wingman repo. Restart open sessions afterward (Codex: `/reload-plugins`); plugins are
discovered at session start only.

Other modes: `--claude` / `--codex` (limit to one tool), `--reinstall` (remove and re-add, for
a stale clone), `--remove`, `--help`.

## The `wingman` plugin

| Component | Contents |
|---|---|
| Skills | `wingman-review` (Sonnet-pinned multi-agent diff review), `deep-think` (hand a hard open problem to `deep-thinker` with an unanchored brief), `handoff` / `resumeFromHandoff` (the context-clear loop around the machine-local `local-handoff.md` baton), `docs-update` (repo documentation audit), `publish-harness` (how to edit and publish anything in this repo) |
| Roles (Claude Code only) | `Explore`, `scout`, `mech-executor`, `verifier` — the pinned cheap-model delegation workers routed by wingman's `AGENTS.md` — plus `deep-thinker` (Fable @ high effort, the one deliberately expensive role; used only via `deep-think`) |

Skill invocations are namespaced: `/wingman:handoff`, `/wingman:wingman-review`, and so on.

## What deliberately stays in the wingman repo

- **`AGENTS.md` / `CLAUDE.md`** — canonical project guidance, safety boundary included; it
  tracks the branch it belongs to.
- **`.specify/` and the ten `speckit-*` skills** — Spec Kit machinery is coupled to `specs/`
  in the repo, and wingman's `speckit-plan` intentionally diverges from the generic donor
  version (it maintains wingman's managed `AGENTS.md` block). They stay under
  `.claude/skills/`, bridged to Codex via `.agents/skills/` symlinks, guarded by
  `pnpm check:agents`.
- **`.codex/agents/*.toml`** — Codex plugins cannot carry subagent roles, so the Codex role
  adapters stay repo-local. Shared skill bodies come only from this marketplace.
- **Hooks** — repo-scoped enforcement (the ci-local pre-push gate) lives in wingman's
  committed `.claude/settings.json`, because plugin hooks would fire in every repo on the
  machine.

## Authoring and publishing

Skills live at `plugins/wingman/skills/<name>/SKILL.md` (frontmatter: `name:`,
`description:`); roles at `plugins/wingman/agents/<name>.md` (`name`, `description`, `model`,
`effort`, `tools`).

Installed plugins are served from a per-version cache, so editing a file here does nothing to
live sessions until you publish:

1. Edit under `plugins/wingman/`, and **test before publishing**: `claude plugin validate
   ./plugins/wingman --strict` for structure, and `claude --plugin-dir ./plugins/wingman` to
   load the edited plugin in a live session without publishing anything (it shadows the
   installed copy for that session only).
2. `sh bumpVersion.sh wingman [patch|minor|major]` — never edit version fields by hand; it
   updates all three manifests that record the version and verifies they agree
   (`--check` audits without bumping).
3. Commit and push. Repo CI re-checks version consistency, bump-on-change, manifest validity,
   and dead references.
4. On each machine: `sh registerAgents.sh`, then restart open sessions.

Forgetting step 2 is the classic failure: the update reports "already at the latest version"
and silently serves old content. The `publish-harness` skill carries this same workflow so a
session in any repo knows to come here.

## Model pins

Roles pin model **aliases**, which float to the current generation (`haiku` → current Haiku,
`sonnet` → current Sonnet): `Explore`/`scout`/`verifier` = haiku @ low effort,
`mech-executor` = sonnet @ medium, `deep-thinker` = fable @ high (the premium tier, on purpose
— it exists for the few problems worth it; everything else stays cheap). `wingman-review` pins
its worker agents to `sonnet` purely for cost. Codex-side pins live in the wingman repo's
`.codex/agents/*.toml`.

Two open questions, deliberately recorded rather than guessed (per Anthropic's guidance to
re-run effort sweeps per model generation, reaching for lower *effort* before a cheaper
*model*):

- **`wingman-review`: Opus-at-low-effort finders vs the Sonnet pin** has never been A/B'd on
  a diff with known findings. Until measured, Sonnet stays as the cost-safe default.
- **The delegation crossover point** — whether work currently delegated to `scout`/`Explore`
  repays its coordination cost on current-generation drivers — is unmeasured; the routing
  policy in wingman's `AGENTS.md` is unchanged in spirit.

## Codex support

The same repo is a Codex marketplace (`.agents/plugins/marketplace.json`); both tools read the
same `plugins/wingman/skills/` tree. `registerAgents.sh` drives the `codex plugin` CLI
(~v0.121+); on an older build it prints the in-session equivalents instead. Two asymmetries:
plugin-packaged subagent roles are Claude-only (Codex roles stay in the wingman repo's
`.codex/agents/`), and skill bodies stay runtime-neutral — never fork a skill body per tool.

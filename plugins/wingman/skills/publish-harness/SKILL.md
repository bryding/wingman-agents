---
name: publish-harness
description: Add, edit, or fix a Wingman harness skill or delegation role and publish it so every machine, clone, and worktree gets it. Use whenever the user asks to change how a harness skill behaves, add a new skill or role, or fix something a skill got wrong. Harness skills do NOT live in the wingman repo — they live in the wingman-agents marketplace repo, and edits there require a version bump to reach live sessions.
---

# Publishing Wingman harness skills and roles

The harness skills (`wingman-review`, `handoff`, `resumeFromHandoff`, `docs-update`,
this skill) and the four delegation roles (`Explore`, `scout`, `mech-executor`,
`verifier`) are NOT in the wingman repo. They live in the **wingman-agents**
marketplace repo (`https://github.com/bryding/wingman-agents`) and are installed as
Claude Code / Codex plugins at user scope. Editing anything under a wingman-repo
`.claude/` directory does nothing for these — those copies were removed on purpose.
(The `speckit-*` skills are the exception: they stay repo-local in wingman, and
`pnpm check:agents` guards them there.)

Live sessions read from each tool's own clone plus a per-version cache, **not** from
any working checkout. An edit only reaches anybody after a version bump + push +
update.

## 1. Locate the working checkout — STOP if it cannot be verified

The checkout can be anywhere — its location differs per machine, so never assume a
path. `registerAgents.sh` records where it lives when it runs. Read that marker:

```sh
cat ~/.claude/wingman-agents-checkout
```

It must hold one absolute path. Check ALL of:

1. The marker file exists and is non-empty.
2. It contains exactly one path — not multiple lines, not a comment.
3. That path exists, is a directory, and contains `.claude-plugin/marketplace.json`.
4. That directory is a git repo (`git -C <path> rev-parse --show-toplevel` succeeds).

### If ANY check fails: STOP and do nothing else

Report which check failed, the exact value found (or that the marker was missing),
and the fix:

> Run `sh registerAgents.sh` from your wingman-agents checkout to re-record the path.
> If there is no checkout on this machine, clone one first:
> `git clone https://github.com/bryding/wingman-agents`

Then **end your turn**. Do not work around it. Specifically, do NOT:

- clone the repo yourself to "fix" it, or write the marker file yourself
- guess or search for a checkout in other directories
- edit `~/.claude/plugins/marketplaces/wingman-agents/` (the managed clone —
  marketplace updates reset it, so edits there are silently lost)
- recreate any of these skills/roles under a wingman-repo `.claude/` directory
  (edits there reach nobody, and `check:agents` refuses them)
- make the requested change anywhere else, or hold it "for later"

### If every check passes

`git -C <checkout> pull` before editing so you are not branching off stale content.

## 2. Make the change

Skills: `plugins/wingman/skills/<name>/SKILL.md` (frontmatter needs `name:` and
`description:`; supporting files live next to it). Roles:
`plugins/wingman/agents/<name>.md` (frontmatter: `name`, `description`, `model`,
`effort`, `tools`). Keep skill bodies runtime-neutral between Claude Code and Codex,
and keep Wingman's safety language intact — advisory-only wording and the
never-invent-operational-facts rules in role prompts are load-bearing, not
boilerplate.

## 3. Publish

1. Bump the version: `sh bumpVersion.sh wingman [patch|minor|major]` — never edit
   version fields by hand; it updates all three manifests and verifies they agree.
2. Commit and push (`origin main`). CI on the repo re-checks version consistency,
   bump-on-change, manifest validity, and dead references.
3. On each machine: `sh registerAgents.sh` to pull and update, then restart open
   sessions (Codex: `/reload-plugins`). No version bump = no publish — the classic
   failure is an edit that "already at the latest version" silently never serves.

---
name: docs-update
description: Audit and refresh all Wingman repository documentation after code, product, workflow, or agent-configuration changes. Use for documentation sweeps, stale-doc fixes, README updates, docs navigation/index maintenance, instruction-file cleanup, or requests to make Claude Code and Codex guidance consistent.
---

# Update repository documentation

Bring the documentation back into agreement with the repository without changing
product behavior, historical decisions, or safety boundaries merely to make the docs
look consistent.

## Workflow

1. **Inventory before editing.** Run `git status --short` and enumerate Markdown with
   `rg --files -g '*.md' -g '!node_modules'`. Include root instructions, `.agents/`,
   `.claude/`, `.specify/`, `docs/`, package/service/fixture READMEs, and active specs.
   Preserve unrelated user edits.

2. **Establish sources of truth.** Read `AGENTS.md`, `.specify/memory/constitution.md`,
   `docs/INDEX.md`, the active feature's `spec.md`, `plan.md`, and `tasks.md`, plus the
   implementation or config that the affected docs describe. Treat code and tests as
   the source for current mechanics, the constitution/specs as the source for approved
   behavior, and dated decision logs as historical records.

3. **Update by audience.** Keep each surface focused:

   - `README.md`: concise and developer-oriented. Include only purpose/safety status,
     prerequisites, setup, run/verify commands, repository map, and links to deeper
     docs. Move research, product narrative, and long policy explanations elsewhere.
   - `AGENTS.md`: canonical always-on guidance for both tools, no more than 150 lines.
     Keep safety rules, current scope, commands, Spec Kit state, Slack/founder rules,
     delegation policy, and correct links to `docs/INDEX.md`, the constitution, active
     plan/tasks, and `.claude/memory/MEMORY.md`.
   - `CLAUDE.md`: no more than 150 lines and normally a thin `@AGENTS.md` compatibility
     entrypoint. Do not duplicate the canonical guidance.
   - `docs/INDEX.md`: keep a navigable map of every file under `docs/`, then point to
     active specs, package READMEs, agent setup, and durable memory. Distinguish
     current guidance, time-sensitive research, trackers/logs, and historical reviews.
   - `docs/`, service, fixture, and package docs: update commands, paths, status, and
     links from the implementation. Keep detailed material out of the root README.
   - Spec Kit artifacts: preserve their roles and approval gates. Do not silently turn
     a documentation cleanup into a scope, taxonomy, safety, or architecture decision.

4. **Protect history and uncertain facts.** Founder feedback, decisions, handoffs, and
   dated reviews are append-only unless correcting a clear transcription error. Keep
   dates and superseded conclusions visible. For provider terms, availability, laws,
   prices, or other time-sensitive claims, verify primary sources before asserting a
   current fact; otherwise preserve the date and label it unverified.

5. **Maintain the shared agent surface.** The harness skills (this one, `handoff`,
   `resumeFromHandoff`, `wingman-review`, `publish-harness`) and the four delegation
   roles live in the **wingman-agents marketplace repo**
   (`https://github.com/bryding/wingman-agents`), not in the wingman repo — edit them
   there via the `publish-harness` skill, never by recreating copies under the wingman
   repo's `.claude/`. Only the `speckit-*` skills remain repo-local under
   `.claude/skills/`, bridged to Codex through links in `.agents/skills/`; repair
   missing links but never copy skill bodies into `.agents/skills/`. Keep `AGENTS.md`
   canonical and keep Spec Kit configured to update it.

6. **Validate.** Run:

   ```sh
   pnpm check:docs
   pnpm check:agents
   ```

   Also run any commands whose documented behavior changed. Review `git diff` for
   accidental decision changes, stale links, duplicated guidance, and claims of tests
   or approvals that did not happen. Report exact verification results and any factual
   drift that needs a founder or licensing decision.

---
name: resumeFromHandoff
description: Resume Wingman work after a context clear by reading the machine-local `local-handoff.md` that /handoff wrote. Use when the user says "resume", "pick up where we left off", or invokes this right after clearing context — it replaces copy-pasting a next-session prompt.
---

# resumeFromHandoff

Pick up exactly where the previous session left off. `/handoff` wrote a resume prompt to
**`local-handoff.md`** at the repo root (gitignored, machine-local). Read it and follow it.

**Invocation**: `/wingman:resumeFromHandoff` (Claude Code) · the
`wingman:resumeFromHandoff` skill (Codex).

## Procedure

1. **Read `local-handoff.md` at the repo root.**

   If it is missing, do NOT guess a task. Say so, then offer the fallbacks in order:
   - the dated handoff in the active feature's `specs/<NNN>-*/plan.md` (find the feature via
     `.specify/feature.json` if present),
   - available project memory for an ACTIVE pointer,
   - `git log --oneline -15` plus `git status` to infer recent work.
   Ask which to use.

2. **Check staleness.** The header records the branch and the HEAD it was written at. Compare with
   the current branch and `git rev-parse HEAD`:
   - Same HEAD → proceed.
   - HEAD moved → say so, skim `git log --oneline <written-sha>..HEAD`, and treat the prompt's
     state claims as needing re-verification.
   - Different branch → stop and ask. Never silently apply a handoff from another branch.

3. **Follow the prompt as if the user had just typed it** — do its reading list in order, honour its
   working-style and safety sections, and carry out its task.

4. **Verify before relying on it.** It was written by a session that no longer exists. Re-check
   anything load-bearing against the code and `git status` before building on it — particularly
   "X is done" / "Y is not proven", and any **source, licensing, or replay-fixture provenance**
   claim, which in this repo is product behaviour rather than disclaimer polish and must not be
   inherited on trust.

5. **Do not delete or edit `local-handoff.md`.** The next `/handoff` overwrites it wholesale.

## Why this exists

The durable record is the committed handoff in the repo; `local-handoff.md` is only the baton —
a machine-local pointer saying "read that, then do this". It is gitignored deliberately: it
describes one moment on one machine and must never be mistaken for project state.

Companion skill: `handoff` (writes the file). The loop is `/wingman:handoff` → clear →
`/wingman:resumeFromHandoff`, with nothing to copy.

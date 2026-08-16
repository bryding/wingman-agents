---
name: handoff
description: Prepare a durable Wingman checkpoint before context is cleared or work moves between Claude Code and Codex. Capture completed work, next steps, blockers, safety/licensing decisions, and Spec Kit state in the active plan plus available memory/docs so a fresh coding agent can resume safely.
---

# Handoff: prepare for a context clear

Conversation context will disappear. Distill the current Wingman state into durable
locations so a fresh coding agent can resume without guessing. Preserve details that were
expensive to learn, especially corrections, source/license constraints, replay fixture
provenance, safety gates, and unverified claims.

## Model: pin the write-up to the Opus tier

The `Skill` tool has no model parameter — a skill always runs in whatever model the
calling session is already on, and Wingman sessions often run on the premium tier
(Fable-class), which is unnecessarily expensive for structured distillation. So this
skill pins its actual synthesis work to the `opus` alias instead, regardless of caller
(the alias floats to the current Opus generation; re-point only if a cheaper tier
demonstrably preserves handoff fidelity):

1. In the **current session**, do step 1 below (Take stock) and recall every
   session-specific fact only this conversation knows — corrections, dead ends,
   decisions, unverified claims, anything expensive to learn that a fresh agent
   could not reconstruct from files alone. A brand-new `Agent` call starts with no
   memory of this conversation, so anything not written into the brief is lost.
2. Compile that into a **self-contained brief** (plain prose is fine, no fixed
   format required).
3. Perform steps 2–9 below via a **single `Agent` tool call** with
   `subagent_type: 'general-purpose'` (needs Read/Edit/Write/Bash) and
   `model: 'opus'` explicitly set. Pass the brief from step 2, plus the full text of
   steps 2–9, as the prompt — the agent needs no other context to complete the
   handoff, including writing `local-handoff.md` in step 9.

## Procedure

1. **Take stock.** Record the current branch, active `specs/<NNN>-*/` feature, recent
   commits (`git log --oneline -15`), active work, decisions, dead ends, and corrections.

2. **Protect work in progress.** Run `git status`. Commit coherent work when authorized
   and appropriate; otherwise state exactly what remains uncommitted and where. Never
   claim tests, evaluations, licenses, or live validation that did not occur.

3. **Checkpoint the active Spec Kit plan.** Prepend a dated `Session Handoff` to the
   active `specs/<NNN>-*/plan.md` and mark the previous current handoff `SUPERSEDED`.
   Then move every `SUPERSEDED` handoff section verbatim into the feature's
   `handoff-archive.md` sibling (create it with a short "superseded checkpoints —
   never resume from this file" header if absent, newest first), leaving plan.md with
   only the new CURRENT handoff plus a one-line pointer to the archive — plan.md is
   mandatory session-start reading, and dead handoffs left in place cost every future
   session tokens. Never delete handoff history, only relocate it, and commit the
   archive file together with plan.md (a plan.md pointer to an uncommitted archive is
   a broken link in CI). If no feature is active, write the checkpoint in the most
   relevant durable project document. Include:

   - **START HERE**: exact files to read first;
   - **ONE-LINE STATE**: what the work is;
   - **DONE + committed**: commit subjects/hashes and actual verification results;
   - **NOT DONE / NOT PROVEN**: bluntly identify incomplete or unverified claims;
   - **NEXT STEP**: a concrete, executable action;
   - **BLOCKERS / OPEN QUESTIONS**: especially founder, licensing, FAA access, data,
     safety, and product-scope decisions;
   - **KEY FACTS / COMMANDS**: paths, fixture IDs, source dates, evaluation commands,
     and gotchas; and
   - **SPEC CHECKPOINT**: which spec/plan/tasks items are complete, any constitution
     concerns, and whether implementation is authorized.

4. **Update memory if available.** Update `.claude/memory/MEMORY.md` and the focused
   memory files it links with durable facts and corrected premises. The directory name
   is legacy but the contents are shared project context for Claude Code and Codex. If
   no memory store exists, do not invent a private path; keep the facts in the plan
   handoff. Never put credentials, licensed audio, private pilot data, or restricted
   raw feeds in memory.

5. **Repair documentation drift.** Update any research, architecture, contract, or
   quickstart document that became inaccurate. Time-stamp provider/licensing findings
   and distinguish source fact from inference.

6. **Run the safety/spec checkpoint.** Confirm advisory-only language, evidence and
   abstention requirements, public/authorized-data restrictions, the no-network-bypass
   rule, and KDEN replay scope remain intact. Re-run applicable spec checks or tests;
   record exact results rather than saying “looks good.” Do not begin new product work.

7. **Commit the durable handoff** when repository policy and user authority allow it,
   using a message such as `handoff: <feature> — <one-line state>`. If a commit is not
   possible, say so prominently and leave the working tree paths explicit.

8. **Confirm safety to clear** in three to five lines: one-line state, most important
   next step, commit/uncommitted status, and the exact START HERE path. Finish by telling the
   user that after clearing they just run **`/wingman:resumeFromHandoff`** (Claude
   Code) or the `wingman:resumeFromHandoff` skill (Codex) — there is nothing to copy
   or paste.

9. **Write the next-session prompt to `local-handoff.md` at the repo root.** This file is
   **gitignored and machine-local** — the baton handed to the next session, NOT durable state.
   The durable record is the committed handoff from the earlier steps; this only points at it and
   says what to do next. Overwrite it wholesale each time (it describes ONE handoff, never a log).

   Do NOT print the prompt into the chat and do NOT copy it to the clipboard — the user should not
   have to select, copy, or paste anything. Write the file and say it is ready.

   Begin the file with a header so a resuming session can detect staleness:

   ```markdown
   <!-- written by /handoff — gitignored, machine-local, safe to delete -->
   # Resume prompt — <work name>

   - **Written:** <UTC timestamp>
   - **Repo / branch:** <path> / <branch>
   - **HEAD at write time:** <full sha>

   ---

   <the prompt itself>
   ```

   The prompt body tells a zero-context coding agent: the project/branch and "you have zero prior
   context — read the handoff before doing anything"; the ordered reading list; the current
   done/NOT-proven state with any earlier false claim retracted by name; the concrete next task;
   the safety/licensing/provenance constraints that apply; and the working style to carry forward.
   Keep it directive — the detail lives in the committed handoff it points to.

## Notes

- Put detail in the committed handoff and keep chat concise.
- Explicitly retract earlier claims that proved wrong.
- “Advisory only” and evidence provenance are product behavior, not disclaimer polish.
- A replay is not proven unless its source permissions, time alignment, and expected
  anomaly labels are recorded.

---
name: deep-think
description: Hand a hard problem to the deep-thinker agent (Fable, high effort) — spec/plan authoring, adversarial review of a plan or diagnosis, or root-causing a defect that resisted a first pass. Builds the brief so the agent is grounded but NOT anchored by the driver's assumptions.
---

# deep-think

Delegate a hard problem to `deep-thinker` (Fable, high effort). The driver model stays the main
worker; this skill buys lateral thinking on the problems where that matters, without paying
Fable rates for ordinary work.

**Invocation**: `/wingman:deep-think <problem>` (Claude Code) · the `wingman:deep-think` skill
(Codex). `$ARGUMENTS` is the problem statement accompanying this invocation.

## When to use it

- Authoring a spec/plan where the trade-offs are genuinely open.
- Adversarially reviewing a plan, diagnosis, or change before building on it.
- A defect where the obvious explanation was tried and failed.
- A conclusion you are about to act on that would be expensive to get wrong.

**Do not use it for**: lookups or "where is X" (`scout`/`Explore`), mechanical edits
(`mech-executor`), or anything already settled. If you know the answer, write it. This is also
not a way around `AGENTS.md`'s rule against verification subagents: a deep-think runs on an
OPEN question with an unanchored brief — never to double-check reasoning you have already
finished and believe.

## The core discipline: ground it, don't anchor it

The failure mode is handing over so much of your own reasoning that the agent grades your
homework instead of thinking. The opposite failure is a brief so thin the agent rediscovers what
you already knew. Aim between them.

**Give it (grounding):**

- The **question**, stated as a question, not as a solution to validate.
- **Verified facts with `file:line`** — what the code does, what a measurement showed, exact
  revisions/run IDs. These save the agent hours and are checkable.
- **Entry points**: the files, specs, docs, and artifacts to start from.
- **What was ruled out AND how** — "X is not the cause; here is the measurement/grep that
  excluded it." Without the *how*, it is just an assertion and the agent should re-test it.
- **Constraints that are genuinely fixed**: the safety boundary, licensing posture, founder
  decisions, gates that cannot move.
- **Success criteria**: what a good answer must contain.

**Withhold (anchoring):**

- **Your preferred solution.** If you have one, say a solution exists and you are deliberately
  not stating it, so the agent can propose freely and you can compare.
- **Your hypothesis dressed as a finding.** If you have not proved it, either omit it or label it
  loudly (below).
- **Your framing of *why* it is hard** — that pre-decides where it looks.
- Long narratives of what you already tried, unless the attempt produced *evidence*. "I looked
  and did not find it" anchors without informing.

**Label everything you do pass:**

- `[MEASURED]` — you ran it; give the number and the conditions.
- `[SOURCE]` — you read it; give `file:line`.
- `[HYPOTHESIS]` — you think so; explicitly invite refutation.
- `[UNVERIFIED]` — inherited from a doc/handoff/another agent and not personally checked.
  External claims (prices, licences, provider terms) are ALWAYS this until checked at the
  primary source.

A brief with no `[HYPOTHESIS]`/`[UNVERIFIED]` labels is usually a brief that is smuggling
assumptions as facts. Re-read it before sending.

## Always include these instructions to the agent

1. **Read `AGENTS.md` first**; its rules bind you, especially the safety boundary and the
   path-trace rule.
2. **Cite `file:line` or the exact command for every claim**, including negative claims.
3. **Verify the load-bearing parts of this brief.** I may be wrong; finding that is your most
   valuable output.
4. **Disagree explicitly.** Agreement that only confirms my framing is near-worthless.
5. **Separate proved / could-not-determine / hypothesis.**
6. **Say what cheap probe would settle anything you could not close.**

## Scope and safety

- Default the agent to **read-only** for review and diagnosis. Grant `Write` only when the task
  is to author artifacts, and name the exact paths.
- Its result is a **proposal the parent verifies and owns** — design decisions, safety and
  licensing judgment, and final adjudication stay in the parent session. Before acting on it,
  re-open the cited code and check the load-bearing claims yourself. Do not relay its findings
  to the user as established fact until you have.
- Safety/licensing surfaces (advisory-only wording, taxonomy, constitution, evidence/abstention
  rules, acceptance gates and amendments, corpus posture) may be *inspected* and proposed
  against, but the parent decides every finding, and constitution-touching changes go to the
  founders, never into an edit.
- Never let it commit, push, touch corpus audio or spoken-word transcript text, or build
  anything Phase-8-gated.
- Direct children only, max four active threads.

## Runtime translation

- **Claude Code**: spawn the `deep-thinker` agent (`model: fable`, `effort: high`) — the role
  ships in this plugin (`wingman`); no repo-local setup is needed.
- **Codex**: spawn the `deep-thinker` role (`.codex/agents/deep-thinker.toml`, repo-local).
  Never invoke the `claude` CLI. Record the runtime/model actually used; never claim Codex ran
  Fable.

## Brief template

```
You are investigating <one-sentence problem> in <repo path>, branch <branch>, HEAD <sha>.
Think at high effort. MANDATE: <read-only | may write exactly these paths: ...>.

## The question
<state it as a question>

## Grounding
[SOURCE] <fact> (`file.ts:line`)
[MEASURED] <number, with the conditions it was measured under>
[UNVERIFIED] <inherited claim> — from <where>; not personally checked.
[HYPOTHESIS] <suspicion> — treat as a suspect, not a conclusion; refute it if you can.

## Ruled out, and how
<claim> — excluded by <measurement/grep and its scope>.

## Fixed constraints
<safety boundary, licensing posture, founder decisions, gates that cannot move>

## What I want back
<the specific artifacts/answers, and what a good answer must contain>

Cite file:line for every claim including negative ones. Verify the load-bearing parts of this
brief — I may be wrong, and finding that is your most valuable output. Separate what you PROVED
from what you could not determine from hypotheses. Tell me plainly what I got wrong.
```

## After it returns

1. **Verify before relaying.** Re-open the cited `file:line` for anything load-bearing. Agents
   have been confidently wrong, and have also been right where the driver was wrong — you cannot
   tell which without looking.
2. Adjudicate each finding yourself; keep or discard with a reason.
3. If it authored artifacts, check every `[UNVERIFIED]` marker before committing.
4. Relay what matters to the user, marking which parts you independently confirmed.

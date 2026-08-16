---
name: wingman-review
description: Cheap multi-agent code review of the current diff, pinned to Sonnet 5. Use after finishing a task, user story, or phase, before calling code changes done. Distinct from the hosted /code-review ultra command, which is heavier, billed, and user-triggered only.
---

# Wingman review: Sonnet-5 multi-agent diff review

A repo-local substitute for reflexively reaching for the hosted `/code-review ultra`
on every change. This skill is cheap enough to run often (every completed task or
user story is a reasonable cadence); ultra is for milestone gates (see `AGENTS.md`
"Code review" section).

## 1. Scope the diff — without loading it

Run `git status --short` and a `--stat` only. Never run the full diff in the
driver: the review agents fetch it themselves, so pulling the diff text into the
driver's context is pure waste.

- Clean tree, working on a branch: the diff command is `git diff main...HEAD`;
  size it with `git diff --stat main...HEAD`.
- Uncommitted changes present: prefer committing first and reviewing the
  committed range (`git diff <base>..HEAD`) — a commit range is an immutable
  snapshot, so every agent provably reviews identical bytes. If reviewing
  pre-commit anyway, the command is `git diff HEAD` (staged + unstaged
  together); size it with `git diff --stat HEAD`.
- Nothing in either: say so and stop; there is nothing to review.

If the stat shows a very large diff (roughly 2500+ changed lines), do not
silently truncate. Tell the user the size and either scope the diff command to
the files most central to the change (e.g. `git diff HEAD -- <paths>`) or ask
which subset to review first.

Accepted tradeoff of agents fetching the diff themselves: with a working-tree
command (`git diff HEAD`) there is no snapshot mechanism — each agent's fetch
is live, so a mid-review tree change makes agents review different code. That
is why the committed-range form is preferred, and why the tree must stay
untouched (no edits, no background jobs writing to tracked files) for the
duration of a working-tree review.

## 2. Run the multi-agent review

Claude Code sessions: call the `Workflow` tool with a script along these lines.
Every `agent()` call must set the model explicitly — an unpinned `agent()` inherits
the session model, which is the expensive failure mode this skill exists to avoid,
and the hosted ultra review does not let us pick at all. The current pin is
`model: 'sonnet'` (the alias floats to the current Sonnet generation); an A/B of
Opus-at-low-effort finders against this Sonnet pin has never been run, so treat the
pin as the cost-safe default rather than a measured optimum — the open question is
recorded in the wingman-agents repo README ("Model pins"). Pass the diff COMMAND via `args`, never the diff text: each agent runs the
command itself with its Bash tool, which keeps the whole diff out of the driver's
context. Adapt the dimension prompts to what actually changed; do not run a
dimension against a diff that has nothing relevant to it (e.g. skip the
safety-language pass on a pure test-fixture change).

```js
export const meta = {
  name: 'wingman-review',
  description: 'Sonnet-5 multi-agent review of the current diff, with adversarial verification',
  phases: [{ title: 'Review' }, { title: 'Verify' }],
}

const FINDINGS_SCHEMA = {
  type: 'object',
  properties: {
    findings: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          file: { type: 'string' },
          line: { type: 'number' },
          summary: { type: 'string' },
          failure_scenario: { type: 'string' },
        },
        required: ['file', 'summary', 'failure_scenario'],
      },
    },
  },
  required: ['findings'],
}

const VERDICT_SCHEMA = {
  type: 'object',
  properties: { refuted: { type: 'boolean' }, reason: { type: 'string' } },
  required: ['refuted'],
}

// Fail closed on a missing/garbled diff command. args HAS been observed to
// arrive as a JSON string rather than an object (2026-07-22: the first run of
// this script got a stringified args, args.diffCommand resolved to undefined,
// and every agent was told to run the literal command "undefined" — producing
// a clean-looking false-negative empty review). Normalize, then validate: a
// bad value must throw here, loudly. The metacharacter check keeps the string
// a single plain git-diff invocation — agents execute it verbatim in Bash.
let rawArgs = args
if (typeof rawArgs === 'string') {
  try { rawArgs = JSON.parse(rawArgs) } catch {
    throw new Error('wingman-review: args must be {diffCommand: "git diff ..."} — got an unparseable string')
  }
}
const diffCommand = rawArgs?.diffCommand
if (typeof diffCommand !== 'string' || !/^git diff( |$)/.test(diffCommand) || /[;&|`$<>\\\n]/.test(diffCommand)) {
  throw new Error(`wingman-review: args.diffCommand must be a single plain "git diff ..." command (no shell metacharacters), got ${JSON.stringify(diffCommand)}`)
}

const FETCH = `First run this exact command with your Bash tool and treat its full output as the diff under review (do not run any other git command or review anything outside it): ${diffCommand}\nIf that command errors or prints nothing, do NOT review anything and do NOT return an empty findings list — return exactly one finding whose summary says the diff could not be fetched, quoting the command's output verbatim.`

const DIMENSIONS = [
  {
    key: 'correctness',
    prompt: `${FETCH}\n\nReview the diff for logic bugs, wrong assumptions, and edge cases the author missed.`,
  },
  {
    key: 'security',
    prompt: `${FETCH}\n\nReview the diff for injected secrets, unsafe handling of external/user input, and unvalidated data crossing a trust boundary.`,
  },
  {
    key: 'simplification',
    prompt: `${FETCH}\n\nReview the diff for unnecessary complexity, premature abstraction, or speculative generality beyond what the change needs.`,
  },
  {
    key: 'wingman-safety',
    prompt: `${FETCH}\n\nWingman is advisory-only aviation software. Review the diff against these rules: never presents output as authoritative/dispatchable/safe-to-rely-on or tells a pilot what to do; never lets a model invent or silently repair numbers, callsigns, identifiers, timestamps, or quoted operational facts; every user-visible claim resolves to retained evidence and freshness metadata, abstaining when support is absent, stale, conflicting, or weak; only uses data within its licensed access/reuse rights; never probes, evades, or bypasses an airline network or access control. Flag any violation.`,
  },
  {
    key: 'test-coverage',
    prompt: `${FETCH}\n\nReview the diff for new logic that lacks tests, or existing tests weakened/removed without replacement.`,
  },
]

const results = await pipeline(
  DIMENSIONS,
  (d) => agent(d.prompt, { label: `review:${d.key}`, phase: 'Review', schema: FINDINGS_SCHEMA, model: 'sonnet' }),
  (review) =>
    parallel(
      (review?.findings ?? []).map((f) => () =>
        agent(
          `${FETCH}\n\nThen try to refute this code review finding against the diff. Default to refuted=true if you cannot confirm it from the diff alone.\n\nFinding: ${JSON.stringify(f)}`,
          { label: `verify:${f.file}`, phase: 'Verify', schema: VERDICT_SCHEMA, model: 'sonnet' },
        ).then((v) => ({ ...f, verdict: v })),
      ),
    ),
)

const confirmed = results.flat().filter(Boolean).filter((f) => f.verdict && !f.verdict.refuted)
return { confirmed }
```

Pass the diff command via `args: { diffCommand: "<the command chosen in step 1>" }`
— the command string, not the diff text. The script's guard makes a missing or
non-`git diff` command fail the whole workflow immediately; if that happens, fix
the `args` you passed rather than editing the guard.

## 3. Report

Report the returned `confirmed` findings with `ReportFindings`, most severe first
(empty list if nothing survived verification). Do not also restate them as prose.

Then record that the review ran, so the pre-push gate can see it (machine-local,
inside `.git/`, never committed):

```sh
printf '%s %s\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$(git rev-parse HEAD)" > .git/wingman-review-last
```

Record it only after the review actually completed — never touch this file to
silence the gate's warning.

## Codex sessions

Codex has no `Workflow` tool equivalent. Perform a single thorough manual pass over
the same diff and the same five dimensions above yourself, and report findings as
plain text — no subagent fan-out, no Sonnet pinning (not applicable to Codex).

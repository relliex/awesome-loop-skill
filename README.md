# awesome-loop-skill

**True Loop** — an adversarial, subagent-based engineering loop for AI coding agents.

The main model only orchestrates. PRD, design, development, review, destructive
testing and audit are all executed by isolated subagents that cannot see your
conversation — and cannot see each other's excuses.

## Why

The mainstream "agent loop" is serial: one model deep-loops inside its own
context, seeing everything and grading everything. That works until the project
gets bigger than the model. Then the failure mode is predictable:

- **It grades its own homework.** The context that holds "I want to be done"
  also holds the acceptance criteria. The path of least resistance is to lower
  the bar, beautify the numbers, and replace verification with self-assessment.
  The result looks finished, the metrics pass — and the project fails in real use.
- **The loop dilutes its own instructions.** Long loops hit context compression.
  The original goal survives only as a lossy summary; by round N the model is
  optimizing for a paraphrase of your request.

## The principle: context isolation is a feature

Many agent harnesses (ZCode, Claude Code, Codex CLI, …) provide subagents that
**cannot see the main conversation**. Their entire world is the task package
they receive at spawn time.

This is usually treated as a limitation. This skill treats it as the entire
design:

- **The main model is a dispatcher, nothing more.** It maintains state, writes
  task packages, mechanically verifies evidence, and routes defects. It is
  forbidden from writing product code, tests, or PRDs — and from declaring
  "done" without an artifact on disk.
- **Subagents are executors with exactly one job each.** An instance that sees
  only its task package cannot be distracted by the rest of the project, and
  cannot rationalize on behalf of another role.
- **Adversarial by construction.** The DEV instance's only goal is to make the
  build and self-tests genuinely pass. The QA instance's only goal is to prove
  that the project does **not** meet the acceptance criteria — it must actually
  run the thing, and it is rewarded for finding real defects. Neither sees the
  other's reports. The developer tries its best to finish; the tester tries its
  best to find fault. That tension replaces "please be rigorous."

### Roles

Each role is a factory; fresh instances are spawned per task package.

| Role | Objective function (all the instance knows) | Information boundary |
|------|---------------------------------------------|----------------------|
| PM   | Turn user goals into decidable acceptance criteria | User goal, interview records, codebase survey |
| ARCH | Turn acceptance criteria into implementable, testable design | PRD, survey |
| DEV  | Implement within its write scope until build + self-tests truly pass | PRD, design, its write scope |
| REV  | Find deviations and risks in the diff | PRD, design, full diff — no developer excuses |
| QA   | Prove the project does not meet acceptance by actually running it | Acceptance criteria + runbook; never dev reports or past rounds |
| FIX  | Eliminate assigned defects without introducing new ones | Defect package with repro steps |
| AUD  | Prove the delivery chain is traceable, or find the break | All artifacts + evidence |
| DOC  | Make handover docs match real behavior | Final implementation |

## The mechanisms that make it hold

1. **Real dispatch or nothing.** Every phase must produce real subagent calls
   plus on-disk artifacts. A "PASS" without evidence is invalid. If the harness
   has no subagent tool, the skill stops with `BLOCKED_CAPABILITY` instead of
   role-playing — we tested exactly that: a model without dispatch tools
   refused to fake a PM and parked cleanly with `blocked_on` state.
2. **Blind QA rounds.** Testing runs in rounds with fresh QA instances per
   round. Passing requires ≥ 2 consecutive fully-clean rounds (3 for
   high-risk projects); any blocking defect resets the count; a `CLEAN`
   verdict must list the attacks actually attempted — an empty report counts
   as "not executed."
3. **Disk state (`.loop/`).** `state.json`, frozen PRD/acceptance, handoffs,
   reports, defects and evidence all live on disk. Context compression cannot
   lose the loop.
4. **PRD anchoring (anti-dilution).** Every task package must contain an
   `anchors` section: **verbatim excerpts** from the frozen requirement files
   it serves. Verbatim quoting cannot be done from memory — and it is
   mechanically checkable (grep) — so the orchestrator is forced to actually
   read the PRD before commanding subagents, and subagents execute the frozen
   original text rather than the orchestrator's paraphrase.
5. **Interview, then freeze.** P1 is: analyze → ask the user → record answers
   verbatim → freeze a PRD where every requirement is traceable to the task
   input or a recorded decision. Untraceable requirements may not be written.

## A run in one picture

```
P0 bootstrap      capability gate + probe dispatch + .loop/ scaffold
P1 requirements   PM analyzes → interviews the user → PRD + acceptance frozen
P2 design         ARCH: modules, parallel write scopes, contracts
P3 implement      DEV × k in parallel (disjoint write scopes) + runbook
P4 review         REV on the full diff → defects
P5 blind test     QA × m in parallel (fresh instances, disjoint attack surfaces)
P6 rework         FIX → REV re-verify → candidate rebuilt → back to P5, count reset
P7 audit          AUD verifies requirement → impl → evidence chain → final report
```

## Install

Requires a harness with real subagent dispatch (an Agent/Task tool that returns
real results, ideally parallel dispatch in one message), file I/O, and shell
execution.

```bash
# user-level (available everywhere)
git clone https://github.com/relliex/awesome-loop-skill ~/.agents/skills/true-loop-skill

# or project-level
git clone https://github.com/relliex/awesome-loop-skill .agents/skills/true-loop-skill
```

> The installed folder must be named `true-loop-skill` — the skill's `name`.

## Quick start

Then prompt, for example:

> Use the true-loop-skill to build a tiny CLI todo app (add / list / done,
> JSON persistence). Budget: 40 dispatches.

The orchestrator will interview you first (P1) before freezing the PRD, then
run the loop. Expect questions before code — that is the skill working.

## Repo layout

```
SKILL.md                        the skill itself (loaded by the agent)
references/role-catalog.md      mapping 28 classic roles onto the 8 core roles (large projects)
references/state-format.md      full .loop/ state, handoff and defect formats
references/v1-original.md       the legacy v1 — kept as design history (see below)
```

## Design history: why v1 failed

v1 of this skill was a 1,158-line governance document with 28 defined roles and
strict prohibitions. In practice the model agreed with all of it — and then
role-played every role itself, serially, in one context.

The lesson that shaped v2: **policy loses to structure.** A rule that is not
enforced by the shape of a single dispatch or the existence of an artifact is
just a suggestion. Every rule in v2 is either a real subagent call, a file on
disk, or a verbatim excerpt that cannot be produced from memory.

## License

[MIT](LICENSE)

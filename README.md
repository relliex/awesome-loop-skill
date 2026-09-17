# awesome-loop-skill

**True Loop** — a domain-agnostic, adversarial delivery loop for AI agents.

The main model only orchestrates. Requirements are clarified with you, a team
of experts is generated for your project, and the actual work — writing,
coding, research, design, analysis — is executed by isolated subagents that
cannot see your conversation and cannot see each other's excuses.

## Why

The mainstream "agent loop" is serial: one model deep-loops inside its own
context, seeing everything and grading everything. That works until the
project gets bigger than the model. Then the failure mode is predictable:

- **It grades its own homework.** The context that holds "I want to be done"
  also holds the acceptance criteria. The path of least resistance is to lower
  the bar, beautify the numbers, and replace verification with self-assessment.
  The result looks finished, the metrics pass — and the deliverable fails in
  real use.
- **The loop dilutes its own instructions.** Long loops hit context
  compression. The original goal survives only as a lossy summary; by round N
  the model is optimizing for a paraphrase of your request.

## The principle: context isolation is a feature

Many agent harnesses provide subagents that **cannot see the main
conversation**. Their entire world is the task package they receive at spawn
time.

This is usually treated as a limitation. This skill treats it as the entire
design:

- **The main model is a dispatcher, nothing more.** It maintains state, packs
  task packages, mechanically verifies evidence, and routes problems. It is
  forbidden from producing deliverables — no code, no copy, no plans, no tests.
- **Subagents are executors with exactly one job each.** An instance that sees
  only its task package cannot be distracted by the rest of the project and
  cannot rationalize on behalf of another role.
- **Adversarial by construction.** The producer's only goal is to make its
  artifact pass the frozen criteria. The adversarial verifier's only goal is
  to **prove it does not** — it must actually exercise the artifact, and it is
  rewarded for finding real problems. They never see each other's reports.

## Dynamic teams, not a fixed org chart

The skill ships no fixed role catalog as its backbone. It ships a
**role-generation method** (SKILL.md §2) and **teaching cases** from eight
domains ([references/role-design.md](references/role-design.md)), plus one
software-domain appendix that is never a prerequisite. Every project grows
its own team, built from three archetypes:

| Archetype | Objective function | Information boundary |
|-----------|--------------------|----------------------|
| Producer | Produce the assigned artifact so it passes the frozen criteria | Criteria, source material, own write scope |
| Adversarial verifier | Prove the assigned artifact fails, by actually exercising it | Artifact, criteria, real verification means; never the producer's self-assessment |
| Arbiter / auditor | Define and freeze criteria; verify the traceability chain | All artifacts and evidence |

Generation is followed by a gate checklist (every deliverable has a producer,
every critical deliverable has an information-isolated adversary with means
capable of failing it, criteria are defined by a third party, no
"everything-expert" roles).

Domain examples included: software/systems, long-form writing, research & due
diligence, design/UX, operations & campaigns, data/ML, translation &
localization, curriculum design — each with a generated roster, adversarial
pairings, and a full prompt template.

## The flow

```
G1  clarify      main model analyzes requirement completeness
                 → multi-round user selector until no blocking gaps
G2  plan         plan role writes the plan (prd.md) + acceptance criteria
                 → "do you accept this plan?" via selector
G3  team         roles generated from the plan + teaching cases → team.md
                 → roster shown to you
                 → "may I assemble this team?" via selector
RUN              main model orchestrates; isolated subagents produce,
                 adversarially verify in blind rounds, and rework
                 until criteria genuinely pass — then independent audit
```

Two things are deliberately non-negotiable: the model asks **before** it
builds your plan, and it asks **before** it builds your team. Vague approvals
("sure, whatever") do not count as approval.

## Mechanisms that make it hold

1. **Real dispatch or nothing.** Every phase must produce real subagent calls
   plus on-disk artifacts. A "PASS" without evidence is invalid. If the
   harness has no subagent tool, the skill stops with `BLOCKED_CAPABILITY`
   instead of role-playing — we tested exactly that: a model without dispatch
   tools refused to fake a role and parked cleanly. (A serial self-check
   fallback exists only with your explicit consent; it is labeled
   "unverified" and "self-assessment risk", ships per-criterion steps you can
   re-run yourself, and is barred for high-risk work.)
2. **Blind adversarial rounds.** Verification runs in rounds with fresh
   instances. Passing requires ≥ 2 consecutive fully-clean rounds (3 for
   high-risk work); any blocking problem resets the count; "no problems found"
   must list what was actually attempted — an empty report counts as
   "not executed."
3. **Disk state (`.loop/`).** `state.json`, plan, acceptance criteria
   `team.md`, handoffs, reports, issues and evidence all live on disk.
   Context compression cannot lose the loop.
4. **Anchoring against dilution.** Every task package must contain an
   `anchors` section: **verbatim excerpts** from the frozen files it serves.
   Verbatim quoting cannot be done from memory — and it is mechanically
   checkable (grep) — so the orchestrator is forced to actually read the plan
   before commanding subagents, and subagents execute the frozen original
   text rather than the orchestrator's paraphrase.
5. **Verbatim decision records.** Every selector answer is stored word for
   word, so you can later audit what was delivered against what you actually
   agreed to.

## Install

Requires a harness with real subagent dispatch (an Agent/Task tool returning
real results, ideally parallel dispatch in one message), file I/O, shell
execution, and a structured question tool for the user gates.

```bash
# user-level (available everywhere)
git clone https://github.com/relliex/awesome-loop-skill ~/.agents/skills/true-loop-skill

# or project-level
git clone https://github.com/relliex/awesome-loop-skill .agents/skills/true-loop-skill
```

> The installed folder must be named `true-loop-skill` — the skill's `name`.

## Quick start

Then prompt, for example:

> Use the true-loop-skill to write a 6000-word technical explainer on
> retrieval-augmented generation for backend engineers. Budget: 40 dispatches.

or

> Use the true-loop-skill to build a tiny CLI todo app (add / list / done,
> JSON persistence). Budget: 40 dispatches.

Expect to be interviewed first (G1), asked to approve a plan (G2), and asked
to approve a generated team (G3) before any deliverable work starts. That is
the skill working as designed.

## Repo layout

```
SKILL.md                        the skill itself (loaded by the agent)
references/role-design.md       role-generation method + 8 domain teaching cases
references/state-format.md      full .loop/ state, team.md, handoff, issue formats
references/role-catalog.md      software-domain appendix: 28 classic roles mapped
references/v1-original.md       the legacy v1 — kept as design history
docs/upgrade-to-v3-requirements.md   the requirements doc driving the v3 redesign
docs/adversarial-review-v3.md        independent adversarial review of this upgrade
```

## Design history

- **v1** was a 1,158-line governance document with 28 predefined roles. The
  model agreed with all of it — and then role-played every role itself,
  serially, in one context.
- **v2** replaced policy with structure: isolated subagents, blind adversarial
  QA rounds, disk state, and PRD anchoring. Every rule is a real subagent
  call, a file on disk, or a verbatim excerpt that cannot be produced from
  memory. But v2 was still software-shaped, with a fixed 8-role cast.
- **v3** (current) makes it domain-agnostic: the model **generates** the team
  for each project from a teaching method plus cross-domain cases, and three
  user gates (clarify → plan → team) put you in control before any work
  starts.

The lesson that shaped v2 and still holds: **policy loses to structure.** A
rule that is not enforced by the shape of a single dispatch or the existence
of an artifact is just a suggestion.

## License

[MIT](LICENSE)

---
name: implement-plan
description: >-
  Use when the user wants to implement, execute, or work through an existing
  plan with a team of agents — "implement the plan", "execute milestone 2",
  "build these user stories", "work through plan/user-stories.md". Builds an
  agent team in which the team lead only delegates (it never writes code):
  backend/frontend dev personas implement stories in isolated git worktrees,
  and every story passes verification gates (acceptance criteria checked
  semantically, architecture conformance, optional performance/security/
  compliance review of the implemented change). Also use to smoke-test the
  agent-team setup locally, or to resume a started implementation (a
  plan/implementation-state.md exists). Not for drafting plans
  (draft-plan-with-docs) or for reviewing existing code or architecture
  without a plan to execute.
---

# Implement Plan

Executes an existing plan with an agent team. The session that invokes this
skill becomes the **team lead**. `<skill-dir>` below is this skill's base
directory — pass file paths from it to teammates as absolute paths, because
teammates don't inherit your skill context.

## Principles

- **The lead delegates — it never implements.** You do not edit files, write
  code, or run build/git commands. Your tools are team, task, messaging, and
  read tools; every mutation (worktrees, commits, merges, even preflight) is
  delegated. Two reasons: your context must stay small enough to orchestrate
  the whole plan, and the judge of "done" must not be the author of the work.
  The single exception: you write `plan/implementation-state.md` (your own
  bookkeeping, see [Handoff.md](Handoff.md)).
- **Acceptance criteria are the contract.** A task is completed when its
  verification gates pass — never because an implementer reports it done.
  Only the lead marks tasks completed.
- **Fresh contexts beat big contexts.** One teammate per story by default;
  gates get fresh reviewers per story. Anyone approaching ~200k tokens hands
  off per [Handoff.md](Handoff.md) instead of degrading.
- **Docs are read-only input.** Follow their terminology; report deviations
  instead of patching docs (changing docs is `draft-docs`' job, changing the
  plan is `draft-plan-with-docs`' job).

## Input

| What  | Default                                            | Notes                                          |
| ----- | -------------------------------------------------- | ---------------------------------------------- |
| Plan  | `plan/user-stories.md`                              | any plan file or a pasted plan works           |
| Scope | the whole plan                                      | or one milestone / named stories, per request  |
| Docs  | `Architecture.md`, `DomainModel.md`, `plan/design.md` | context for personas and gates              |

Stories with `Test:` / `CI:` / `Check:`-prefixed acceptance criteria (the
`draft-plan-with-docs` format) are used verbatim. For plan items without
acceptance criteria, derive minimal ones and list them in the kickoff report
so the user can object before work starts.

`<plan-slug>` below: kebab-case of the plan file name, plus the scope when
narrowed (`user-stories`, `user-stories-m2`). It's recorded in the state
file — on resume, the recorded slug wins. All branches live under
`implement/<plan-slug>/…`, so that prefix must never itself be a branch.

## Workflow

1. **Preflight** — spawn one subagent to run [Preflight.md](Preflight.md). It
   verifies git worktrees actually work, establishes the build/test baseline,
   checks the plan, and creates the integration branch + worktree. On FAIL,
   stop and report the failed checks — don't build a team on a broken base.
2. **Plan → tasks** — `TeamCreate` (team `implement-<plan-slug>`), then one
   `TaskCreate` per story in scope. Each description carries: story ID and
   title, acceptance criteria **verbatim**, links to the design/doc sections
   it implements, the chosen persona, and which optional gates apply (table
   below). Encode ordering with `addBlockedBy` (story order, shared
   contracts, backend-before-frontend on the same feature).
3. **Build the team** — spawn implementers with the Agent tool
   (`team_name`, `name`, `run_in_background: true`) using the spawn template
   below. Default 2 parallel implementers, max 3 — and only on tasks that are
   unblocked and unlikely to touch the same files. Sequential work needs no
   parallelism: one teammate at a time is fine.
4. **Delegate & gate** — assign a task (`TaskUpdate` owner + `SendMessage`),
   wait for the implementer's report, then run the gates (below). Findings go
   back to the same implementer; after a pass, the implementer merges and you
   mark the task completed. Rules that keep this loop sound:
   - **Merges are serialized.** Authorize one merge at a time and wait for
     its report before authorizing the next — two agents merging in the
     shared integration worktree at once corrupt it. If an implementer had to
     resolve non-trivial conflicts, send the merged result through the
     Verifier once more.
   - **Silent teammates.** An idle notification without the expected report →
     nudge once via `SendMessage`; still silent on its next idle → shut it
     down and spawn a successor ([Handoff.md](Handoff.md)).
   - **Three failed gate rounds** on one story → shut down its implementer,
     record the worktree and branch as parked in the state file, report, and
     move on.
5. **Checkpoint** — after every completed story, update
   `plan/implementation-state.md` ([Handoff.md](Handoff.md)). This is what
   makes compaction and resumption safe at any moment.
6. **Report & teardown** — when scope is done: send every remaining teammate
   a `shutdown_request`, wait for the confirmations (`TeamDelete` fails while
   members are active), then `TeamDelete`. Mark the state file
   `Complete — ready for PR` at the top. Report per story: gate verdicts,
   deviations from plan/docs, doc gaps the Architect found — and that
   `implement/<plan-slug>/integration` is ready for the user to review/PR.
   The integration worktree stays behind for exactly that; say where it is.
   You never merge into the user's branch.

## Spawn template (implementers)

> You are "<name>" on team "<team>".
> 1. Read `<skill-dir>/personas/<Persona>.md` — your role and boundaries.
> 2. Read `<skill-dir>/Handoff.md`, section "Teammate handoff".
> 3. Your task is #<id> "<subject>" — run TaskGet for the story and
>    acceptance criteria, and set its status to in_progress (you are already
>    its owner).
> 4. Create your worktree:
>    `git worktree add <worktrees-dir>/<story-id> -b implement/<plan-slug>/<story-id> implement/<plan-slug>/integration`
>    and work only inside it.
> 5. Baseline: build `<cmd>` · test `<cmd>` · pre-existing failures: <list
>    from preflight>.
> 6. When done, report to "team-lead" via SendMessage (format in your persona
>    file). Do not mark the task completed — the lead does that after
>    verification.

`<worktrees-dir>` is `<repo-parent>/<repo-name>.worktrees` (created by
preflight). Personas are archetypes: for stacks without a frontend/backend
split (CLI, infra, library), pick the closer persona and say so in the spawn
prompt — a CLI tool is "backend".

## Verification gates

Spawn gate reviewers **fresh per story and per round** (fresh eyes are the
point), in parallel, as background teammates with this template:

> You are "<gate>-<story-id>-r<round>" on team "<team>".
> 1. Read `<skill-dir>/personas/<Gate>.md`.
> 2. Verify story <story-id> (task #<id>): worktree `<path>`, branch
>    `<branch>`, integration branch `implement/<plan-slug>/integration`.
> 3. Baseline: build `<cmd>` · test `<cmd>` · pre-existing failures: <list
>    from preflight>.
> 4. The implementer reports: <report>. Treat it as a claim, not evidence.
> 5. Send your verdict to "team-lead" via SendMessage, then you are done.

| Gate                                     | Runs                                                              |
| ---------------------------------------- | ----------------------------------------------------------------- |
| [Verifier](personas/Verifier.md)         | always — semantic check of every acceptance criterion             |
| [Architect](personas/Architect.md)       | always — change vs. `Architecture.md` / `DomainModel.md`          |
| [Performance](personas/Performance.md)   | docs/design state budgets or NFRs, or the story touches a hot path or bulk data |
| [Security](personas/Security.md)         | story touches authn/authz, input parsing, secrets, PII, new endpoints or dependencies |
| [Compliance](personas/Compliance.md)     | docs carry regulatory requirements (audit, retention, licensing, residency) |

…or whenever the user asks for a gate. Decide the optional gates per task in
step 2, from the plan and docs — record the decision in the task description.

A story passes when **all** its gates pass. Collect all verdicts before
sending findings back, so the implementer fixes everything in one round.
After collecting a verdict, shut that reviewer down (`shutdown_request`) —
reviewers never carry over between rounds or stories. A new round re-runs
**all** the story's gates, not just the failed ones: a fix can break what
previously passed.

## Context budget (the lead's own)

Keep your context lean enough to survive the whole plan: never read diffs or
source files — consume reports and verdicts; don't re-read the plan per story
— the task descriptions carry what each story needs. When your context
approaches **200k tokens** (or the harness warns about auto-compaction), run
the lead handoff in [Handoff.md](Handoff.md) before continuing.

## Smoke test

When asked to smoke-test the setup ("test implement-plan locally", "check the
agent team works"), prove the machinery without a real plan. The slug is
`smoke`; skip the state file. Run preflight (minus the plan check — there is
no plan), create team `implement-smoke`,
spawn one implementer whose task is a no-op story ("add a scratch file" with
one `Check:` criterion) in a worktree, gate it with a fresh Verifier, let the
implementer merge — then delegate teardown to a cleanup subagent: integration
worktree and branch (`implement/smoke/integration`), any leftover story
worktree or branch, `git worktree prune`. Shut down the teammates,
`TeamDelete`, and report the checklist: worktrees ✓, team ✓, messaging ✓,
task list ✓, gates ✓, merge ✓, cleanup ✓. Nothing of the smoke test survives.

## Resume

If `plan/implementation-state.md` exists and the user asks to continue: read
it, run `TaskList` for its team (task lists persist in
`~/.claude/tasks/<team>/`), re-run preflight only if the environment may have
changed, respawn teammates for in-progress tasks (their worktrees and
branches are in the state file), and continue at step 4. A state file marked
`Complete` is not resumable — tell the user the integration branch is ready
instead.

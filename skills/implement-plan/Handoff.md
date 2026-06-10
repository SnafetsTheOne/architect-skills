# Handoff

Reference for `implement-plan` context management. Contexts that grow past
~200k tokens get slow, expensive, and forgetful — nobody on the team works in
one. The cure is cheap because state lives **outside** the context: the team
task list persists on disk (`~/.claude/tasks/<team>/`), git holds the work,
and the state file holds everything else.

## The state file — `plan/implementation-state.md`

The only file the lead ever writes. Created after preflight, updated after
**every** completed or blocked story — not only when a handoff looms, so a
handoff (or an unplanned compaction) never loses more than the story in
flight.

```markdown
# Implementation State — <plan> (<scope>)

Skill: architect:implement-plan (re-invoke to reload instructions)
Team: <team-name> · Integration branch: implement/<plan-slug>/integration
Worktrees dir: <path> · Baseline: build `<cmd>` · test `<cmd>`

## Preflight
<the preflight report, verbatim>

## Stories
| Task | Story | Status | Owner | Branch | Gates passed |
|---|---|---|---|---|---|
| #3 | M1-2 | merged | — | implement/<slug>/M1-2 | Verifier, Architect |
| #4 | M1-3 | in gate round 2/3 | backend-1 | implement/<slug>/M1-3 | — |

## Decisions & deviations
- <dated, one line each: contract conflicts resolved, criteria derived, doc gaps found>

## Next actions
- <what the lead would do next, concrete enough for a successor>
```

## Lead handoff

When the lead's context approaches 200k tokens, or the harness warns about
auto-compaction:

1. Bring the task list up to date (statuses, owners, blockers).
2. Update the state file — especially **Next actions** and anything decided
   since the last checkpoint that a successor couldn't reconstruct.
3. Let compaction happen (or trigger it). Afterwards — or in a brand-new
   session — the resume path is the same: re-invoke `implement-plan`, read
   the state file, `TaskList`, continue. Running teammates survive; they're
   still addressable by name.

## Teammate handoff

For a teammate (implementer or gate) whose context nears 200k mid-task:

1. Finish the current coherent step — don't stop mid-edit.
2. Commit work in progress in your worktree:
   `wip(<story-id>): handoff — <one line on what remains>`.
3. Append a handoff note to your task (`TaskUpdate` description): what is
   done, what remains, gotchas discovered, files in flight.
4. `SendMessage` the team lead that you're handing off, then go idle and
   approve the shutdown request when it arrives.

The lead then spawns a successor with the same persona, pointed at the same
task, worktree, and branch — the handoff note plus `git log`/`git diff` in
the worktree is the successor's starting context. One task should rarely need
this; if a single story burns through two contexts, tell the lead it's too
big and should be split (the lead reports that rather than splitting the plan
itself — plans change via `draft-plan-with-docs`).

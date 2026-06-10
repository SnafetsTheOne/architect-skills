# Preflight

Reference for step 1 of `implement-plan`. The **lead delegates this whole
file to one subagent** (general-purpose, not read-only — it mutates git
state). The subagent runs every check, performs the setup at the end, and
returns the report. The lead aborts on any FAIL.

Before spawning: the lead itself confirms the team tools (`TeamCreate`,
`SendMessage`, `TaskCreate`) are available in its own tool list. If they are
not, stop immediately and tell the user agent teams aren't enabled in this
Claude Code setup — nothing else in this skill works without them.

## Checks

Run in order; keep going after a failure so the report is complete.

1. **Git repository** — `git rev-parse --show-toplevel` succeeds. Record the
   repo root, current branch, and current commit.
2. **Clean working tree** — `git status --porcelain` is empty, ignoring
   `plan/implementation-state.md` (the skill's own bookkeeping). Other
   uncommitted changes are a FAIL: implementers must branch from a defined
   state, and the user's half-done work must not leak into story branches.
   The fix is the user's call (commit or stash) — never do it for them.
3. **Worktree round-trip** — prove worktrees work here (filesystem,
   permissions, git version), don't assume. Use a **nested** branch name so
   the run also proves the `implement/<plan-slug>/…` naming scheme:
   - `git worktree add <worktrees-dir>/preflight -b implement/preflight/round-trip`
   - in that worktree: write a scratch file, `git add`, `git commit`
   - `git worktree remove --force <worktrees-dir>/preflight`
   - `git branch -D implement/preflight/round-trip`, `git worktree prune`
   - `git worktree list` shows no leftovers
   `<worktrees-dir>` is `<repo-parent>/<repo-name>.worktrees`; if the repo's
   parent isn't writable, fall back to the OS temp dir and record the chosen
   path in the report — every later worktree uses it.
4. **Build/test baseline** — detect the build and test commands (in order:
   `CLAUDE.md`/`AGENTS.md`, `README`, then the manifests — `*.csproj`/`*.sln`,
   `package.json`, `pom.xml`, …) and run them. Record the commands and the
   result. Pre-existing failures are not a FAIL — list them verbatim, so
   gates later don't blame implementers for them. Finding **no** way to build
   or test is a FAIL: gates would have nothing to verify with.
5. **Plan** — the plan file exists (or the pasted plan was provided), the
   in-scope items are identifiable, and their links to `plan/design.md` /
   docs resolve. Items without acceptance criteria are not a FAIL — list them
   so the lead derives criteria visibly.

## Setup (only if all checks pass)

- Create the integration branch from the current commit:
  `git branch implement/<plan-slug>/integration`. Story branches will live
  beside it as `implement/<plan-slug>/<story-id>` — that's why the
  integration branch carries the `/integration` suffix: the bare prefix
  `implement/<plan-slug>` must never be a branch, or git refuses every
  story branch under it.
- Create the persistent integration worktree:
  `git worktree add <worktrees-dir>/integration implement/<plan-slug>/integration`.
  Story branches are merged **there** (the user's checkout never switches
  branches); implementers run their post-merge build in it.
- Add `plan/implementation-state.md` to `.git/info/exclude` so the lead's
  state file never dirties the tree or sneaks into a commit.

## Report

Return exactly this shape — the lead stores it in the state file:

```markdown
## Preflight — PASS | FAIL
| Check | Result | Detail |
|---|---|---|
| Git repository | PASS | <root>, <branch>@<commit> |
| Clean working tree | PASS/FAIL | <leftover files if any> |
| Worktree round-trip | PASS/FAIL | <worktrees-dir> |
| Build/test baseline | PASS/FAIL | build: `<cmd>` · test: `<cmd>` |
| Plan | PASS/FAIL | <plan file, scope, items found> |

Pre-existing test failures: <none | verbatim list>
Items without acceptance criteria: <none | story IDs>
Integration branch: implement/<plan-slug>/integration · worktree: <path>
```

# User Story Format

Reference for the **user-stories level** of `draft-plan-with-docs`. Stories are
written into `plan/user-stories.md`. Keep the format below exactly.

## Story Format

Every story has a fixed opener and a fixed closing section. Optional sections go between the two:

    #### <ID> — <Short title>

    **As a** <role>
    **I want** <goal>
    **so that** <benefit>.

    [optional sections — see below]

    **Acceptance Criteria:**
    - <concrete, testable criterion>
    - …

Rules:
- **"As a", "I want", "so that" are always on separate lines**, in that order, at the top of every story — no exceptions.
- One role, one goal, one benefit per story.
- Link to related docs or domain entities where relevant.

### Acceptance Criteria rules

- **As few as possible** — only the criteria needed to prove the story is done.
- **Short natural sentences.** Prefer "When [action], [outcome]." No bullet-point checklists of sub-features.
- **Every criterion carries a prefix** that says how to verify it:

  | Prefix | Meaning | Work implied |
  |--------|---------|-------------|
  | `Test:` | Automated test in test code | Write a Playwright / JUnit / integration test |
  | `CI:` | A pipeline command must pass | Add a CI step (`helm lint`, `docker build`, etc.) |
  | `Check:` | Manual verification | Inspect logs, output, or status — no automation implied |

- Prefer `Test:`. Use `CI:` for infrastructure/tooling verifications. Use `Check:` only when automation is genuinely not practical.

## Optional Sections — contracts

A story may touch technical contracts: a database table, a data contract (API
route / request / response / query params), an event or stream schema, a page, or
config.

**Prefer linking over embedding.** When a `plan/design.md` exists (the Design
level), those contracts live **there**. The story links to the relevant section
instead of re-embedding it:

    **Design:** plan/design.md#incident-tables, plan/design.md#post-incidents

**Embed inline only as a fallback** when no Design level was drafted — and even
then, use the same formats, defined once in **[Design.md](Design.md)**. Never both.

    **Table Schema:** / **Data Contract:** / **API Endpoints:** / **Config Parameters:**
    …per [Design.md](Design.md)…

## IDs

IDs are a milestone prefix and a running number scoped to that milestone: `M1-1`, `M1-2`, `M2-1`, `M2-2`, etc.

- Numbers increment sequentially within a milestone across all epics — no gaps, no reuse.
- Story type (user-facing vs. developer) is communicated by the epic/theme heading, not the ID.

## Document Structure

Group stories by milestone using headings; within a milestone, optionally group by theme/epic. Keep the document scannable:

    ## Milestone N — <Title>

    ---

    ### <Epic or theme>

    #### M1-1 — <Short title>

    **As a** <role>
    …

If no roadmap level was drafted, still group stories under milestone headings you define inline (the higher level is not required to exist as its own file).

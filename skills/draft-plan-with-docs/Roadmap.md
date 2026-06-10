# Roadmap Format

Reference for the **roadmap level** of `draft-plan-with-docs`, written into
`plan/roadmap.md`.

## Milestone Format

The roadmap is a list of milestones in delivery order. One section per milestone:

    ## Milestone N — <Title>

    **Goal:** <one sentence: the outcome this milestone delivers>

    **Scope:**
    - <theme / epic / capability included>
    - …

    **Depends on:** <prior milestone(s), or "—">

Rules:

- **Ordered & numbered.** `Milestone N` is numbered by delivery order; `Depends on`
  names the milestone(s) that must ship first (or `—` for none).
- **Outcome, not output.** The Goal states what becomes possible, not the tasks.
- **Scope is themes, not stories.** List capabilities/epics; per-story detail
  belongs to the user-stories level, the contracts to the design level.
- **No dates.** Sequence and dependencies only — a roadmap of order, not a schedule.

## What belongs here vs. elsewhere

| Put here                                               | Put elsewhere                                                    |
| ------------------------------------------------------ | --------------------------------------------------------------- |
| Milestone goals, scope themes, ordering, dependencies  | Story-level detail → [UserStories.md](UserStories.md)           |
|                                                        | Technical contracts (tables, APIs, schemas) → [Design.md](Design.md) |

## IDs

Milestones are numbered `1, 2, 3, …` in delivery order. Story IDs reference the
milestone number (`M1-1`, `M2-3`) — see the ID scheme in
**[UserStories.md](UserStories.md)**.

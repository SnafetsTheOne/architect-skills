# Requirements.md — guidance

Guidance for maintaining a project's `Requirements.md` — the *what* and *why*. Not the
*how* (→ `Architecture.md`), and not acceptance criteria / milestones / story
breakdowns (→ the plan / user stories). *(Meta-guidance for the skill, not an actual
requirements doc.)*

Per header, deltas from the standard workflow in
[SKILL.md](SKILL.md#workflow-for-every-change) on **create / edit / delete** — plus a
**Visualize** default for sections worth drawing.

## Vision

One short paragraph: what the product is for, and for whom.

- **Create** — once, at doc creation.

> A web/mobile app that helps a person track personal spending: record expenses, group
> them by category, and see spend against monthly budgets.

## Functional Requirements

One heading per capability — what a user can do and why.

- **Create** — add a `### Capability` heading; describe **behaviour**, not UI or tech.
  Link domain terms to `DomainModel.md`, entity detail to `entities/`.
- **Delete** — retiring a capability: remove it, then check whether any entity it was
  the *sole* reason for is now orphaned (→ `DomainModel.md` delete).
- **Visualize** — User Journey / capability map: the capabilities as steps a user walks
  through.

> ### Record an expense
>
> The user records an [Expense](DomainModel.md) with an amount, a date and a
> [Category](DomainModel.md).

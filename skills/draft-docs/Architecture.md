# Architecture.md — guidance

Guidance for maintaining a project's `Architecture.md` — the *how*. Term meanings live
in `DomainModel.md`; capabilities live in `Requirements.md` — link, don't restate.
*(Meta-guidance for the skill, not an actual architecture doc.)*

Per header, deltas from the standard workflow in
[SKILL.md](SKILL.md#workflow-for-every-change) on **create / edit / delete** — plus a
**Visualize** default for sections worth drawing.

## Context

External systems the app interacts with, plus the app itself.

- **Create** — the system + what it does / what data it owns. An external system that
  *owns* domain data is usually also a **Bounded Context** (`DomainModel.md`).
- **Delete** — removing an integration: also drop its Data Flow steps and any Module
  that existed only for it.
- **Visualize** — C4 L1 – System Context: the app and the external systems around it.

> **Expense Tracker** — this app. Lets a User record expenses and track budgets. No
> external systems in scope.

## Modules

The components and each one's responsibility.

- **Create** — name + one-line responsibility; tag the layer (Frontend/Backend).
- **Edit** — keep responsibilities non-overlapping (one home per concern).
- **Delete** — re-home or drop its responsibilities; fix Data Flow.
- **Visualize** — C4 L2 Container / L3 Component: the modules and their responsibilities.

> - **API** *(Backend)* — application logic and validation.

## Tech Stack

Chosen technologies per concern.

- **Create** — one entry per concern.
- **Edit** — note *why* a swap was made if the reason isn't obvious.

> - **Database** — PostgreSQL

## Data Flow

The end-to-end flows, step by step.

- **Create** — ordered steps; link entities/terms to `DomainModel.md`.
- **Edit** — a new entity often adds a persistence step here; a new integration adds a
  hop.
- **Visualize** — Sequence diagram: the ordered steps as messages between modules.

> - The User submits an [Expense](DomainModel.md) via the Web module.
> - The API validates the [Category](DomainModel.md) and persists it.

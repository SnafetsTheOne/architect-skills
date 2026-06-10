# Entity.md — guidance

How to write and maintain an `entities/<Term>.md` — the detailed structure of one
Core Domain term. The term's *meaning* stays in `DomainModel.md`; this doc elaborates
only its structure. *(Meta-guidance for the skill, not an actual entity.)*

Per header, deltas from the standard workflow in
[SKILL.md](SKILL.md#workflow-for-every-change) on **create / edit / delete** — plus a
**Visualize** default for sections worth drawing.

## Does the term even need its own doc?

Give a term its own doc when it has **rich structure** — its own keys, a lifecycle
(status), or several fields; leave trivial terms as a one-line Core Domain entry. Be
consistent within a project: document only the rich entities, or give every core
entity a doc — don't mix.

Creating one → also add the index row in `AGENTS.md` / `CLAUDE.md`.

## Description

- **Create** — one or two lines; link the `DomainModel.md` term, don't restate its
  meaning.

> Detailed structure of the **Expense** entity. The term is defined once in the
> [Domain Model](../DomainModel.md); this document only elaborates its structure.

## Key(s)

The natural identifier(s).

- **Create** — name the natural key; note a surrogate (DB) key if there is one.
- **Edit** — a key change is structural; check the Relationships that reference it.

> - numeric primary key in the database
> - **Reference** — a human-readable identifier (e.g. `EXP-2026-00042`)

## Status

Lifecycle states — **omit the whole section if the entity has none.**

- **Create** — list the states + what each means.
- **Edit** — adding/renaming a state: check Business Rules tied to status.
- **Delete** — removing a state: ensure nothing still transitions into it.
- **Visualize** — State-machine diagram: the states and the transitions between them.

> - Draft — not yet counted against a budget
> - Recorded — counts against the matching Budget
> - Voided

## Fields / Information

The data the entity holds.

- **Create** — `**Name** — what it is`; link types/terms to their SSoT (`DomainModel.md`,
  `Architecture.md`).
- **Visualize** — single-entity ER / class box: this entity's keys and fields in one box.

> - **Amount** — a monetary value (single currency, see [Architecture](../Architecture.md))
> - **Category** — exactly one [Category](../DomainModel.md)
> - **Note** — optional free text

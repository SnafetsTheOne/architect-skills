# DomainModel.md — guidance

Guidance for maintaining a project's `DomainModel.md` — the single source of truth
for the ubiquitous language. Every other doc links here for term meaning instead of
redefining. *(This is meta-guidance for the skill, not an actual domain model.)*

Below: per header, what's special **on create / edit / delete**. These are *deltas*
from the standard per-change workflow in [SKILL.md](SKILL.md#workflow-for-every-change)
— where a verb adds nothing, the standard workflow is all that applies. Keep entries
terse; not every header needs all three verbs. Sections worth drawing also carry a
**Visualize** default.

## Actors

Roles / external parties that use or feed the system. *An Actor is not automatically
an entity.*

- **Create** — `**Name** — one line: what it does / why it touches the system`. It
  earns a Core Domain entry **only if the entity gate (below) also holds**.
- **Delete** — only once no Relationship or Business Rule still names it; re-point
  those first.

> **User** — records expenses and sets budgets.

## Core Domain

The entities the system owns and persists. One line each: `**Term** — definition`
(+ `` `eng: snake_case` `` if you map terms to code).

- **Create** — **run the entity gate (below) first.** Then, in the same pass:
  - add the term + one-line definition (+ code name);
  - wire its **Relationships** to existing entities;
  - place it in a **Bounded Context**;
  - add any new **Business Rules** it introduces;
  - if it has rich structure, give it an `entities/<Term>.md` (see
    [Entity.md](Entity.md)) + an index row in `AGENTS.md` / `CLAUDE.md`;
  - link it from the **Requirement** that introduces it.
- **Edit** — a changed *definition* propagates to everything linked here (that's the
  point of SSoT). **Renaming** the term is a *rename ripple*: update every doc that
  uses it + the index, re-point links and the `eng:` name.
- **Delete** — confirm another entity already carries everything still known about it,
  or that it is truly gone from the model. Then re-point or drop every Relationship and
  Business Rule that names it, delete its `entities/<Term>.md` + index row, and **demote
  to Glossary** if the word is still spoken but no longer persists.
- **Visualize** — ER / class diagram: the entities and the Relationships among them.

> **Expense** — a single recorded spending event with an amount and a date. `eng: expense`

### Entity gate — before adding to Core Domain

A concept earns an entity only if it survives all of:

1. **Owns persistent state?** Does the system store/reason about it with its *own*
   identity and lifecycle, beyond what existing entities already carry? If not → it's
   an Actor, an attribute, or a Glossary term — not an entity.
2. **Removal test.** Remove it as an entity: does the model still hold because another
   entity already carries everything known about it? If yes, it's the *removable* one
   → don't add it.
3. **Additional value.** Does it answer a question the existing entities cannot? If no
   → don't add it.
4. **Stable-referent rule.** If something must be referenced *before* a related entity
   exists, promote the **minimal stable referent**, not the richest-sounding concept
   behind it.
5. **Phantom-entity smell.** If Relationships/Business Rules name a concept that has no
   definition or persistent state, fix it — rephrase the rule against the entity that
   *does* persist, or promote the concept per 1–4. Don't leave it dangling.

An Actor *may* also be an entity, but only when checks 1–3 hold. (E.g. a role the
system never stores beyond an authenticated session usually stays an Actor; an
identifier that other records must reference even before that session exists earns
promotion.)

## Relationships

Cardinalities between things that actually persist.

- **Create** — both ends must be defined entities (or a defined reference); state the
  cardinality.
- **Edit** — if a cardinality changes, confirm the affected Business Rules still hold.
- **Delete** — when the underlying link no longer persists; check no Business Rule
  depends on it.
- **Visualize** — ER / class diagram (same view as Core Domain): the cardinality on each
  association.

> - An **Expense** is assigned to exactly one **Category**.

## Bounded Contexts

Context boundaries, including external ones.

- **Create** — name the boundary + what it covers; mark external ones and their data
  owner.
- **Edit** — moving an entity between contexts is a real design decision, not a
  cosmetic move.
- **Delete** — re-home the entities it held first.
- **Visualize** — DDD Context Map: the contexts and the relationships between them.

> **Budgeting Context** — setting budgets and tracking spend against them.

## Business Rules

Invariants the domain must uphold.

- **Create** — phrase against entities that *persist*; every term named must be defined
  (else it's a phantom — gate #5).
- **Delete** — only when the invariant truly no longer holds.

> - Every **Expense** must have a **Category**.

## Glossary

Supporting terms that are **not** entities.

- **Create** — a term that's spoken but not persisted with its own identity.
- **Edit** — if it starts owning persistent state, **promote** it via the entity gate
  and move it to Core Domain (leave a "not a standalone object — see X" pointer if the
  word is still used).
- **Delete** — when it falls out of use.

> **Period** — the calendar month a budget applies to.

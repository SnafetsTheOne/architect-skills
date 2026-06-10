---
name: draft-plan-with-docs
description: >-
  Use when the user wants to plan upcoming work — a roadmap (milestones & ordering)
  , a feature's technical design contracts (SQL, API, event schemas, config), or user stories.
---

# Draft Plan (with Docs)

## Principles

- **Read the docs, don't change them.** Use `Requirements.md`, `DomainModel.md`
  and the entity docs as **read-only** input and follow their terminology. To
  change a fact in the docs, use `draft-docs` — not this skill.
- **One level per invocation.** A single run drafts **exactly one** level —
  roadmap, design, or user stories. Pick the level from the request.
- **No level requires the one above it.** You can draft user stories even if no
  `plan/roadmap.md` or `plan/design.md` exists; each level stands alone.

## Levels & output

Each level's format lives in its own reference file:

| Level        | Drafts…                                         | Output file            | Format reference                 |
| ------------ | ----------------------------------------------- | ---------------------- | -------------------------------- |
| Roadmap      | the milestone outline (goals & scope, ordering) | `plan/roadmap.md`      | [Roadmap.md](Roadmap.md)         |
| Design       | the technical contracts work must honor         | `plan/design.md`       | [Design.md](Design.md)           |
| User Stories | the stories themselves, in full story format    | `plan/user-stories.md` | [UserStories.md](UserStories.md) |

Create the `plan/` folder if missing.

## Workflow (every invocation)

1. **Read first** — the SSoT docs relevant to the plan (Requirements, DomainModel,
   entities) and the existing `plan/` file for this level if present.
2. **Pick the level** — from the user's intent. Do **only** that level; if they
   ask for more than one, do one and tell them to invoke again for the others.
3. **Clarify** — one focused question if the scope is ambiguous.
4. **Draft** the single level into its `plan/` file (formats below).
5. **Report** — summarize what you drafted; surface gaps or contradictions you
   found against the docs.

The Roadmap and Design levels are drafted straight from their reference files
([Roadmap.md](Roadmap.md), [Design.md](Design.md)). Design is written **once**
so user stories *link* to its contracts instead of re-embedding them. The User
Stories level has extra steps:

## User Stories level → `plan/user-stories.md`

Follow the full story format in **[UserStories.md](UserStories.md)** (story
opener, acceptance-criteria prefixes, optional contract sections, ID scheme,
document structure). Group stories under milestone headings; if `plan/roadmap.md`
exists, mirror its milestones and goals.

**Link to `plan/design.md` when it exists** — a story references the Design
sections it implements (`Design: plan/design.md#incident-tables`) instead of
re-embedding the schema/contract. Embed inline only as a fallback when no Design
level was drafted. See the Linking note in **[UserStories.md](UserStories.md)**.

**Before drafting — ClickDummy check.** Look for a detailed ClickDummy: a
`plan/clickdummy/` folder or a prototype link in `Requirements.md` / `plan/design.md`.
If none is found, ask whether to create one first. If yes, offer to generate a prompt
for a UI-prototyping tool (Claude Code, v0, Lovable, Figma Make) from the `### Pages`
in `plan/design.md`, written to `plan/clickdummy.prompt.md`. Advisory — stories can
still be drafted without one.

**After drafting — feasibility check.** Offer to check the plan for feasibility. If
the user agrees, run the per-story review in **[Feasibility.md](Feasibility.md)**.

## Language & terminology

- Match the documentation language of the project.
- Use terms already defined in the project's `DomainModel.md` / glossary — don't
  introduce synonyms.
- Code identifiers (table names, field names, modules) follow the project's code
  conventions.

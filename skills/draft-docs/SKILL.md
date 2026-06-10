---
name: draft-docs
description: >-
  Use when the user wants to create, change, extend, or restructure
  single-source-of-truth project documentation — specs, design, requirements,
  architecture, domain model / terminology, or entity docs (a DDD set or any docs)
---

# Single-Source-of-Truth Docs

Use this skill for **any** change to project documentation that is treated as a
source of truth (specs, design notes, domain/architecture docs).

When you work on a specific doc, **read its guidance file** (table below) for the
per-header create / edit / delete rules.

## Principles

- **One language.** Write in the project's existing documentation language; don't
  switch or mix.
- **Consistent terminology.** Use the terms defined in the project's glossary /
  domain doc. Don't introduce a synonym for an existing concept.
- **Single source of truth.** Document each fact exactly once and **link instead
  of copying**. If a fact already lives elsewhere, link to it.
- **Surface contradictions, don't guess.** Flag gaps and conflicts to the user
  instead of silently filling them.

## Workflow for every change

1. **Read first** — the target doc, the docs it links to, the doc index in
   `AGENTS.md` / `CLAUDE.md`, and the doc's guidance file (table below).
2. **SSoT check** — is this fact already documented? → keep it there and link.
   Where should it live? → the doc whose guidance says it belongs there.
3. **Make the change** in the doc's existing style and section, following the
   create / edit / delete rule for the affected header.
4. **Report** any contradictions or open questions you found.

## Drafting: interview, don't invent

Docs are often written before everything is known — that's normal. Build them by
interviewing the user relentlessly through the open questions, not by guessing. Walk
down each branch of the decision tree, resolving dependencies between decisions, until
you reach a shared understanding:

- **One question at a time.** Settle a decision before the ones that depend on it;
  don't batch questions.
- **Recommend an answer.** Offer your best suggestion with each question, so the user
  confirms or corrects rather than starting from blank.
- **Explore before asking.** If existing code, docs, or a referenced repo already
  answer it, find the answer there instead of asking.

Besides answering, three responses are **always open** to the user at any question —
each handled below:

- **"idk"** — defer the decision; capture it as a `TBD`, don't fill it in.
- **"zoom out"** — reject the *question*, not just the answer; back up to the decision
  it rested on and re-open it.
- **"visualize it"** — draw the decision and its competing options so the user can see
  it before answering; then re-pose the question.

### Marking a deferred decision — `TBD`

A `TBD` records that a decision was **deliberately not made yet**, so a reader (or a
later agent) can tell it apart from an oversight:

```md
**TBD** — <the open question>; <options or current leaning, if any>.
```

- Put it where the answer will live — the right doc and header — so the gap shows in
  context, and every open decision stays greppable as `TBD`.

### Zooming out — challenge the question, not the answer

When the user zooms out — however they phrase it ("zoom out", "wrong question", "I
think I got that one wrong") — the question was off, or an earlier answer no longer
holds. Don't press for an answer. Instead:

- **Stop drilling.** Surface the assumptions the question rested on; name them so a
  wrong one becomes visible.
- **Go up a level** to the decision those assumptions came from, and re-open it.
- **Offer a pause.** End the response with a low-key suggestion to break and resume
  later; work-in-progress is safe (settled facts are in the docs, open ones are
  `TBD`s). Offer once; never nag.

### Visualizing a decision — draw it, then re-ask

When the user says **"visualize it"**, render the decision on the table so they can
*see* it before answering — above all the **competing options**, drawn side by side to
compare. A line or two in chat rarely carries it; produce a real **draw.io** file.

- **Scope to the question, not the whole doc.** Draw the parts under discussion and the
  options being weighed, built from the *current* interview state (including `TBD`s) —
  not re-derived from the saved files. (The whole-doc-set view is the separate
  `visualize` skill; this is a throwaway thinking aid.)
- **Pick the view from the section's guidance file** — each visualizable header carries a
  **Visualize** default. When a question doesn't fit one, draw whatever makes the options
  clearest.
- **It's an aid, not an answer.** After opening the diagram, re-pose the pending
  question. The decision stays open — the user can answer, or still say `idk` or
  `zoom out`.
- **Never in the repo.** Temp folder only, no index row, no changelog — same rule as the
  `visualize` skill.

**Mechanics** — write a self-contained `.drawio` to the OS temp dir (never the repo) at a fresh timestamped path (`$TMPDIR`, or
`%TEMP%` on Windows → `…/draft-docs-viz/<topic>-<timestamp>.drawio`); put the
options side by side; validate it as XML; open it (`start` / `open` / `xdg-open`); then
state the full absolute path.

## What lives in which doc

Keep each fact in exactly one doc. Section names are illustrative — follow the
project's language and conventions; the point is *what information lives where*.
Each doc has a guidance file with per-header create / edit / delete rules.

| Doc                   | Holds                                                                                       | Guidance                       |
| --------------------- | ------------------------------------------------------------------------------------------- | ------------------------------ |
| `Requirements.md`     | the *what* & *why* — vision, functional capabilities                                        | [Requirements.md](Requirements.md) |
| `Architecture.md`     | the *how* — context, modules, tech stack, data flow                                         | [Architecture.md](Architecture.md) |
| `DomainModel.md`      | terminology SSoT — actors, core domain, relationships, bounded contexts, rules, glossary    | [DomainModel.md](DomainModel.md)   |
| `entities/<Term>.md`  | one entity's structure — keys, status, fields                                               | [Entity.md](Entity.md)             |

Add more docs as a project grows — most often one entity doc per major term.

## Indexing docs in AGENTS.md / CLAUDE.md

The agent guide should **point to** docs, never duplicate their content. Keep a
single index table there linking every SSoT doc with a one-line summary:

```md
| Topic              | Document                                     |
| ------------------ | -------------------------------------------- |
| Vision & reqs      | [docs/Requirements.md](docs/Requirements.md) |
| Context & data flow| [docs/Architecture.md](docs/Architecture.md) |
| Terminology/domain | [docs/DomainModel.md](docs/DomainModel.md)   |
```

- Creating a doc → add a row. Renaming/removing → update/remove the row.
- No index yet? Create one.

## Creating a new document

1. Create it in the docs location, in the project's language and style; link to
   existing facts instead of restating them. Follow its guidance file (table above).
2. Add it to the index in `AGENTS.md` / `CLAUDE.md`.

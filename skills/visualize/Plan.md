# Plan mode – diagrams from the plan/ folder

The plan is authoritative; derive fresh from the `plan/` files each run.

## Precondition

A `plan/` folder — the output of the **draft-plan-with-docs** skill (`plan/roadmap.md`, `plan/design.md`, `plan/user-stories.md`). Not every file need exist; draw the views the available files support and note which were skipped for lack of source.

## Views (min. 6 = up to 8 pages)

Use the standard notation/colors of each diagram type.

1. **Story Map** – from `plan/user-stories.md`, sliced into releases per `plan/roadmap.md`.
2. **Hierarchy Tree** – Epic → Story → Task from `plan/user-stories.md` under `plan/roadmap.md` milestones.
3. **Roadmap / Timeline** – epics/milestones over time from `plan/roadmap.md`.
4. **ER diagram** – entities/tables from the SQL in `plan/design.md`.
5. **API/Data Contract Map** – routes + request/response bodies from `plan/design.md`.
6. **Event/Stream Schema** – topics/events from `plan/design.md` (producers → topic → consumers); skip if the design defines none.
7. **Sequence diagram** – a key planned flow from `plan/design.md` contracts + acceptance criteria in `plan/user-stories.md`.
8. **C4 L2 – Container (as planned)** – to-be target architecture from roadmap scope + design.

## Assumptions to flag

The plan often leaves gaps the diagram has to bridge — story→task decomposition, the chosen key flow, container boundaries not yet built. Flag these as assumptions in the chat report. Where `plan/` files contradict each other or the docs, report it instead of guessing.

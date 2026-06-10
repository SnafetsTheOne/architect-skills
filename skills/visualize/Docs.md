# Docs mode – diagrams from the project docs

The docs are authoritative; regenerate fresh from them each run.

## Precondition

SSoT project docs — by default `docs/Requirements.md`, `docs/Architecture.md`, `docs/DomainModel.md`, `docs/entities/*`; follow the doc index in `AGENTS.md`/`CLAUDE.md` if the names differ. Not every file need exist; draw the views the available docs support and note which were skipped for lack of source.

## Views (min. 6 = 6 pages)

Use the standard notation/colors of each diagram type.

1. **User Journey** – the primary persona's end-to-end journey, phases from the requirements doc.
2. **DDD Context Map** – bounded contexts and external systems from the domain model / architecture docs.
3. **Event Storming** – one swimlane per domain process or flow described in the docs.
4. **C4 L1 – System Context** – personas, the system, external systems.
5. **C4 L2 – Container** – deployable units from the architecture doc (frontend, services, databases, brokers).
6. **C4 L3 – Component** – modules of the most central container from the architecture doc.

## Assumptions to flag

DDD relationship patterns (OHS/PL/Conformist/Customer-Supplier) and Event Storming policies are rarely stated verbatim in docs → flag them as assumptions in the chat report. Where the docs are silent, name the gap instead of guessing.

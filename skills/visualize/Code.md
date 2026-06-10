# Code mode – diagrams from the implementation

The code is authoritative, not the docs. Where code and docs diverge, report it.

## Precondition

A detectable implementation: build files (`pom.xml`, `build.gradle`, `package.json`/`angular.json`, `*.csproj`, `go.mod`, `Cargo.toml`, …) or a `src/` tree with source files.

## Sources

Actual source tree, build files, imports, ORM entities / schema migrations.

## Views (min. 6 = 6 pages)

Use the standard notation/colors of each diagram type.

1. **C4 L1 – System Context** (as built) – the system, its users, and the external systems the code actually talks to.
2. **C4 L2 – Container** – deployable units found in the repo: apps, services, databases/schemas, brokers/connectors.
3. **C4 L3 – Component** – packages/modules per container.
4. **Module/package dependency graph** – from imports/build files.
5. **Sequence diagram** – a key flow picked from the code (e.g. the main request path or a central job); flag the pick as an assumption.
6. **Class/ER diagram** – core entities/tables from ORM entities or schema migrations.

## Assumptions to flag

External systems and personas are inferred from integration code and config — name the evidence. The key flow chosen for the sequence diagram is a pick, not a fact.

# Design Format

Reference for the **design level** of `draft-plan-with-docs`, written into
`plan/design.md`. This file is the source of truth for the contract formats
below; the inline fallbacks in **[UserStories.md](UserStories.md)** use the same
shapes.

## Form

- **One markdown file by default.** Write each artifact as a fenced code block
  (` ```sql `, ` ```json `, ` ```yaml `) with its rationale beside it — narrative +
  cross-reference, not a runnable script.
- **Group by bounded context / feature**, so a feature's table, API and event sit
  together; add a shared section for cross-cutting artifacts.
- **Stable heading per artifact** so stories can deep-link:
  `plan/design.md#incident-tables`.

## Document Structure

    ## <Context / Feature>

    ### Tables
    ```sql
    CREATE TABLE <schema>.<table> ( ... );
    ```

    ### API
    **`POST /incidents`** — <what it does>
    …query params / request / response…

    ### Events
    **`incident.created`** (<topic>) — …schema…

    ### Pages
    **`/incidents`** — …interactions…

    ### Config
    - `Namespace.Key` — description

## Artifact Formats

### Tables

Pseudo-SQL `CREATE TABLE` specifying schema, table, and every column with its type
and nullability. Use `--` comments for anything non-obvious.

    ```sql
    CREATE TABLE <schema>.<table> (
        column_name   TYPE         NOT NULL,
        column_name   TYPE         NULL,
        -- comment if non-obvious
    );
    ```

- List all columns, including PKs and FKs.
- Use the target database's type names (`BIGINT`, `TEXT`, `TIMESTAMP`, `BOOLEAN`).
- For an alter, show only the affected columns with `-- ADD` / `-- DROP` / `-- MODIFY`.

### API / Data Contracts

One entry per endpoint: the route, plus the query params, request body and response
body that apply. Inline the JSON, or reference the file where a large body lives.

    **`POST /incidents`** — create an incident

    Query params: `?status=<enum>&page=<int>`

    Request:
    ```json
    { "field": "type / example" }
    ```
    Response:
    ```json
    { "field": "type / example" }
    ```

- When just listing endpoints, one line each: `` `GET /incidents/{id}` ``.
- Reference form for a large body: `→ plan/contracts/incident.response.json`.

### Events / Stream Schemas

For each topic/stream: the name, when it is emitted, the payload schema (JSON or
Avro), and the key fields. Inline small schemas; promote Avro/registry schemas.

    **`incident.created`** (Kafka topic) — emitted when an incident is opened

    ```json
    { "incidentId": "uuid", "status": "string", "openedAt": "timestamp" }
    ```

    or → `plan/contracts/incident.avsc`

### Pages

One entry per page: the route (with query params when they carry state) and the
user interactions. No API wiring or states — the endpoints live under `### API`,
the states are user-story acceptance criteria.

    **`/incidents?status=<enum>&page=<int>`**

    Interactions: filter by status/date, open incident details, acknowledge an incident.

### Config

List each parameter with a suggested name in `Namespace.Key` dot-notation.

    - `Namespace.Key` — description, e.g. default value or unit

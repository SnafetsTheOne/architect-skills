# Feasibility Review

Optional **exit check** for the user-stories level of `draft-plan-with-docs`. After
the stories are drafted, offer to verify the plan is actually buildable; if the user
agrees, run this review. Chat with the user about the findings.

## Procedure

1. **Gather inputs** — the drafted stories in `plan/user-stories.md`, the Design
   artifacts they link to in `plan/design.md`, and the tech stack from
   `Architecture.md`.
2. **One subagent per story.** Spawn the subagents in parallel (Agent tool). Give
   each: the single story, its linked Design sections (tables / contracts / schemas
   / pages), and the relevant tech stack. Ask it to run every check below and return
   structured findings.
3. **Report in chat** — present the collected findings to the user (format below)
   and hold them in context, ready to discuss. Do not write a file.

## Subagent input / output

Each subagent receives **one** story + its design + the tech stack, and returns, per
check: a verdict — `OK`, or a `Concern` with a severity (**Blocker** / **Major** /
**Minor**) — and a one-line note. For a concern, the note says what the limitation is
and which artifact it affects.

## Checks

Run every check against the story:

1. **Tech ↔ data-format compatibility.** Is the chosen technology compatible with the
   story's data formats, or are there limitations that were not considered up front?
   E.g. column types vs the database's capabilities, payload size vs broker/message
   limits, JSON vs Avro / schema-registry support, field types vs the framework/ORM.
2. **Contract completeness.** Does the story link to Design artifacts that actually
   exist and cover what it needs? Catches a page using an endpoint that was never
   defined, an API field with no backing column, or a `Design:` anchor that points at
   nothing.
3. **Testability of acceptance criteria.** Is each criterion verifiable with the prefix
   it claims (`Test:` / `CI:` / `Check:`)? Catches untestable or mis-prefixed criteria.

## Reporting

Present the concerns in chat as a single list **ordered by severity, worst first**
(Blocker → Major → Minor); each line names its story and the check it came from:

    [Blocker] <Story ID> · Tech ↔ data-format — <note>
    [Major]   <Story ID> · Contract completeness — <note>
    [Minor]   <Story ID> · Testability — <note>

Collapse or omit the all-OK results. Then invite the user to dig into any finding — the
goal is a conversation about feasibility, not a report to file away.

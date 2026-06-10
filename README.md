# architect-skills

## Skills

| Skill | Use for |
| --- | --- |
| `draft-docs` | Maintain single-source-of-truth Markdown docs (requirements, architecture, domain model, entities). |
| `draft-plan-with-docs` | Plan upcoming work into a `plan/` folder — roadmap, design contracts, or user stories. |
| `visualize` | draw.io diagrams from the code, docs, or plan (mode optional — detected from the request/repo), or anything ad hoc. |
| `write-e2e-tests` | Write BDD-style Given/When/Then E2E tests; scaffold the layered test harness for C#, Java, or TypeScript. |
| `implement-plan` | Execute an existing plan with an agent team — a delegating-only team lead, dev personas in isolated git worktrees, and per-story verification gates (Verifier, Architect; optional Performance/Security/Compliance). |

## Install

### Claude Code:

```text
/plugin marketplace add SnafetsTheOne/architect-skills
/plugin install architect@architect-skills
/reload-plugins
```

Skills become available in every session, namespaced as `architect:<skill>`
(e.g. `architect:draft-docs`).

### Github Copilot CLI:

```text
/plugin marketplace add SnafetsTheOne/architect-skills
/plugin install architect@architect-skills
```

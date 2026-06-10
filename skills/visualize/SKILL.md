---
name: visualize
description: >-
  Generates draw.io (diagrams.net) diagrams as a single multi-page .drawio file
  in a fresh, timestamped temp folder (never in the repo), validates it, and
  opens it. Three standard modes, picked from an optional argument or detected
  from the repo — code (C4 as built, dependency graph, sequence, class/ER from
  the source tree), docs (user journey, DDD context map, event storming, C4
  from the project docs), plan (story map, epic/story tree, roadmap timeline,
  ER/API/event contracts from plan/) — plus an ad-hoc mode for anything else
  worth drawing. Use when the user wants to visualize the code, architecture,
  docs, domain, plan, or anything on the fly, asks for draw.io / diagrams.net
  diagrams, a C4 model, context map, story map, dependency graph, ER/class,
  sequence, event-storming, or user-journey diagram, or runs /visualize
  (optionally /visualize code|docs|plan|<topic>).
---

# visualize – draw.io diagrams from code, docs, plan, or ad hoc

## Principle

Diagrams are **derived views**, not a single source of truth.

- **Never write into the repo.** No signpost entry (`CLAUDE.md`/`AGENTS.md`), no changelog.
- Derive fresh from the chosen source each run. On contradictions, **report instead of guessing**.
- Diagram labels use the project's documentation language; apply the ubiquitous language from the domain doc (e.g. `docs/DomainModel.md`) consistently.

## Picking the mode

The argument is **optional**. Resolve in this order:

1. **Named mode** — the user said `code`, `docs`, or `plan`, or clearly meant one ("as built" → code, "the domain" → docs, "the roadmap" → plan). Use it.
2. **Free-form subject** — the user named something that isn't one of the three sets (a single flow, one entity, a decision, something from the conversation). Use **Ad-hoc mode** below.
3. **No hint** — detect what the repo offers: `plan/` files → plan, SSoT docs (`docs/*.md`) → docs, a detectable implementation → code. Exactly one match → use it; several → ask the user which (offering "all of them" is fine).

For a standard mode, read its reference — [Code.md](Code.md), [Docs.md](Docs.md), or [Plan.md](Plan.md) — for sources, views, and mode-specific assumptions. Each standard mode produces **≥ 6 pages**. If the chosen mode's sources are missing, say so and name the modes whose sources do exist instead.

## Ad-hoc mode

For anything that doesn't fit the three sets:

- **Scope = the request.** Draw exactly what the user asked about, from whatever source fits (code, docs, plan, or the conversation itself).
- **No page minimum.** One page is fine; add pages only when they clarify.
- Pick the diagram types that make the subject clearest; standard notation/colors still apply.

## Output & open

Write a self-contained, **multi-page** .drawio file to the OS temp directory so nothing lands in the repo. Resolve the temp dir from `$TMPDIR` (fallback `/tmp`, or `%TEMP%` on Windows); each run gets a fresh path `<tmpdir>/visualize-<mode>-<timestamp>/<project>-<mode>-<timestamp>.drawio`, where `<mode>` is `code`/`docs`/`plan`/`adhoc` and `<project>` is the project name. Validate it as XML after writing (snippet below). Open it for the user — `xdg-open <path>` (Linux), `open <path>` (macOS), `start <path>` (Windows). **Always** end by stating the **full absolute path** to the generated file (so the user can reopen it even if it didn't come to the foreground), along with the assumptions you interpreted.

## draw.io mechanics

- One `<mxfile>` with multiple `<diagram name="…" id="…">` (one per view) → draw.io tabs.
- Escape text: `&` → `&amp;`, line break `&#10;`, avoid raw `<`/`>` (use `→`), non-ASCII as UTF-8.
- Validate after writing:
```powershell
Get-ChildItem "$dir\*.drawio" | ForEach-Object { try { [xml]$x=Get-Content $_.FullName -Raw; "OK $($_.Name) cells=$($x.SelectNodes('//mxCell').Count)" } catch { "FAIL $($_.Name): $($_.Exception.Message)" } }
```

## Flag assumptions

Every inferred choice (the key flow picked for a sequence diagram, a boundary the source leaves implicit) is an **assumption** — list them in the chat report. Where sources are silent or contradict each other, name the gap instead of guessing. Mode-specific pitfalls are listed in each reference file.

# plans/

This directory holds an immutable record of the prose plan behind each step of implementation work, so the reasoning that went into a change survives past the commit that implements it.

## Directory layout

`plans/v{N}/{NN}-{slug}.md`, e.g. `plans/v1/01-project-scaffolding.md`.

- `{N}` is the ROADMAP.md version the step belongs to.
- `{NN}` is a two-digit, zero-padded, sequential number within that version's folder, in execution order.
- `{slug}` is a short, hyphenated description of the step.

## Plan file header

Each file starts with this header before the plan prose:

```
# {NN} — {Title}

**Date:** YYYY-MM-DD
**Implements:** ROADMAP v{N} — {version name}
**Status:** done
```

`Status` is the only line ever edited after creation. Values:

- `done`
- `superseded — see plans/vN/NN-slug.md`

## Immutability

The rest of the file is frozen — a record of what was approved going in, not what happened. Deviations discovered during implementation belong in commit history, not edits to the plan.

## ROADMAP.md status lines

Each version section in `ROADMAP.md` carries a `**Status:**` line directly under its `**Goal.**` line, kept in sync with this directory:

- `**Status:** not started` — default, no plan files yet for that version.
- `**Status:** in progress — plans/v1/01-slug.md, plans/v1/02-slug.md` — updated, with links appended, as each plan file is created.
- `**Status:** done — plans/v1/01-slug.md, ...` — set only when that version's own "Done when" criteria are met.

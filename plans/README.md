# plans/

This directory holds an immutable record of the prose plan behind each unit of approved work, so the reasoning that went into a change survives past the commits that implement it.

A plan is the unit of *approval*, not the unit of commit. One plan covers a block of work that the user has signed off on as a whole, and a block typically spans several commits. The small-increment rule in `CLAUDE.md` still governs those commits: within a plan, work lands one logical change at a time, with a review pause between each. The plan says what the whole block is for; the commits inside it stay small.

## Directory layout

`plans/v{N}/{NN}-{slug}.md`, e.g. `plans/v1/01-plumbing.md`.

- `{N}` is the ROADMAP.md version the block belongs to.
- `{NN}` is a two-digit, zero-padded, sequential number within that version's folder, in execution order.
- `{slug}` is a short, hyphenated description of the block.

## Plan file header

Each file starts with this header before the plan prose:

```
# {NN} — {Title}

**Date:** YYYY-MM-DD
**Implements:** ROADMAP v{N} — {version name}
**Status:** planned
```

`Status` is the only line ever edited after creation. Values, in the order a plan normally moves through them:

- `planned` — written and approved, no code from it has landed yet. This is the value a plan carries on creation.
- `in progress` — the block's first step has landed; the rest has not.
- `done` — every step in the block has landed.
- `superseded — see plans/vN/NN-slug.md`

## Immutability

The rest of the file is frozen — a record of what was approved going in, not what happened. Deviations discovered during implementation belong in commit history, not edits to the plan.

## ROADMAP.md status lines

Each version section in `ROADMAP.md` carries a `**Status:**` line directly under its `**Goal.**` line, kept in sync with this directory. It uses the same vocabulary as the plan files above, so the two never have to be translated into each other:

- `**Status:** not started` — default, no plan files yet for that version.
- `**Status:** planned — plans/v1/01-slug.md, plans/v1/02-slug.md` — plan files exist and links are appended as each is created, but no code from them has landed.
- `**Status:** in progress — plans/v1/01-slug.md, ...` — implementation of at least one plan has begun.
- `**Status:** done — plans/v1/01-slug.md, ...` — set only when that version's own "Done when" criteria are met.

A version's line is the coarser of the two: it says where the version stands, while each plan file's own `Status` says where that block stands.

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

This repository is a fresh scaffold for **spotify-playlist-organizer**. No code, dependency manifest, or build tooling exists yet — only documentation: `README.md`, this file, and `ROADMAP.md`.

There is no architecture or command set to document at this stage. The stack, feature scope, and version-by-version roadmap are decided and recorded in `ROADMAP.md`. When the project's structure is real, update this file with:

- Build, lint, and test commands (including how to run a single test)
- High-level architecture and module boundaries
- Any non-obvious conventions specific to this codebase

The prose plan behind each step of implementation work is recorded under `plans/`, following the convention in `plans/README.md`. Check there for the plan a given step was implemented from.

## Working with the user

- Don't hand-format code — defer to the linter. If no linter is configured yet, say so explicitly rather than formatting manually.
- After ANY change, check both `README.md` and `CLAUDE.md` for staleness. If the change is non-trivial, document it in each; if it makes an existing statement inaccurate, correct it. Skip only when the change genuinely has no bearing on anything those files say.

See *Teaching the stack* below for what a brainstorm in this repo must additionally cover.

## Teaching the stack

The user is building this project to gain experience with its stack. Assume fluency in general programming and in JavaScript. The TypeScript type system, React's rendering and state model, the Vite and Vitest tooling, and OAuth — the PKCE flow, token handling, and rate-limit-aware API client design — are the areas the project exists to build experience in, so treat them as material to teach rather than as shared background.

**Assess before planning.** During brainstorming, before entering Plan mode, identify which of the concepts above the upcoming work depends on. Ask the user in a single batch whether they have worked with each one — not one interruption per concept, and not a quiz about things they have already used in this repo. Their answer decides which get taught and which get a passing reference.

**Introduce briefly while planning.** For each concept the user has not used before, the brainstorm and the plan's prose get a short orientation only — a sentence or two on what it is and why this project needs it. Enough that the plan reads clearly; no depth, no mechanics, no survey of alternatives. Those wait for implementation.

**Teach in full at implementation.** After implementing a step that introduced a new concept, give the real explanation: what the concept is in depth, what the alternative was and why it lost, and a walk through what the code does — line by line wherever a line is doing something non-obvious. The small-increment rule makes this cheap: a step is roughly one commit, so there is never much to cover at once. Go deep the first few times a concept appears, then speed up as the pattern repeats — once the user has seen three components wired the same way, the fourth needs a sentence, not a walkthrough. Tedium defeats the purpose, so err toward moving faster when the user is clearly ahead of the explanation.

**Guards.**

- Teaching goes in conversation and plan prose, never into the source as tutorial comments. The "avoid self-explanatory comments" rule still holds.
- Explain a concept the first time it appears; afterwards refer back to it rather than re-teaching it.
- Explain the concept, not the syntax. Language mechanics already covered by JavaScript need no coverage.
- If the user says they already know something, drop it and move on.

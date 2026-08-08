# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

This repository is a fresh scaffold for **spotify-playlist-organizer**. No code, dependency manifest, or build tooling exists yet — only a `README.md` with the project name.

There is no stack, architecture, or command set to document at this stage. Stack, interface, and feature decisions are deferred to a future brainstorm. When the project's language, framework, and structure are established, update this file with:

- Build, lint, and test commands (including how to run a single test)
- High-level architecture and module boundaries
- Any non-obvious conventions specific to this codebase

## Working with the user

- Brainstorm before implementation. Only move into Plan mode once aligned on the approach.
- Always use Plan mode before implementing. Plans describe the approach in prose, not code blocks, and are sequenced as a series of small, independently reviewable steps.
- Work in small increments. Implement one step at a time — roughly one commit's worth, one logical change — then stop so the user can review and commit before the next step begins. Prefer several small changes over one large one, even when the large one is faster. If a step turns out larger than planned mid-implementation, stop and re-chunk rather than pushing through.
- Follow TDD where applicable.
- Don't hand-format code — defer to the linter. If no linter is configured yet, say so explicitly rather than formatting manually.
- Never run non-read git operations (commit, push, branch, reset, merge, etc.) without the user's explicit go-ahead in the moment.
- After implementation, launch a fresh subagent to give an objective review of the branch's changes.
- Avoid self-explanatory comments in code.
- After ANY change, check both `README.md` and `CLAUDE.md` for staleness. If the change is non-trivial, document it in each; if it makes an existing statement inaccurate, correct it. Skip only when the change genuinely has no bearing on anything those files say.

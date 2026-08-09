# 03 — Triage — find the unfiled songs and decide

**Date:** 2026-08-08
**Implements:** ROADMAP v1 — Find and file
**Status:** planned

## What this block delivers

At the end of this block the app answers the question it exists to ask. Picking up where `02-getting-the-data.md` leaves off — a completed scan holding the contents of **Playlist A** and of every playlist in the **comparison group** — the app computes which songs in Playlist A appear nowhere in the group, presents them one at a time, and lets the user decide each one from the keyboard: file it into one or more **target playlists**, mark it for removal from Playlist A, or both, or neither. Those decisions accumulate as **staged changes** that survive a refresh, a crash, or a closed tab.

The app remains one screen with no router. Nothing in this block talks to Spotify — every request the app makes still belongs to the login and scan phases built in the two previous blocks. A session here ends with a screen full of decisions and no way to apply them, which is the intended end state: applying them is `04-commit.md`.

## New concepts

**Test-driven development against pure functions.** The matching rules are the first thing in this project with a crisp, testable contract — data in, data out, no network, no React. Writing the Vitest cases before the implementation is cheap here and pins down the awkward inputs (relinked tracks, missing ISRCs, local files) before there is any code to be attached to.

**TypeScript discriminated unions and narrowing.** A union type whose members share a common literal field — a `kind` or `type` tag — lets the compiler figure out which member it is holding once that field has been checked, and refuse access to fields the other members do not have. Playlist items come in kinds that must be treated differently (a real track, a local file, a podcast episode), so this is the natural shape for the matching module's internal model.

**`noUncheckedIndexedAccess`.** This is the compiler flag from `ROADMAP.md`'s Stack section, and this block is the reason it is on. It makes every index lookup produce `T | undefined` rather than `T`, which is exactly right for the ID-and-ISRC index built here, where a lookup miss is the normal case — a miss *is* the answer "this song is unfiled" — rather than an error.

**Keyboard focus management in React.** React owns the DOM, so "which element is focused" has to be treated as state the app drives rather than something the browser is left to decide. The triage list needs one focused song at a time, keystrokes that land on it reliably, and the browser's own focus kept in step with the app's idea of it.

## Steps

### Step 7 — Matching

Build the matching module first, test-first, as a pure function with no React and no I/O: it takes the fetched contents of Playlist A and of the comparison group and returns the songs in Playlist A that appear nowhere in the group. Start the Vitest file, watch the cases fail, then write the module.

The rules are the ones fixed in `ROADMAP.md`. Two songs match if their track IDs are equal *or* their ISRCs are equal — either alone is sufficient. Relinked tracks are collapsed via `linked_from`, so a track object that carries a `linked_from` is indexed and compared under the original ID as well as the market-specific one, letting the same recording match itself across markets. Local files and podcast episodes are skipped entirely, because neither carries an identifier the rules can use; the same goes for the null track objects Spotify returns for items that have gone away.

Internally, normalize each playlist item into a small discriminated union before any comparison happens — one member for a matchable track carrying its identifiers, others for the kinds that are skipped — so the skip decision is made once, in one place, and the comparison code only ever sees things it can compare. The comparison group becomes an index keyed by both track ID and ISRC; each song in Playlist A is then a pair of lookups against it, and a song survives to the results only when both miss. This is where `noUncheckedIndexedAccess` earns its place: the misses are the interesting outcome and the compiler will not let them be ignored.

Worth covering in the tests: an ID-only match, an ISRC-only match, a track with no ISRC at all, a relinked track matching its original, a local file and an episode in each of Playlist A and the group, duplicate entries within a single playlist, and an empty comparison group.

Finish the step by surfacing the output in the UI as a plain results list — the count of unfiled songs and a row per song with title and artists, no interactivity. That is what makes this step a visible, working commit rather than a library with nothing calling it.

### Step 8 — The triage UI

Turn the results list into the triage screen. One song is in focus at a time; the rest of the list stays visible for context but only the focused song accepts decisions. Focus moves with the arrow keys and by clicking a row, and the focused row is given real DOM focus through a ref so keystrokes reach the handler and the browser's focus ring agrees with the app's state. Keep the key handler on the list container rather than on `window`, so typing elsewhere on the page cannot be mistaken for triage input.

Derive the target playlists from the comparison group by removing what cannot be written to: playlists the user follows but does not own, and Playlist A itself. `ROADMAP.md` treats these as the same set by design, so there is no separate destination picker to build — the targets are the comparison group, filtered.

Number keys toggle target playlists for the focused song, bound to the first nine targets in the order they are displayed. Targets past the ninth are reachable by click; every target, including the first nine, is clickable, so the keyboard is an accelerator rather than the only route. Toggling is symmetric — pressing the same key again removes the target.

A separate per-song toggle marks the song for removal from Playlist A, available both as its own key and as its own control. `ROADMAP.md` is emphatic on this point: removal is never implied by filing. Filing a song into three playlists stages three adds and nothing else. Staging a removal with no adds at all is a legitimate decision the app must accept without warning or interference.

Hold the decisions in a `useReducer` as one record per song — the song's identifiers, the set of target playlists it is staged into, and whether it is staged for removal. A song with an empty set and no removal has no record. Each row shows its own state at a glance, so a user scrolling back can see what they already decided.

Unless this step and step 9 are merged (see Open items), this step ships a visible "not yet saved" indicator on the screen, stating plainly that staged changes are lost on refresh until the next step lands. The limitation is disclosed rather than hidden.

### Step 9 — Staging persistence

Persist the staged changes to `localStorage` as one record per song, written whenever the reducer's state changes, and read back on load so a refresh, a crash, or a closed tab does not lose triage work. Scope the stored data under a namespaced key tied to Playlist A, so triaging a different source does not inherit an unrelated session's decisions.

Treat what comes out of storage as untrusted input rather than as the type it was written from: parse it, validate its shape, and discard the whole thing rather than crash if it does not match — a stored record from a half-built earlier version must not be able to break the app on load. Restoration happens once the scan has produced a current unfiled list, and reconciles against it: records whose song is no longer in the list are dropped, as are staged targets that are no longer in the comparison group, and the pruned set is written back so storage does not accumulate stale entries.

Remove the "not yet saved" indicator from step 8 as part of this step, and replace it with an honest one — a quiet confirmation that decisions are saved locally, distinct from anything that would suggest they have reached Spotify, which they have not.

## Working-state check

Run `npm run dev`, open the app on the loopback address, log in, and run a scan of a real Playlist A against a real comparison group. The unfiled list appears with a count. Key through it: arrow keys move focus, number keys toggle targets on the focused song, the removal key marks it for removal from Playlist A, and each row reflects its own decisions. Refresh the page. After logging back in and re-running the scan, every decision is still there.

`npm test` passes, including the matching cases written in step 7.

## Deliberately absent

- **Any commit control whatsoever.** Nothing in this block reaches Spotify — no adds, no removes, no verification, no batching. There is no button that would do so, not even a disabled one. Committing is `04-commit.md`.
- **Playback.** Deciding where a song belongs without hearing it is v1's accepted limitation; playback is v2.
- **Fuzzy matching.** Exact ID and ISRC matching only, per `ROADMAP.md`. The "probably already filed" bucket is v6.
- **Undo history, bulk operations, and filtering or sorting the unfiled list.** Decisions are individually reversible by toggling them off, which is enough for v1.
- **Persistence of anything other than staged changes.** The fetched track mirror stays in memory; moving it to IndexedDB is v4.
- **Home-screen installability and offline behavior**, which are `05-installable.md`.

## Open items

- **Whether steps 8 and 9 merge into one commit.** Triage that silently loses work on a refresh is arguably broken rather than merely incomplete, which puts a stand-alone step 8 in tension with the working-software rule. The mitigation if they stay separate is stated in step 8: a visible "not yet saved" indicator, so the limitation is disclosed rather than hidden. Settle this before starting step 8.
- **The keyboard mapping beyond nine target playlists.** `ROADMAP.md` explicitly leaves this open and makes it a v1 non-requirement. Click access to every target is the guaranteed floor; anything better is optional and can be decided while step 8 is being built.
- **How narrowly the `localStorage` key is scoped.** Tying it to Playlist A is the assumption in step 9, but whether the comparison group should also form part of the key — and therefore whether changing the group discards prior decisions or merely prunes them — is not settled by `ROADMAP.md` and should be decided when step 9 is written.

# 04 — Commit — write the staged changes

**Date:** 2026-08-08
**Implements:** ROADMAP v1 — Find and file
**Status:** planned

Everything before this block reads. This block is the first that writes, and it is the only place in v1 where a bug costs the user something they cannot get back. A mistake in the fetch phase wastes a minute; a mistake here removes a song from Playlist A that was never added anywhere else, and the user has no record of what it was. That asymmetry is the reason this block exists as its own document rather than as the tail of `03-triage.md`: it is not large, it is risky, and it deserves to be reviewed on its own terms.

## What this block delivers

After this block the core loop of v1 closes. The user logs in, picks Playlist A, picks a comparison group, waits out the fetch, triages the unfiled songs into target playlists and marks some of them for removal from the source — and then presses commit and the decisions land in Spotify. Opening the Spotify client afterwards shows the songs in their new playlists and gone from Playlist A, exactly as staged.

`03-triage.md` leaves the app with **staged changes** persisted to `localStorage` and no way to apply them. This block adds the commit runner and the single control that starts it, plus enough progress reporting that a run of a couple dozen requests does not look like a hang. When the runner finishes cleanly it clears the staged changes it applied. When it fails part way — a dropped connection, a `429` storm that outlasts its retries — it holds the staged changes rather than discarding them, so the user can press commit again later and land the rest.

The invariant the block is built around, and the one the working-state check verifies deliberately, is that at every instant during and after a commit each song is either in its original playlist or in its new one. Never in neither.

## New concepts

None. This block introduces nothing new to the stack — no new library, no new React pattern, no new build tooling, no new part of the auth flow. It uses the API client from `01-plumbing.md` with its `429` and `Retry-After` handling already in place, the pure-function-plus-Vitest habit established in `03-triage.md`, and the same `localStorage` record shape that block already persists.

That is precisely why it stands alone instead of being folded into `03-triage.md`. A step whose difficulty is entirely domain correctness is a step where the review should be about the rules — batch sizes, ordering, the removal gate, what counts as an ambiguous outcome — and not about how the code is wired together. Folding it into the triage block would bury those rules under UI work and make the one genuinely dangerous change in v1 the least conspicuous part of a large diff. Kept separate, the commit runner gets a diff a reader can hold in their head.

## Steps

### Step 10 — The commit runner

Build the runner in two movements inside one commit: first the decision-making, as pure functions with tests, then the execution that calls them.

**Planning the run.** A pure function takes the persisted staged changes and produces the ordered list of requests to make. It groups the staged changes by destination, then splits each group into batches at that endpoint's own limit — 100 track URIs per playlist add or remove, 50 bare track IDs per saved-tracks write. The two paths are not interchangeable and the plan must keep them distinct in both respects: the playlist path carries `spotify:track:` URIs in batches of 100, the Liked Songs path carries bare IDs in batches of 50. Getting one identifier format into the other path is the kind of mistake that fails loudly, which is fine; getting a batch size wrong is the kind that fails only on the user's larger sessions, which is not. Both are cheap to pin down in Vitest with synthetic staged changes sized to straddle the boundaries — 99, 100, 101 for the playlist path, 49, 50, 51 for saved tracks.

The planner also fixes the global ordering: every add batch is emitted before any remove batch. This is not an optimization, it is the safety property. Adds create the second copy; removes destroy the first. Doing them in that order means an interruption at any point leaves a duplicate rather than a hole.

**Gating removals.** Each song's staged removal carries a dependency on that song's own adds. The runner tracks, per song, whether every add batch that song appeared in reported success; a removal is issued only if they all did. Because batches mix songs, a single failed add batch withholds the removals of every song in it and no others — the gate is per song, not per batch and not global. The important subtlety, and the one worth a test of its own: a song staged for removal with no adds at all has an empty dependency set, and an empty set is satisfied. The user who marks a song for removal without filing it anywhere is making a deliberate choice, and the gate must not read the absence of adds as a failed add. Test both shapes — removal-with-adds where an add fails, and removal-with-no-adds — so a later refactor cannot quietly collapse them.

**Resolving ambiguous outcomes.** A request that returns a clear success or a clear rejection needs no thought. The dangerous case is the one where the app does not know: a timeout, an aborted connection, a response that never arrives. The write may or may not have landed, and both guesses are wrong somewhere. Resolve it with a single verification request rather than an assumption. For a playlist write, read the playlist's `total` and compare it against the value captured immediately before the write; for the Liked Songs path, use the saved-tracks contains endpoint, which answers the question directly for the specific IDs in the batch. `ROADMAP.md` records the caveat on the `total` comparison: it is sound only while the user is not editing playlists elsewhere, an assumption v5 exists to remove. If the verification request itself fails, do not escalate to another verification — fall back to retrying the write. Retrying a write that already landed produces a duplicate, and a duplicate is an acceptable outcome where a lost track is not. That ranking decides every ambiguous case in this runner, and it should be legible in the code's structure rather than living only in this document.

**Holding a failed commit.** When the run cannot continue — connectivity lost, retries exhausted — stop and hold. The staged changes stay in `localStorage`, minus whatever demonstrably completed, so pressing commit again resumes rather than redoing. Nothing is discarded on failure. `ROADMAP.md` notes iOS has no Background Sync, so there is no silent retry to build: the app surfaces that the commit is incomplete and waits for the user to press commit again the next time they are active. The UI for this can be plain — a message and the same commit control, still enabled — but it must not present a failed commit as a finished one.

**The accepted consequence.** Playlist removal by URI removes every occurrence of that track unless explicit positions are supplied, so a removal issued after a duplicate-producing retry can delete more copies than the user staged. v1 accepts this; it is the same duplicate-tolerance tradeoff seen from the other end, and supplying positions would mean tracking indexes that shift under every preceding write. Do not add position tracking here.

**Wiring it up.** The commit control lives on the triage screen from `03-triage.md`. It reports progress as batches complete, disables itself while a run is in flight so a double press cannot start two runners against the same staged changes, and on clean completion clears the applied records and returns the screen to an empty staged state.

## Working-state check

Run the app with `npm run dev` and complete a small real session end to end: pick a Playlist A, pick a comparison group of one or two playlists, triage two or three songs into targets with at least one of them also marked for removal, and commit. The commit control reports progress and finishes. Opening the Spotify client shows those songs present in their new playlists and absent from Playlist A, matching what was staged.

Then run the failure case, which is one of v1's three "Done when" criteria. Stage a batch large enough to span more than one request, induce a failure part way through the run, and inspect the result in the Spotify client: every song in the session is in its original playlist, or in its new one, or in both. No song is in neither. Press commit again once the fault is cleared and confirm the run completes and the staged changes clear.

## Deliberately absent

**Snapshot revalidation.** The runner does not check whether the playlists it is about to write have changed since the fetch. `ROADMAP.md` assigns that to v5, which depends on the `snapshot_id` tracking v4 introduces. Until then the app assumes the user is not editing playlists in the Spotify client mid-session — the same assumption the `total` comparison rests on.

**Undo.** There is no reversal of a committed change. Staged changes are the undo point; once committed, the Spotify client is where a mistake gets fixed.

**Explicit-position removal.** Not built, per the accepted consequence above.

**Background retry.** A held commit waits for the user. No service worker sync, no polling for connectivity.

## Open items

**How the deliberate mid-commit failure is induced.** Two candidates. Toggling offline in devtools part way through a run is zero-effort but hard to land at the same point twice, which makes it poor for confirming a fix. An injected fault in the runner — a way to make the Nth request fail on demand — is repeatable, lands exactly where it is aimed, and would let the failure case be exercised without a live session at all. If the injected fault is built, decide in-block whether it earns a permanent place as test scaffolding rather than being removed after the check; a runner this consequential arguably wants a standing way to prove the gate holds. Whichever is chosen, the manual check in the Spotify client still runs — it is the criterion `ROADMAP.md` names.

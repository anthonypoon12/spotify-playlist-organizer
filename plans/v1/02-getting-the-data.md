# 02 — Getting the data — scan a playlist group

**Date:** 2026-08-08
**Implements:** ROADMAP v1 — Find and file
**Status:** planned

## What this block delivers

At the end of this block the app is a working scanner. The user logs in as `01-plumbing.md` already allows, lands on a session-setup screen listing every playlist their account can see, picks **Playlist A** — one of their own playlists, or Liked Songs — multi-selects a **comparison group** from the same list, chooses how many songs to pull from Playlist A, and starts the scan. The app then fetches: Playlist A subject to the cap, and every comparison-group playlist in full. While it runs, the user watches live progress and a time estimate that starts from a conservative guess and converges on the truth as real throughput is measured, and can cancel at any point. When it finishes, a summary screen reports what was fetched — each playlist by name with the number of tracks pulled from it, and the totals.

The tracks land in memory in a typed shape suitable for the matching work in `03-triage.md`, but nothing in this block interprets them. The app reads and reports; the answer to "which songs are unfiled" is not computed here, and no write of any kind happens anywhere in v1 before `04-commit.md`.

## New concepts

**React's state model in earnest.** `01-plumbing.md` needed only local component state; this block is the first where several screens have to agree about the same facts. Two React patterns cover it. *Lifting state up* means that when two sibling components both depend on a value, the value lives in their nearest common parent and is passed down, rather than being duplicated in each — here, the source picker and the comparison-group list both care about which playlists exist and which are chosen, so that selection lives above both. *Context* is React's way of making one value readable anywhere below a point in the tree without threading it through every intervening component; this project uses exactly one, the session Context, which holds the configured session — Playlist A, the comparison group, the cap — and later the fetched tracks. `ROADMAP.md` settles that these built-ins are the whole state story: no external store.

**CSS Modules.** A styling approach where a stylesheet is imported by the component that uses it and its class names are rewritten at build time to be unique, so a `.selected` rule in the playlist list cannot collide with a `.selected` rule anywhere else. This block is the first with real UI, so it is where the convention gets established: one stylesheet per component, colocated with it.

**TypeScript generics.** A generic is a type that takes a type as a parameter, the way a function takes a value. The pagination helper is the concrete motivation: it walks any Spotify paged endpoint the same way — read a page, follow `next`, stop when `next` is null — but the items it yields are playlists in one case and playlist tracks in another. Written generically, the helper is one piece of code and each caller still gets back an array of the right item type with no casting, which is what makes `strict` and `noUncheckedIndexedAccess` worth having downstream.

## Steps

### Step 4 — Pagination helper and the playlist list

Two pieces land together, because the helper has no reason to exist without a consumer and the consumer cannot be written without the helper.

The first is a generic pagination helper that walks a Spotify paged endpoint to exhaustion. Spotify's paged responses carry `items`, `total`, `limit`, `offset`, and a `next` URL that is null on the last page; the helper follows `next` until it runs out and accumulates the items. Every request it makes goes through the rate-limit-aware API client from `01-plumbing.md`, so `429` handling and `Retry-After` waiting are inherited rather than reimplemented — this helper's only job is the walk. It is parameterized by item type so a caller asking for playlists gets playlists back. It also needs to report progress and to stop early: both the fetch engine in step 6 and any future caller want to know how many items have arrived so far and want the ability to abandon the walk mid-flight, so the helper accepts a per-page callback and an `AbortSignal` rather than being a black box that only returns when complete. Request the largest page size each endpoint allows, since reads dominate the cost.

The second is its first consumer: a call that walks `/me/playlists` to exhaustion and produces the account's full playlist list. Each entry keeps the fields later steps need — `id`, `name`, `owner`, `tracks.total`, `snapshot_id` (unused in v1 but free to carry, and v4 is built on it) — plus a derived flag for whether the user owns the playlist, computed by comparing the playlist's `owner.id` against the current user's own ID, which `01-plumbing.md` already fetches for the profile display. That flag is load-bearing: `ROADMAP.md` records that followed playlists are readable but not writable, so ownership is what separates a playlist that can be a **target playlist** from one that can only ever be compared against.

This step is testable without UI. Vitest coverage belongs on the helper against a fake paged endpoint: a single-page response, a multi-page walk, an empty result, and an abort partway through.

To keep the app working at the end of the step, the playlist list is fetched after login and rendered as a plain read-only list — names, track counts, and a marker on the ones the user does not own. Nothing is selectable yet.

### Step 5 — Session setup

This step turns that read-only list into the configuration screen, and introduces the session Context that holds the result.

The screen has three parts. The **source picker** chooses Playlist A: a single-select over the user's own playlists, with Liked Songs offered alongside them as a first-class option even though it is not a playlist and is read through `/me/tracks` rather than a playlist endpoint. Playlists the user merely follows are not offered as sources — the fetch would work, but Playlist A is the thing being triaged and the triage in `03-triage.md` may stage removals from it, which a followed playlist cannot accept.

The **comparison-group multi-select** runs over the full playlist list, followed playlists included, since a song filed into a followed playlist is still filed. Followed playlists carry a read-only marker, and the screen carries the visible note `ROADMAP.md` requires: that Spotify forbids writing to playlists the user does not own, so these count toward whether a song is already filed but cannot be filed into. The note is not decoration — its purpose is that a read-only playlist reads as a platform limitation rather than as a bug in this app. Playlist A is excluded from the comparison group once chosen, for the same reason it is not a target: comparing a playlist against itself answers nothing.

The **fetch cap** applies to Playlist A only, offered as a small set of presets plus a custom value, with an option for no cap at all. Alongside it sits a first, static time estimate derived from a conservative default throughput and the number of pages implied by the cap and the comparison group's total track counts, which the playlist list already provides. Step 6 makes that estimate self-correcting; here it is a fixed number so the user sees the cap's cost before committing to it.

All three selections live in one place. Rather than scattering `useState` across the three components, the configured session is a single object — source, comparison group, cap — held in a `useReducer` at the screen's root, with the actions being the user's gestures: pick a source, toggle a playlist, set the cap. That object is what the session Context publishes, and step 6 reads it from there rather than receiving it as props through a chain of components. The Start button is disabled until the session is coherent — a source chosen and at least one comparison playlist selected — so no invalid configuration is reachable.

Because step 6 does not exist yet, the Start button is present but performs no fetch; it is disabled with the configuration still fully usable, so nothing reachable errors. This is the step where the CSS Modules convention gets set, since the selection states — selected, read-only, disabled — are exactly the stateful selectors that motivated the choice.

### Step 6 — The fetch engine

The fetch engine takes the configured session from the Context and runs it.

Playlist A is fetched subject to the cap, through `/me/tracks` when the source is Liked Songs and through the playlist items endpoint otherwise. Every comparison-group playlist is fetched in full, one after another. That asymmetry is the whole point and `ROADMAP.md` gives the reason: a partial fetch of a comparison playlist does not yield a partial answer, it yields a wrong one, because a song sitting in the unread tail of a comparison playlist would be reported as unfiled. So the cap is a Playlist A concept exclusively and the engine must have no path that applies it elsewhere. The requested fields are narrowed to what matching in `03-triage.md` needs — track `id`, `name`, artists, `external_ids.isrc`, `linked_from`, `is_local`, `type`, and the item's `added_at` — both to cut response size and because `ROADMAP.md` specifies that local files and podcast episodes are skipped for lack of a usable identifier, which requires the fields that identify them.

Playlists are fetched sequentially rather than in parallel. With an unpublished request quota inside a rolling window, parallelism buys speed at the cost of provoking the `429`s that the client then has to wait out, and a sequential crawl also produces the clean per-playlist progress the next paragraph depends on.

Progress is live. The screen shows which playlist is currently being fetched, how far through the overall work the run is, and a remaining-time estimate. The estimate is the self-correcting one `ROADMAP.md` calls for: it starts from the conservative default seeded in step 5 and, once pages have actually come back, is recomputed from measured throughput — items per second observed so far — against the work remaining, which is known up front because every playlist's `tracks.total` is already in hand from step 4 and Liked Songs' total arrives with its first page. The user should see a number that converges rather than a fixed guess. Rate-limit waits are part of the measurement, not an exception to it: a run that spends time waiting out `429`s is genuinely slower, and the estimate should say so.

The run is cancellable. Cancel propagates the `AbortSignal` that step 4's helper already accepts, so the walk stops between pages rather than after the current playlist finishes, and the app returns to the configuration screen with the session intact and any partial results discarded. Discarding is deliberate — a half-fetched comparison group would produce wrong answers downstream, and there is nothing in this block that could use it.

The run ends on a scan summary screen: Playlist A with the number of tracks pulled and whether the cap bound, each comparison playlist with its count, the number of items skipped as local files or episodes, the elapsed time, and the totals. The summary is the block's terminal state; the button that would lead onward to triage does not exist yet.

Fetched tracks are held in the session Context in memory. `ROADMAP.md` puts durable storage of the mirror in v4, keyed by `snapshot_id`, so a reload here loses the fetch and the user re-runs it — acceptable and explicitly the state v4 exists to improve.

## Working-state check

Run the dev server with `npm run dev`, open the app on the loopback address, and log in. Configure a session: pick a source, select two or three playlists into the comparison group including at least one followed playlist, choose a cap, and start the scan. Watch progress advance and the time estimate move off its initial guess as the run proceeds. Let it finish and read the summary.

The check passes when each comparison playlist's reported count matches the track count the Spotify client itself shows for that playlist, and Playlist A's count equals the cap (or its true total, if the cap is larger). A small discrepancy is expected only where the summary reports skipped local files or episodes; the reported skips should account for the difference exactly. Also cancel a run partway and confirm the app returns to the configuration screen with the selections intact and no error. `npm test` passes, covering the pagination helper and the session reducer.

## Deliberately absent

- **Matching.** Nothing in this block compares Playlist A against the comparison group. Track IDs, ISRCs, and `linked_from` are fetched and stored because matching needs them, but the comparison itself is `03-triage.md`.
- **Triage UI.** No keyboard handling, no per-song focus, no **target playlists**, no **staged changes**. The summary screen is where this block stops.
- **Writes of any kind.** No adds, no removes, no saved-tracks calls. Write scopes are already granted by `01-plumbing.md`'s login, but nothing in this block uses them.
- **Persistence of the fetched mirror.** In-memory only. IndexedDB and `snapshot_id` keying are v4.
- **The all-playlists preset.** Selecting everything into the comparison group with one click is v3, deliberately ordered after the manual multi-select this block builds.
- **Parallel fetching and request-rate tuning.** Sequential is the v1 answer. Optimizing the read path further is not a v1 goal beyond narrowing requested fields and using maximum page sizes.

## Open items

**How the fetch cap selects "the most recently added" songs.** This is the block's main unresolved design question and it should be settled early in step 6, because the answer changes what step 6 fetches and in what order.

`ROADMAP.md` requires that the cap take the most recently added songs rather than the first N in playlist order, and gives the reason: the songs most likely to need filing are the ones added most recently, and for Liked Songs the two orderings differ substantially. The difficulty is that only `/me/tracks` — the Liked Songs path — returns items newest-first natively, so for that source the requirement is free. A playlist's items arrive in playlist order, which the user can reorder arbitrarily and which carries no relationship to `added_at`. Identifying the true most-recent N therefore requires reading every item's `added_at`, which is exactly the full read the cap exists to avoid.

Two candidate resolutions, to be decided in-block:

- **Approximate by taking the last N in playlist order.** Page to the end of the playlist and take the final N items, using the `total` already known from step 4 to jump straight to the right offset. Cost is minimal — a couple of requests regardless of playlist size. Correct only for playlists the user appends to and never reorders, which is the common case but not a guaranteed one; for a manually curated or sorted playlist the result is simply the wrong N songs, silently.
- **Page the playlist's metadata in full, sort by `added_at`, then fetch in detail.** Walk every page requesting only `added_at` and `track.id`, sort, take the newest N, and only then request full track detail for those. Always correct. Costs a full pass over the playlist's pages, though a cheap one — the narrow field set makes each response small, and it is still one request per page rather than per track, so the cap continues to save real work on the expensive part.

The tradeoff is exactness against a full metadata pass whose cost scales with playlist size. Note that whichever is chosen, the summary should be honest about it, and that the choice affects the time estimate's seeding since the second option adds a phase the first does not have.

**Whether the cap applies to Liked Songs and playlists identically in the UI.** Following from the above: if the two paths differ in cost or exactness, the setup screen may need to say so when the source is a playlist rather than Liked Songs. Deferred until the resolution above is picked.

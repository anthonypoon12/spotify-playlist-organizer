# ROADMAP.md

Roadmap for **spotify-playlist-organizer**. This document is the reference a fresh session reads before touching code: what the app is for, what constrains it, and what each version delivers. Scope decisions are recorded here so they are not re-litigated.

## Ultimate goal

A personal web app for triaging a music library. It surfaces the songs that live in one playlist and nowhere else across a chosen group of other playlists, lets the user file those songs into other playlists quickly, and doubles as a remote control for the Spotify client so the user can listen to each song while deciding where it belongs.

This is explicitly personal-use software. It is not a product, it will not be distributed publicly, and it does not need to accommodate users other than its author.

## Constraints and assumptions

These are the facts that shape every version. They were expensive to establish and would be painful to rediscover.

- **The Spotify Web API is free.** There is no billing and no paid tier. The only cost is rate limiting, enforced by `429` responses against a rolling window of roughly 30 seconds. The window is documented; the request quota within it is not. With no published quota to budget against, the app must handle `429` responses by respecting `Retry-After` rather than by pre-computing a safe request rate.
- **Playback control requires Spotify Premium.** Every playback endpoint is Premium-only. This is acceptable — the user has Premium — but it means playback features cannot be assumed to work for anyone else and should degrade rather than crash.
- **Development-mode apps are capped at 25 manually-added users.** Each user must be added by hand in the Spotify developer dashboard. This is fine for personal use and permanently rules out public distribution without a quota-extension request.
- **Redirect URIs must use the explicit loopback IP address**, not the hostname `localhost`. Spotify rejects `localhost` redirect URIs. Development and production redirect URIs must both be registered in the dashboard.
- **The scope set is known from the outset.** Reading needs `playlist-read-private`, `playlist-read-collaborative`, and `user-library-read`. Writing needs `playlist-modify-private`, `playlist-modify-public` (target playlists may be either, so both are required), and `user-library-modify` for removing from Liked Songs. Playback in v2 adds `user-read-playback-state` and `user-modify-playback-state`. Request the read and write scopes from v1 so the user is not re-prompted mid-project.
- **Playlists the user follows but does not own cannot be written to.** The API will not accept adds or removes against them. They can still be read, and they still count when deciding whether a song has already been filed somewhere.
- **Write endpoints have different batch limits.** Playlist adds and removes accept up to 100 track URIs per request. Saved-tracks writes — the Liked Songs path — accept up to 50, and they take bare track IDs rather than URIs. The two paths are not interchangeable.
- **Reads dominate the cost, not writes.** Building the local mirror of every track in every selected playlist is hundreds of requests. Committing a session's changes is a couple dozen requests at most. Optimization effort belongs on the read path; the write path is cheap and should prioritize correctness over efficiency.
- **Installing the app to an iOS home screen puts it in a separate storage partition from Safari.** A standalone launch does not share `localStorage`/`sessionStorage` with a normal Safari tab at the same origin, and navigating to Spotify's cross-origin authorization page from a standalone app can drop the user into Safari instead of returning to the installed app. The PKCE `code_verifier` stashed before redirect must still be readable when it returns, so login has to be tested specifically as an installed home-screen app, not just in a Safari tab.

## Stack

A static single-page app built with Vite, React, and TypeScript.

Authentication uses the Authorization Code flow with PKCE, which means there is no backend and no client secret anywhere in the codebase. The app talks to the Spotify Web API directly from the browser. The build output is static files, deployable to any static host. Being static files also makes the app installable to a mobile home screen via a web app manifest — no separate architecture is needed for that.

## Core concepts

Four terms the version descriptions rely on:

- **Playlist A** — the source being triaged. It is a single playlist, or the user's Liked Songs. The app's central question is which songs in Playlist A appear nowhere else.
- **Comparison group** — the set of playlists checked against Playlist A. The user multi-selects these. A song in Playlist A that appears in any comparison-group playlist is considered already filed; a song appearing in none of them is the app's output.
- **Target playlists** — the playlists a song can be filed *into* during triage. The targets are the comparison group itself, minus the ones that cannot be written to: followed playlists, which are readable but not writable, and Playlist A, which is the source rather than a destination. The set is deliberately the same as the comparison group because the two questions are the same question — the playlists worth checking a song against are the playlists worth filing it into. There is no separate destination picker.
- **Staged changes** — the queue of pending moves. Triage decisions are held locally and applied to Spotify only on an explicit commit, so the user can work through a long list without partial writes landing along the way.

## Versions

### v1 — Find and file

**Goal.** Deliver the complete core loop — log in, pick a source, compare it against a group, triage the leftovers, commit the result — without any playback.

**Includes:**

- **Authentication.** PKCE login and token refresh, with tokens held client-side.
- **Source selection.** Pick Playlist A from the user's own playlists or from Liked Songs.
- **Comparison group selection.** Multi-select any number of playlists. Followed playlists appear in the list and can be selected for comparison, but are marked read-only, with a visible note explaining that Spotify forbids writing to playlists the user does not own. The note exists so a read-only playlist reads as a platform limitation rather than an app bug. Read-only playlists count toward whether a song is already filed, but are not offered as targets during triage.
- **Fetch cap for Playlist A.** A configurable limit on how many songs to pull from the source, offered as presets plus a custom value. The cap takes the most recently added songs, not the first N in playlist order — the songs most likely to need filing are the ones added most recently, and for Liked Songs the two orderings differ substantially. The cap is paired with a time estimate, which seeds from a conservative default and self-corrects from measured throughput as the fetch runs, so the user sees a number that converges on reality instead of a fixed guess.
- **Comparison-group playlists always fetch in full.** A partial fetch of a comparison playlist does not produce a partial answer, it produces a wrong one — a song would be reported as unfiled when it is already filed in the untouched tail of a playlist. The cap applies only to Playlist A.
- **Matching.** Two songs match if their track IDs are equal *or* their ISRCs are equal. Either alone is sufficient; requiring both would make ISRC dead weight, and ISRC agreement is exactly the case that catches the same recording released under two catalog entries. Relinked tracks are collapsed via `linked_from` so the same recording under a different market's ID matches itself. Local files and podcast episodes are skipped — neither carries a usable identifier.
- **Keyboard-driven triage.** One song in focus at a time. Number keys toggle target playlists for that song, bound to the first nine targets; beyond nine, remaining targets are reachable by click, and a better mapping is an open question rather than a v1 requirement. A separate per-song toggle marks the song for removal from Playlist A. Removal is never automatic — filing a song elsewhere does not imply taking it out of the source. The user decides per song, and may stage a removal with no adds at all.
- **Crash-safe staging.** Staged changes persist to `localStorage` as one record per song, so a refresh, a crash, or a closed tab does not lose triage work.
- **Home-screen installability.** A web app manifest, a full set of `apple-touch-icon` sizes, and standalone-display meta tags, plus a minimal service worker that caches the app shell — JS, CSS, HTML — only, not API responses, so the shell can boot with no network. Nothing else in v1 needs to work offline from a cold start, since every other v1 feature depends on live Spotify data anyway.
- **Commit runner.** Batches at each endpoint's limit — 100 URIs per playlist add or remove, 50 IDs per saved-tracks write. Runs all adds before any removes. Each song's removal is gated on that song's own adds having succeeded, so no partial failure can remove a song from Playlist A before it exists somewhere else. A song the user deliberately staged for removal with no adds is removed as instructed; that is a choice, not a failure, and the gate does not second-guess it. Ambiguous outcomes — a timeout where the write may or may not have landed — are resolved by a single verification request rather than assumed: compare the playlist's `total` against its pre-write value, or use the saved-tracks contains endpoint for Liked Songs. The `total` comparison is sound only while the user is not editing playlists elsewhere, which is the assumption v5 exists to remove. If verification itself fails, fall back to retrying the write, accepting that a duplicate is the worst case. A duplicated track is an acceptable outcome; a lost track is not. A commit attempt that fails due to lost connectivity is held, not discarded — staged changes already persist to `localStorage`, so the user retries once connectivity returns. iOS has no Background Sync, so retry happens the next time the user is active in the app, not silently in the background.
- **Removal deletes every copy.** Playlist removal by URI removes all occurrences of that track unless explicit positions are supplied. Since the app tolerates duplicates on the write path, a later removal can delete more copies than the user staged. v1 accepts this; it is the same tradeoff, seen from the other end.

**Done when.** A real triage session moves real songs between real playlists end to end, a deliberately failed mid-commit leaves every song either in its original playlist or in its new one — never in neither, and the app installs to an iOS home screen, launches standalone, and logs in successfully as an installed app, not just in a Safari tab.

### v2 — Remote control

**Goal.** Let the user hear the song they are deciding about, without leaving the app.

**Includes.** Playback of the focused song on the user's active Spotify device: play, pause, seek, and skip. Device listing, with graceful handling of the case where no device is active — the app should say so and offer to refresh the device list, not fail silently. Playback state is polled on a slow interval, with progress ticked locally between polls rather than polled at high frequency, to keep request volume low.

**Done when.** The focused song can be played, paused, scrubbed, and skipped from the app, and the absence of an active device produces a clear message rather than a broken control.

### v3 — Liked Songs preset

**Goal.** Make the most common pass — "what have I liked and never filed anywhere?" — a single click.

**Includes.** A preset that sets Playlist A to Liked Songs and pre-selects every other playlist into the comparison group. Pre-selection is a starting state, not a lock: each playlist can be de-selected individually before the fetch runs. The preset is a faster entry into v1's multi-select, not a separate mode with its own behavior. Followed playlists are pre-selected too, since a song filed into a followed playlist still counts as filed even though the app cannot write to it.

**Done when.** One click configures a full Liked-Songs-against-everything session, and individual playlists can still be removed from the group before fetching.

**Note.** Small release. Manual multi-select from v1 remains the general path.

### v4 — Persistent cache

**Goal.** Turn a minute-long cold start into a near-instant one.

**Includes.** Move the track mirror out of memory and into IndexedDB, keyed by each playlist's `snapshot_id`. A playlist whose snapshot is unchanged is never refetched. Liked Songs has no snapshot ID, so derive one from its total count plus the `added_at` timestamp of its newest item — that pair catches every change that matters in practice, and can be read in a single request. It is a heuristic, not a proof: `added_at` has second granularity, so a remove-and-add within the same second is invisible to it. That is an acceptable miss for personal use. This cache serves a second purpose beyond cold-start speed: iOS routinely restarts a backgrounded home-screen app from scratch, wiping in-memory state. With the track mirror durable in IndexedDB rather than living only in React state, a triage session survives that restart.

**Done when.** A second session against the same playlist group completes its fetch phase without refetching any unchanged playlist, and a triage session fetched while online continues to work — browsing and staging decisions — after the app is backgrounded, reloaded, and reopened without connectivity.

### v5 — Staleness handling

**Goal.** Handle the case where the user edits playlists in the Spotify client while a triage session is open.

**Includes.** Revalidate snapshot IDs immediately before committing, and re-check only the staged operations that touch playlists whose snapshots changed. Depends on v4's snapshot tracking, which is why it comes after it.

**Done when.** A playlist edited in the Spotify client mid-session is detected at commit time, and only the affected staged operations are re-checked.

**Note.** Deliberately deferred. Versions v1 through v4 assume the user does not edit playlists elsewhere during a session — an assumption that holds for personal use and is cheap to violate occasionally.

### v6 — Fuzzy matching

**Goal.** Catch songs that are already filed under a different recording of the same track.

**Includes.** Normalized artist-and-title comparison, to catch live versions, re-recordings, remasters, and differently-tagged copies that exact ID and ISRC matching misses. Fuzzy matches are surfaced as a separate "probably already filed" bucket for the user to review, not silently excluded from the results — fuzzy matching produces false positives, and a false positive that silently hides a song is worse than one the user can glance at and dismiss.

**Done when.** Fuzzy matches appear in their own reviewable bucket, distinct from both the unfiled results and the exact matches.

### v7 — Mobile drag and drop

**Goal.** Support the originally intended mobile interaction.

**Includes.** A touch-friendly layout with drag-and-drop assignment of songs to target playlists. Keyboard triage from v1 remains the desktop path; this is an additional interaction, not a replacement.

**Done when.** A triage session can be completed on a phone by dragging songs into target playlists.

## Deferred and open

Raised during planning and consciously postponed. These are decisions, not oversights.

- **Playback** was cut from v1 so the core find-and-file loop could ship without depending on Premium-only endpoints. It is v2.
- **Fuzzy matching** was cut from v1 because exact ID and ISRC matching is correct-by-construction and fuzzy matching is not. It is v6.
- **The all-playlists shortcut** was deliberately ordered after manual multi-select, so the general mechanism exists before the convenience wrapper around it. It is v3.
- **Staleness handling** waits on the snapshot tracking introduced by the cache. It is v5.
- **Public distribution** is ruled out by the 25-user development-mode cap and is not planned.
- **A backend** is not planned. PKCE removes the need for one, and adding one would mean hosting, secrets, and deployment for a single-user app.
- **A deployment host** has not been chosen. Any static host works, but whichever is picked determines the production redirect URI that must be registered in the Spotify dashboard. The decision can wait until v1 is worth deploying; local development against the loopback IP does not need it.
- **Duplicate tracks** are an accepted worst case on the write path. When a write outcome cannot be verified, the app retries and tolerates a duplicate rather than risking a lost song.
- **Full offline-first functionality** is out of scope. A cold start with nothing previously fetched has nothing useful to do offline, since every feature needs live Spotify data. What is supported: the app shell boots without network (v1), an in-progress session survives a connectivity gap while the tab stays resident (v1) and survives an iOS-forced reload of a backgrounded tab (v4), and commits queue and retry on reconnect rather than failing outright (v1).

## Open item for the user

`CLAUDE.md`'s project-status section currently states that no stack, architecture, or command set exists. That is still literally true — this document records decisions, not code. It will need revising once v1's scaffolding is real, and that revision belongs with the v1 work rather than with this document.

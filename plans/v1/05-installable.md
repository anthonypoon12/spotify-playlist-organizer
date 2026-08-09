# 05 — Installable — home-screen app

**Date:** 2026-08-08
**Implements:** ROADMAP v1 — Find and file
**Status:** planned

## What this block delivers

At the end of this block the app is installable to an iOS home screen, and v1 is done.

Everything the app does is already built. `01-plumbing.md` through `04-commit.md` deliver the whole find-and-file loop — log in, pick Playlist A, multi-select a comparison group, fetch, triage the unfiled songs against the target playlists, and commit the staged changes — and by the time this block starts, that loop works end to end in a desktop browser. Nothing here adds a feature to it.

What this block adds is the wrapper that makes the same app behave like an installed one. The built site carries a web app manifest, a full set of `apple-touch-icon` sizes, and standalone-display meta tags, so iOS's "Add to Home Screen" produces a real icon that launches the app without Safari's browser chrome around it. Alongside that, a minimal service worker caches the app shell — the built JS, CSS, and HTML — so a cold launch renders the interface without waiting on the network, and renders it even with no network at all. API responses are not cached; `ROADMAP.md` is explicit that they should not be.

Beyond that, the block delivers confidence rather than code. v1's three "Done when" criteria are checked against the real thing — real playlists, a real induced commit failure, a real phone — and any gap that check exposes is fixed here. That verification is the point of Step 12, and it is where v1 is either finished or shown not to be.

## New concepts

**Web app manifest.** A small JSON file, linked from the page's head, that tells the operating system how to treat the site when it is installed: its name, its icons, its start URL, and whether it launches with browser chrome or standalone. `ROADMAP.md` requires it because home-screen installability is a v1 feature and a manifest is what turns a static site into something iOS will install.

**Service worker lifecycle.** A service worker is a script the browser runs separately from the page, sitting between the app and the network so it can answer requests from a cache. It moves through a fixed sequence — *installing*, where it populates its cache; *activating*, where it may clean up after older versions; and then *controlling* the pages it serves. The part that matters here is its default: a newly installed worker waits rather than taking over, and the pages already open keep being served by the previous version, so a freshly deployed build does not reach the user until every tab of the old one has been closed. On an installed home-screen app that can mean running yesterday's code indefinitely.

## Steps

### Step 11 — The PWA shell

Make the built app installable and give it a shell that boots without a network.

The manifest comes first. It declares the app's name and short name, a start URL, `standalone` display, and background and theme colors, and it points at the icon set. Because Vite is producing static files, the manifest and icons are static assets that ship alongside the bundle and the manifest is linked from the HTML entry point. The icon set is the part with real requirements rather than taste: `ROADMAP.md` calls for a *full* set of `apple-touch-icon` sizes, because iOS does not scale a single icon gracefully and a missing size gets a rendering of the page instead of an icon. The standalone-display meta tags are the iOS-specific counterpart to the manifest's `display` field — Safari has historically honored its own `apple-mobile-web-app-*` tags rather than the manifest alone, so both are declared and neither is assumed to be redundant.

The service worker follows, and it is deliberately small. It caches exactly the app shell — the built JS, CSS, and HTML entry point — at install time, and serves those from cache on later loads. It does not cache anything from the Spotify API. `ROADMAP.md` states the reason directly: every other v1 feature depends on live Spotify data, so a cached API response would at best be useless and at worst be a stale answer to the app's central question about which songs are unfiled. Requests to Spotify pass straight through to the network, and when the network is not there they fail the way they already fail today, which the app already handles.

One requirement goes beyond what `ROADMAP.md` states, and it is the reason this is a step rather than a footnote. **The cache is versioned, the worker claims control on activate, and a new build triggers a reload.** Concretely: the cache name carries a build-specific version so a new deployment writes to a new cache rather than reusing the old one; the worker skips the waiting phase and takes control of open clients as soon as it activates, deleting caches that do not match the current version; and the page, on being told that a new worker has taken over, reloads itself so the user is running the build that was just deployed. Without this, the lifecycle's default behavior described above applies — the previously cached shell keeps being served, the app silently runs yesterday's code after every deploy, and every subsequent debugging session is spent chasing a bug that was already fixed. Spending a step to avoid that is cheaper than discovering it once.

The reload needs one piece of care: it must fire once per activation, not in a loop, since a page that reloads on every controller change and re-triggers the same event on load will spin. A one-shot guard is enough, and it is worth testing that the guard holds rather than trusting it, because the failure mode is an app that never finishes loading.

Two things about the shell interact with earlier blocks and should be checked while writing it. The service worker must not intercept the OAuth return: the app detects that return from a `code` query parameter on the entry URL, so a cached response served for that URL has to still be the real entry point with its query string intact. And `04-commit.md`'s held-commit behavior — staged changes survive a lost connection and are retried when the user is next active — must still work with a worker in the middle, which it will as long as API requests genuinely bypass the cache.

This step naturally lands as two commits, and should: the manifest, icons, and meta tags first, which leaves the app installable but still network-dependent and working; then the service worker with its versioned cache, which adds the offline shell. Either commit is a legitimate stopping point with the app in a working state.

### Step 12 — End-to-end verification

Check v1 against its own "Done when" criteria, on real data and a real device, and fix whatever the check exposes.

This is verification, not construction, but it is real work: gaps found here get fixed here, and the fixes may not be small. Treat a failed criterion as the step's actual content rather than as an interruption to it.

`ROADMAP.md` states three criteria, and each is checked as written rather than approximated.

*A real triage session moves real songs between real playlists end to end.* Not a rehearsal against a scratch playlist with three songs in it — a genuine pass over a Playlist A with a real backlog, against a comparison group of several real playlists including at least one followed playlist that is readable but not writable, so the read-only marking and its exclusion from the target playlists are exercised rather than assumed. Triage a meaningful number of songs, stage both adds and removals including at least one removal with no adds, commit, and then confirm in the Spotify client that every song landed where it was staged to land. Two behaviors `ROADMAP.md` calls out are worth confirming deliberately during this pass, since both are correct-but-surprising: a playlist removal deletes every copy of a track, and a duplicate is the accepted outcome when a write cannot be verified.

*A deliberately failed mid-commit leaves every song either in its original playlist or in its new one, and never in neither.* The mechanism for inducing that failure is settled in `04-commit.md`; use it rather than inventing a second one. What this step adds is the audit afterward: for each song that was mid-flight when the commit failed, confirm by inspection in the Spotify client that it is present in its source or its destination, and that no song was removed from Playlist A without having been added somewhere first. Then confirm the recovery path — the staged changes were held rather than discarded, and retrying the commit completes the work rather than starting over or double-applying it.

*The app installs to an iOS home screen, launches standalone, and logs in successfully as an installed app, not just in a Safari tab.* Install from Safari on the phone, confirm the icon is the real one rather than a page rendering, launch it and confirm it opens without browser chrome, and then log in — from the installed app, cold, with no session already present. This is the criterion most likely to fail, and the *Open items* below explain why: `ROADMAP.md` records that an installed app has a storage partition separate from Safari's, and that the hop out to Spotify's authorization page can return the user into Safari rather than into the installed app. If the login does not survive that round trip, the fix belongs here, and it is a revision to the storage decisions made in `01-plumbing.md`.

Once all three pass, v1's status line in `ROADMAP.md` moves to `done` with this version's plan files listed, per the convention in `plans/README.md`, and `README.md` and `CLAUDE.md` are checked for anything that the finished app makes inaccurate — in particular anything describing installability or v1 as pending.

## Working-state check

Build the app with `npm run build` and deploy that output to whichever host the *Open items* question settles on. On an iPhone on any network, open the deployed URL in Safari, add it to the home screen, and confirm the icon is the app's own. Launch it from the home screen: it opens standalone, with no Safari address bar or toolbar. Log in from there and confirm the app returns to the installed app already authenticated, showing a real session rather than the logged-out screen.

Then put the device in airplane mode and launch the app again from the home screen. The shell boots — the interface renders rather than showing Safari's offline error page — and the parts that need Spotify report a network failure clearly instead of hanging or crashing. Restore connectivity and confirm the app recovers on the next action.

Locally, `npm run build` completes without TypeScript errors, `npm test` passes, and `npm run lint` reports clean. `npm run preview` serves the built output so the manifest, icons, and service worker can be inspected against a production build rather than the dev server, which is where they behave differently.

## Deliberately absent

- **Offline caching of API responses.** Explicitly out of scope per `ROADMAP.md`: only the app shell is cached. Every v1 feature needs live Spotify data, so a cached response would be stale rather than useful, and the shell booting offline is the whole of what is promised.
- **Offline functionality beyond the shell.** A cold start with nothing fetched has nothing useful to do. Surviving an iOS-forced reload of a backgrounded session is v4's persistent cache, not this block.
- **Background retry of a held commit.** `ROADMAP.md` records that iOS has no Background Sync; a held commit retries the next time the user is active in the app, which is `04-commit.md`'s behavior and is not extended here.
- **Push notifications, install prompts, and app-store packaging.** None are needed for a personal app installed once by its author.
- **Any new feature.** Nothing in v1's feature list is added or changed in this block except as a fix to something Step 12 finds broken.

## Open items

- **Where the app is deployed, and this block is probably blocked on it.** `ROADMAP.md` defers the deployment host under *Deferred and open*, on the reasoning that local development against the loopback IP does not need one. That reasoning stops holding here, for two independent reasons. The dev server is bound to `127.0.0.1`, which is the loopback address of the development machine and is not reachable from an iPhone on the same network — there is no address the phone can type. And service workers require a secure context: browsers permit them on `localhost` as a development convenience, but anywhere else they need HTTPS, so a plain-HTTP page served from a machine's LAN address will not register one even if the phone can reach it. Both criteria in Step 12's third bullet, and the entire working-state check above, therefore need a real deployed origin over HTTPS. Pick the host before starting this block. Any static host works; `ROADMAP.md` imposes no constraint beyond that.
- **The production redirect URI, which follows from the host and is a user-side prerequisite.** Whichever host is chosen determines the production redirect URI, and `ROADMAP.md` requires that both the development and production redirect URIs be registered in the Spotify developer dashboard. That registration is something only the user can do, it must happen before login can be tested on the phone at all, and Spotify matches the value as a literal string — so the URI registered has to be exactly the one the deployed app sends. This is the second prerequisite to line up before the block starts, alongside the host itself.
- **Whether `01-plumbing.md`'s storage choices survive installation.** That plan decides where the PKCE `code_verifier` is stashed across the redirect and where the refresh token lives, and defers *verification* of both to this block. `ROADMAP.md` records why: an installed home-screen app has a storage partition separate from Safari's at the same origin, and navigating to Spotify's cross-origin authorization page from a standalone app can drop the user into Safari instead of returning to the installed app. Either fact alone can break login in a way that a desktop browser never reveals — the verifier written before the redirect is read back in a different partition, or not read back at all because the return landed somewhere else. Step 12 is where this is actually exercised. If it fails, the fix is a revision to those storage decisions, which `01-plumbing.md` deliberately kept behind the single session Context so there is one place to change. Budget for that revision rather than assuming it away; it is the likeliest source of real work in Step 12.

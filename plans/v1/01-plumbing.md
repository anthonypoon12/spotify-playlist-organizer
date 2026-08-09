# 01 — Plumbing — log in and see your profile

**Date:** 2026-08-08
**Implements:** ROADMAP v1 — Find and file
**Status:** planned

**Prerequisite, before any of Step 2 can be implemented:** the user must register the app in the Spotify developer dashboard and obtain a client ID, and must register `http://127.0.0.1:5173/callback` as a redirect URI on that app. Nothing in Step 2 or Step 3 can be run or tested until both exist. `ROADMAP.md` records that Spotify rejects `localhost` redirect URIs, so the registered value has to be the explicit loopback IP, spelled exactly as above — Spotify matches the redirect URI as a literal string, and a trailing-slash or port mismatch is rejected the same way a wrong host is. Step 1 has no such dependency and can be implemented while the registration is pending.

## What this block delivers

At the end of this block the repository holds a real Vite + React + TypeScript application that the user can run locally, and that application does exactly one thing: it logs into Spotify and shows who you are. Opening the dev server presents a screen with a log-in control. Choosing it hands off to Spotify's authorization page, and returning from that page lands back in the app already authenticated, showing the account's display name and avatar fetched live from the Spotify Web API. A log-out control clears the session and returns the app to its logged-out state.

Underneath that thin surface sit the three pieces every later block builds on: a working build, test, and lint toolchain; a complete PKCE authentication flow including refresh and logout; and a single API-client chokepoint through which every Spotify request in the project will pass. The profile screen exists mainly as proof that the chokepoint works end to end — it is the smallest real request that exercises token injection, typing, and rendering together.

No playlists are fetched and nothing is written. The app is complete for what it claims to do, and everything it does not yet do is absent from the interface rather than present and broken.

## New concepts

**Vite.** The build tool and dev server this project is built on: it serves source files to the browser during development with near-instant reloads, and bundles them into static files for deployment. `ROADMAP.md` settles on it because the app is a static single-page app with no backend, which is exactly what Vite produces.

**Vitest.** The test runner, chosen because it reuses Vite's own transform pipeline, so tests see the same TypeScript and module resolution the app does with no second configuration to maintain.

**OAuth and PKCE.** OAuth is the protocol by which the user grants this app permission to act on their Spotify account without ever giving it their password; PKCE ("Proof Key for Code Exchange") is the OAuth variant designed for apps that cannot keep a secret. `ROADMAP.md` requires it because the app is pure browser-side static files — any client secret shipped in the bundle would be readable by anyone, and PKCE replaces that secret with a one-time value generated fresh for each login.

**Rate-limit-aware client design.** Spotify enforces a request budget by rejecting excess requests with a `429` status and a `Retry-After` header, rather than by publishing a quota an app could stay under. A client designed for this reacts to being told to wait instead of trying to predict the limit — `ROADMAP.md` records that the window is documented but the quota within it is not, which is why the reactive design is the only correct one here.

**Typing external API responses.** TypeScript knows nothing about data that arrives over the network; a JSON response is shapeless until the code declares what shape it expects. Writing those declarations by hand for the specific fields the app uses is what lets the compiler catch a misspelled property or an unhandled missing value, and it is the habit every later block's Spotify data depends on.

## Steps

### Step 1 — Scaffolding

Stand up the project itself: a Vite application using the React and TypeScript template, with the resulting dependency manifest, TypeScript configuration, and entry point committed as the project's first real code.

TypeScript is configured with `strict` enabled and `noUncheckedIndexedAccess` alongside it. `ROADMAP.md` settles the second flag deliberately: later blocks index track collections by ID and by ISRC, where a miss is the ordinary case rather than an error, and this flag makes the compiler type every index access as possibly-undefined so that case cannot be skipped by accident. Turning it on now, before there is code to retrofit, costs nothing; turning it on later would mean revisiting every lookup in the project.

Vitest is wired up and proven rather than assumed. That means a test script in the manifest, whatever configuration Vitest needs to share Vite's transform pipeline, and one genuinely passing test — trivial in content, but real in that it fails if the harness is misconfigured. The point is that from this commit onward, "the tests pass" is a statement about the project rather than about an empty test suite, which is what makes test-driven work possible in every block that follows.

ESLint is configured for TypeScript and React. `CLAUDE.md` states that code in this repository is never hand-formatted and that the linter decides — this step is what makes that rule actionable, since until a linter exists the rule has nothing to defer to. A lint script goes in the manifest alongside the test and build scripts.

CSS Modules are smoke-checked, not merely trusted. Vite supports them with no configuration, but "supported" and "working in this project" are different claims, so one real module file is authored, imported by a component, and its class applied to rendered markup — enough that a broken pipeline would be visible on screen rather than discovered three blocks later. This also establishes where component styles live, which the module layout in `CLAUDE.md` will describe.

The dev server is bound explicitly to host `127.0.0.1` and port `5173` in the Vite configuration, rather than left to whatever the default resolves to. This is not a preference: the browser's origin during development must match the redirect URI registered in the Spotify dashboard exactly, and a server that answers on `localhost` produces an origin Spotify will reject. Pinning both host and port in configuration means the correct origin is not something the user has to remember to type.

Finally, this step carries the documentation revision that `ROADMAP.md` flags in its closing "Open item for the user". That item notes that `CLAUDE.md`'s *Project status* section — which currently states no code, dependency manifest, or build tooling exists — goes stale the moment scaffolding is real. This is that moment. The section is rewritten to describe the project as it now is: the build, lint, and test commands, including how to run a single test file rather than the whole suite, and the module layout with the boundaries between source directories. `README.md`'s *Status* section, which says "No code yet", is corrected in the same commit. Neither file should be left claiming the repository is empty once this step lands.

### Step 2 — PKCE login

Implement the Authorization Code flow with PKCE end to end, so that the app can obtain, refresh, and discard a Spotify access token.

This step depends on the registration described at the top of this plan. The client ID obtained there is not a secret — PKCE exists precisely because browser apps cannot hold secrets — but it is environment-specific, so it belongs in a Vite environment variable read at build time rather than hard-coded in a component, with the redirect URI alongside it.

The flow has five moving parts, and it is worth implementing and testing them as pure functions before wiring any of them to a button. First, generating a `code_verifier`: a high-entropy random string, produced from the browser's crypto API. Second, deriving its `code_challenge` by hashing the verifier with SHA-256 and base64url-encoding the digest — the challenge is what gets sent to Spotify up front, and the verifier is what gets sent later to prove the same client is completing the flow it started. Third, building the redirect to Spotify's `/authorize` endpoint with the client ID, redirect URI, challenge, and requested scopes. Fourth, detecting the return: `ROADMAP.md` settles that there is no router, so the app determines it is handling an OAuth return by finding a `code` query parameter on load, and handles the error case — Spotify returns an `error` parameter instead when the user declines — as a normal outcome rather than a crash. Fifth, exchanging that code plus the stored verifier at Spotify's `/api/token` endpoint for an access token, a refresh token, and an expiry.

The scope set requested is v1's full read *and* write list, up front, at the very first login: `playlist-read-private`, `playlist-read-collaborative`, `user-library-read`, `playlist-modify-private`, `playlist-modify-public`, and `user-library-modify`. `ROADMAP.md` enumerates these and explains the reasoning — requesting them together means the user consents once, rather than being interrupted by a fresh consent screen partway through the project when `04-commit.md` first needs to write. Both playlist-modify scopes are needed because a target playlist may be public or private. Playback scopes belong to v2 and are not requested here.

Refresh and logout complete the step. Refreshing exchanges the stored refresh token for a new access token at the same `/api/token` endpoint, and is the mechanism that keeps a long triage session from dying mid-fetch; it is implemented here and called by Step 3's client. Logout discards the stored tokens and returns the app to its logged-out screen. Once the exchange succeeds, the `code` parameter is stripped from the address bar so a reload does not attempt to redeem an already-spent code.

The session — whether a token is held, and what it is — is exposed through the single React Context that `ROADMAP.md` settles on, with the rest of the app reading it rather than reaching for storage directly. That single point of access is what makes the storage question in *Open items* answerable in one place later.

The pure functions here — verifier generation, challenge derivation, authorize-URL construction, and query-parameter detection — are the natural first use of the Vitest harness proved in Step 1, and are worth writing tests for first. The parts that genuinely talk to Spotify are verified by performing a login, not by mocking the whole flow.

### Step 3 — The API client

Build the single chokepoint through which every Spotify request in this project passes, and prove it by rendering a profile.

The client is one function that takes an endpoint and options and returns typed data, and every later block calls Spotify only through it. Centralizing has three jobs. It injects the current access token into the request's authorization header, so no calling code handles tokens. It handles a `401` by invoking Step 2's refresh, then retrying the original request once — once, not in a loop, because a second `401` after a fresh token means the session is genuinely dead and the correct response is to log the user out rather than spin. And it handles a `429` by reading the `Retry-After` header, waiting that long, and then retrying. `ROADMAP.md` explains why this is reactive rather than proactive: Spotify documents the roughly-thirty-second rolling window but not the request quota inside it, so there is no number to budget against, and the only reliable signal is Spotify telling the app to wait. `02-getting-the-data.md` runs hundreds of requests through this path, which is why it is built correctly now rather than patched then.

Everything else — a network failure, a `403`, a `404` — surfaces as a clear error the caller can act on. Nothing reachable in this block should throw an unhandled failure at the user.

Proving the client means using it for a real request. The app fetches `/v1/me` and renders a profile screen showing the account's display name and avatar image. That response is typed by hand — a declared shape covering the fields actually used, not the whole schema Spotify returns — which is where the typing-external-responses habit starts. The avatar deserves attention: the images array on a Spotify profile can be empty for an account with no picture, and `noUncheckedIndexedAccess` will insist that possibility be handled, so the screen needs a defined behavior when there is no image rather than a broken one.

The retry and refresh behaviors are the interesting things to test, and they can be tested without a network by driving the client against a stubbed fetch that returns a `401` then a success, and a `429` with a `Retry-After` then a success.

## Working-state check

Run `npm run dev` and open `http://127.0.0.1:5173` in a browser. The app shows a logged-out screen with a log-in control. Choosing it redirects to Spotify's authorization page; approving there returns to the app, which now displays the account's display name and avatar, with no `code` parameter left in the address bar. Choosing log out returns the app to the logged-out screen, and logging in again works a second time.

Separately, `npm test` passes and `npm run build` completes without TypeScript errors. `npm run lint` reports clean.

## Deliberately absent

- **Playlists.** No playlist listing, no source selection, no comparison group. Those begin in `02-getting-the-data.md`.
- **Any fetching beyond the profile.** `/v1/me` is the only endpoint called. The client is built to carry more, but nothing else uses it yet.
- **The generic pagination helper.** Spotify returns long collections in pages, and a shared helper for walking them is needed — but it belongs to `02-getting-the-data.md`, where its first real consumer lives. Building it here would mean designing it against no use case.
- **A router.** `ROADMAP.md` settles that the app is one screen plus an OAuth return detected from a `code` query parameter. No routing library is added, here or later.
- **A service worker and web app manifest.** Installability is `05-installable.md`.
- **Styling beyond the smoke check.** The CSS Modules pipeline is proven with one real module; the profile screen is plain and legible, not designed.

## Open items

- **Where the PKCE `code_verifier` is stashed** between redirecting out to Spotify and returning with a `code`. It has to survive a full page navigation, which rules out memory, and it is short-lived and single-use.
- **Where the refresh token lives.** It is long-lived, and its storage choice determines whether a returning user is still logged in.

Both are decided within this block — the flow cannot work otherwise — but neither is *verified* until `05-installable.md`. `ROADMAP.md` records the constraint that decides them: an app installed to an iOS home screen runs in a separate storage partition from Safari, so a verifier written in one context may not be readable in the other, and navigating to Spotify's cross-origin authorization page from a standalone app can drop the user into Safari on the way back. A choice that works perfectly in a desktop browser can therefore still fail as an installed app. Choose a storage mechanism here on the merits, keep the choice behind the session Context so it is reachable in one place, and expect `05-installable.md` to test it specifically as an installed home-screen app and revise it if it does not hold.

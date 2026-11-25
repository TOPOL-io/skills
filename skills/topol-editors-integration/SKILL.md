---
name: topol-editors-integration
description: 'Integrate the Topol Email Editor or Landing Page Editor into an app for the first time, on the `@topol.io/editor*` 1.x npm packages, `^1` (core, -react, -vue, -svelte; Email Editor v4, Landing Page Editor v1). Use when the user wants to add, embed, install, or set up a Topol editor / drag-and-drop email builder / landing page builder, asks "how do I integrate Topol", or wants a Topol editor screen in a React, Next.js, Vue, Svelte, or plain JS/TS app. For apps already on `@topol.io/editor*` 0.x or a `1.0.0` prerelease, use topol-editors-upgrade-v1 instead.'
---

# Integrate a Topol editor (`@topol.io/editor*` 1.x)

> **Versions.** `1.x` here is the npm major of `@topol.io/editor*` (`^1`, the
> `latest` dist-tag). It embeds Email Editor **v4** and Landing Page Editor
> **v1**. `0.3.0` and the `1.0.0` prereleases (`1.0.0-alpha.*`) have a different
> API; for apps on those, use `topol-editors-upgrade-v1`.

Add the Topol **Email Editor**, the **Landing Page Editor**, or both to a host
application. Always integrate through the npm packages; the CDN script tag is a
fallback for pages with no build step (see `references/vanilla.md`).

## Pick the package before writing any code

One core package plus a per-framework wrapper. Use the wrapper that matches the
host app — never hand-roll a React/Vue/Svelte component around the core package.

| App | Package | Editors available |
| --- | --- | --- |
| React 18/19, Next.js, Remix, Vite+React | `@topol.io/editor-react` | Email + Landing Page |
| Vue 3 (`^3.2.25`) | `@topol.io/editor-vue` | Email + Landing Page |
| Svelte 5 (`^5.0.0`), SvelteKit | `@topol.io/editor-svelte` | Email only — use the core package for Landing Page |
| Plain JS/TS, Angular, Vue 2, Svelte 4, no framework | `@topol.io/editor` | Email + Landing Page |

The wrappers depend on `@topol.io/editor` and pull it in themselves. Do not
install both a wrapper and a differently-versioned core.

## Procedure

1. **Detect the app.** Read `package.json` for the framework and its major
   version, and detect the package manager from the lockfile
   (`pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn, `package-lock.json` → npm,
   `bun.lockb` / `bun.lock` → bun). Use the app's own package manager for every
   install.

   If the app already imports `@topol.io/editor*` at 0.x or at a `1.0.0`
   prerelease (`1.0.0-alpha.*`), this is an upgrade, not an integration — stop
   and use the `topol-editors-upgrade-v1` skill.

2. **Check that the registry serves 1.x.** The stable line is `^1` on the
   `latest` dist-tag:

   ```bash
   npm view @topol.io/editor-react dist-tags   # latest must be 1.0.0 or later
   ```

   If `latest` is below `1.0.0` (a private mirror or a stale cache can still
   serve `0.3.0`, which has the old API), stop and tell the user instead of
   installing a prerelease.

3. **Install the package** with the app's package manager:

   ```bash
   npm install @topol.io/editor-react   # example: React app
   npm ls @topol.io/editor                 # must show a single 1.x entry
   ```

   If the app also needs the core package directly (for example the Landing Page
   Editor in Svelte), install `@topol.io/editor` too and check that
   `npm ls @topol.io/editor` still shows one version.

4. **Read the reference for the target framework** — write the integration from
   it, not from memory or older examples. Older docs, blog posts and the
   prereleases show APIs that no longer exist: `TopolPlugin.init({ id, callbacks })`,
   positional callback arguments, the `TopolEditor` component, Svelte `on:`
   events.
   - `references/react.md` — `@topol.io/editor-react`, including Next.js/SSR
   - `references/vue.md` — `@topol.io/editor-vue`
   - `references/svelte.md` — `@topol.io/editor-svelte` (+ core for LPE)
   - `references/vanilla.md` — core `@topol.io/editor`, and the no-build CDN path
   - `references/options.md` — shared config, callbacks, instance methods,
     stage/loader behaviour, nonce authentication and Cloud

5. **Wire the API key from configuration, never a literal.** Read the key from
   the app's existing env mechanism (`import.meta.env.VITE_*`,
   `process.env.NEXT_PUBLIC_*`, …) and add the variable to `.env.example` if the
   project has one. If no key is available, scaffold the integration with the
   env var wired up and tell the user where to put the key — do not invent a
   placeholder that looks real.

   `authorize` is `{ apiKey, entityId }`. `entityId` names the account the editor
   works for (it scopes the image library and counts as one billed user), so
   derive it from the signed-in user or tenant. It must be a non-negative integer
   or match `[A-Za-z0-9_-]+`; `init` rejects anything else. The old `userId`
   key is no longer read.

6. **Give the editor a sized container.** The editor fills its container and
   renders an iframe; a container with no height renders as a blank page. Every
   wrapper component (Email and Landing Page) renders a plain `div` of
   `width: 100%; height: 100vh` with no id. Override it with React
   `className` / `style`, Vue `class` / `style`, or Svelte `class` / `style`.
   With the core package the container is yours: give it a height before
   `render()`.

7. **Add nonce authentication — the recommended setup.** With the API key
   alone the editor runs on a key that sits in the page and never expires. With
   a `getNonce` callback it runs on short-lived nonces that the app's backend
   signs for the signed-in user with the Secret key. Follow *Nonce
   authentication* in `references/options.md`, which has the signing code.

   - **The repo has a server you can add a route to** (Next.js route handler,
     SvelteKit / Nuxt / Remix server route, an Express or similar API in the
     same repo): add the nonce endpoint and wire `getNonce`. The endpoint must
     answer only signed-in users and take `entityId` from the session, never
     from the request. Read the Secret key from a server-only env var
     (`TOPOL_SECRET_KEY`; never `NEXT_PUBLIC_*`, `VITE_*` or any other variable
     that ships to the browser) and add it to `.env.example`.
   - **You cannot tell how the app identifies the signed-in user:** ask. Do
     not ship an endpoint that signs a nonce for whoever calls it.
   - **The repo has no server** (a static SPA whose backend lives elsewhere):
     wire the API key only, do not add `getNonce`, and report nonce
     authentication as the recommended next step with what the backend team
     needs to build. A `getNonce` without a working endpoint stops the editor
     from starting.
   - **The user declined it or asked for the simplest setup:** skip it and say
     so in the report.

8. **Verify.** Run the project's own checks and report the real output:

   ```bash
   npx tsc --noEmit    # or vue-tsc / svelte-check, plus the project's lint / build / test scripts
   ```

9. **Report** which package and version you installed, which editor(s)
   you wired, where the key comes from, whether nonce authentication is in
   place (and if not, why), and the manual steps below.

## Cannot be automated — always flag these

- **The API key is public.** It ships to the browser by design. Any storage the
  editor talks to must be secured server-side via the `api` and
  `apiAuthorizationHeader` options (`references/options.md`). Nonce
  authentication (step 7) is the recommended setup; if you could not add it,
  say so. Do not let the user assume the key is a secret.
- **The Secret key.** Only the user can copy it from their Topol account into
  the server's environment. Never ask for it in chat and never write it into a
  file that is committed.
- **Persisting templates.** `onSave` hands you `{ json, html }`; where they get
  stored is an application decision. Wire the callback (returning its promise)
  and leave a clearly marked TODO rather than inventing an endpoint.
- **Runtime smoke test.** Nothing here proves the editor loads — the loader
  script is fetched at runtime and rejects an invalid key in the browser. Tell
  the user to open the page once and check that `onInit` fires and the console
  is clean.
- **Account and domains.** The API key must be enabled for Email Editor v4, and
  the app's domains must be on the token's Allowed domains list. A page with a
  Content Security Policy must allow `v4.email-assets.topol.io` (Email Editor)
  and `v1.page-assets.topol.io` (Landing Page Editor) in `script-src` and
  `frame-src`.

## Rules

- Never edit files under `node_modules/`.
- Never copy an API key out of another project, example, or this repo's docs.
- Never put the Secret key (`sk_…`) in browser code, a public env var, or the
  repository, and never sign nonces in the browser.
- Do not upgrade unrelated dependencies while installing Topol.
- Do not add a CDN `<script>` tag to an app that has a bundler — use the package.
  The package loads the editor script itself; the Email Editor rejects a
  host-inserted one.
- Import `EmailEditor` and `LandingPageEditor` by name. `TopolPlugin`,
  `TopolEditor` and the default exports no longer exist.
- Never name a handler `onError` or `onClose`; they never fire. Use
  `onEditorError` / `onEditorClose`.
- Do not write names from before 1.0; they are removed, so options and
  callbacks are ignored and methods throw: `userId`,
  `onEdittedWithoutSaveChanged`, `updateTestingEmailAddresses`,
  `onBlocksAndStructuresVisibilityChange`, `toggleControlPanel`,
  `hideControlPanel`. `references/options.md` lists the current ones.

---
name: topol-editors-integration
description: 'Integrate the Topol Email Editor or Landing Page Editor into an app for the first time, on the `@topol.io/editor*` 1.x npm packages from `1.0.0-alpha.10` (core, -react, -vue, -svelte; Email Editor v4, Landing Page Editor v1). Use when the user wants to add, embed, install, or set up a Topol editor / drag-and-drop email builder / landing page builder, asks "how do I integrate Topol", or wants a Topol editor screen in a React, Next.js, Vue, Svelte, or plain JS/TS app. For apps already on `@topol.io/editor*` 0.x or an early 1.x alpha, use topol-editors-upgrade-v1 instead.'
---

# Integrate a Topol editor (`@topol.io/editor*` 1.x)

> **Versions.** `1.x` here is the npm major of `@topol.io/editor*`, from
> `1.0.0-alpha.10` (`@topol.io/editor-react` `1.0.0-alpha.11`). It embeds Email
> Editor **v4** and Landing Page Editor **v1**. `latest` (`0.3.0`) and the earlier
> 1.x alphas use the older Email Editor v3 API; for apps on those, use
> `topol-editors-upgrade-v1`.

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
| Svelte 4 (`^4.2.8`) | `@topol.io/editor-svelte` | Email only — use the core package for Landing Page |
| Plain JS/TS, Angular, Vue 2, no framework | `@topol.io/editor` | Email + Landing Page |

The wrappers depend on `@topol.io/editor` and pull it in themselves. Do not
install both a wrapper and a differently-versioned core.

## Procedure

1. **Detect the app.** Read `package.json` for the framework and its major
   version, and detect the package manager from the lockfile
   (`pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn, `package-lock.json` → npm,
   `bun.lockb` → bun). Use the app's own package manager for every install.

   If the app already imports `@topol.io/editor*` at 0.x or below
   `1.0.0-alpha.10`, this is an upgrade, not an integration — stop and use the
   `topol-editors-upgrade-v1` skill.

2. **Resolve the 1.x version to install.** 1.x is a prerelease on the `alpha`
   dist-tag; `latest` still points at `0.3.0` (old API), so a bare
   `npm install @topol.io/editor` gives you the wrong one. Each package has its
   own number (editor-react is one ahead), so resolve the package you install:

   ```bash
   npm view @topol.io/editor-react dist-tags   # alpha must be 1.0.0-alpha.11+ (core, -vue, -svelte: 1.0.0-alpha.10+)
   ```

   If `alpha` shows `2.0.0-alpha.x`, the 1.x v4 release is not published yet:
   stop and tell the user. Tell the user which prerelease you installed and that
   details can still change before the final 1.0.

3. **Install, pinned exactly** (no `^` — caret ranges across prerelease tags
   resolve unpredictably):

   ```bash
   npm install @topol.io/editor-react@<alpha version>    # example: React app
   npm ls @topol.io/editor                              # must show a single entry
   ```

   If the app also needs the core package directly (for example the Landing Page
   Editor in Svelte), pin it to the version the wrapper depends on
   (`npm view @topol.io/editor-svelte@<version> dependencies`).

4. **Read the reference for the target framework** — write the integration from
   it, not from memory or older examples. The v3-era API (`TopolPlugin.init({ id,
   callbacks })`, positional callbacks) found in older docs and blog posts does
   not work with these versions.
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
   placeholder that looks real. Use a URL-safe `userId` (integer or
   `[A-Za-z0-9_-]+`); `init` rejects anything else.

6. **Give the editor a sized container.** The editor fills its container and
   renders an iframe; a container with no height renders as a blank page. The
   Email Editor wrappers render `width: 100%; height: 100vh` by default (React
   `style`/`className` and Vue `class`/`style` override it; size Svelte's parent).
   The Landing Page Editor wrappers render an absolutely-positioned full-viewport
   div, so mount them on a route/page that can own the whole viewport.

7. **Verify.** Run the project's own checks and report the real output:

   ```bash
   npx tsc --noEmit    # plus the project's lint / build / test scripts
   ```

8. **Report** which package and prerelease version you installed, which editor(s)
   you wired, where the key comes from, and the manual steps below.

## Cannot be automated — always flag these

- **The API key is public.** It ships to the browser by design. Any storage the
  editor talks to must be secured server-side via the `api` and
  `apiAuthorizationHeader` options (`references/options.md`), or by nonce
  authentication with a server-side `sk_` secret. Say this out loud; do not let
  the user assume the key is a secret.
- **Persisting templates.** `onSave` hands you `{ json, html }`; where they get
  stored is an application decision. Wire the callback (returning its promise)
  and leave a clearly marked TODO rather than inventing an endpoint.
- **Runtime smoke test.** Nothing here proves the editor loads — the loader
  script is fetched at runtime and rejects an invalid key in the browser. Tell
  the user to open the page once and check that `onInit` fires and the console
  is clean.
- **Account.** The API key must be enabled for Email Editor v4.
- **Prerelease pin.** The user owns the decision to ship a prerelease to
  production. Flag it.

## Rules

- Never edit files under `node_modules/`.
- Never copy an API key out of another project, example, or this repo's docs.
- Do not upgrade unrelated dependencies while installing Topol.
- Do not add a CDN `<script>` tag to an app that has a bundler — use the package.
  The package loads the editor script itself and rejects a host-inserted one.
- Use the modern names (`EmailEditor`, `LandingPageEditor`). `TopolPlugin` and
  `TopolEditor` are deprecated naming aliases of the same API.
- Never name a handler `onError` or `onClose`; they never fire. Use
  `onEditorError` / `onEditorClose`.

---
name: topol-editor-integration
description: Integrate the Topol Email Editor or Landing Page Editor into an app for the first time, on the `@topol.io/editor*` 1.x npm packages (core, -react, -vue, -svelte; Email Editor v3 runtime, Landing Page Editor v1). Use when the user wants to add, embed, install, or set up a Topol editor / drag-and-drop email builder / landing page builder, asks "how do I integrate Topol", or wants a Topol editor screen in a React, Next.js, Vue, Svelte, or plain JS/TS app. For apps already on `@topol.io/editor*` 0.x, use topol-editor-upgrade-v1 instead.
---

# Integrate a Topol editor (`@topol.io/editor*` 1.x)

> **Versions.** `1.x` here is the npm major of `@topol.io/editor*`. It embeds
> Email Editor **v3** and Landing Page Editor **v1**. `@topol.io/editor*` 2.x
> (Email Editor v4) is a different API; see `topol-editor-upgrade-v2`.

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

   If the app already imports `@topol.io/editor*` at 0.x, this is an upgrade,
   not an integration — stop and use the `topol-editor-upgrade-v1` skill.

2. **Resolve the 1.x version to install.** 1.x is a prerelease. The npm
   `latest` tag still points at 0.3.0 and the `alpha` tag has moved on to
   2.0.0-alpha.x, so neither `npm install @topol.io/editor` nor `@alpha`
   gives you 1.x. Always list the published versions first:

   ```bash
   npm view @topol.io/editor versions --json
   ```

   Install the newest `1.0.0-alpha.N` in that list — as of this writing
   **`1.0.0-alpha.9`**, but check, do not assume. 1.x alphas have introduced
   no breaking changes between each other. Tell the user which prerelease you
   installed and that the 1.x API shape is stable but details can still change.

3. **Install.** Every `@topol.io/*` package must be on the exact same version:

   ```bash
   npm install @topol.io/editor-react@1.0.0-alpha.9    # example: React app
   ```

   Pin the exact prerelease version (no `^`) — a caret range across prerelease
   tags resolves unpredictably.

4. **Read the reference for the target framework** — write the integration from
   it, not from memory. The two editors have genuinely different APIs.
   - `references/react.md` — `@topol.io/editor-react`, including Next.js/SSR
   - `references/vue.md` — `@topol.io/editor-vue`
   - `references/svelte.md` — `@topol.io/editor-svelte` (+ core for LPE)
   - `references/vanilla.md` — core `@topol.io/editor`, and the no-build CDN path
   - `references/options.md` — shared config surface, callbacks, imperative
     methods, and the stage/loader behaviour

5. **Wire the API key from configuration, never a literal.** Read the key from
   the app's existing env mechanism (`import.meta.env.VITE_*`,
   `process.env.NEXT_PUBLIC_*`, …) and add the variable to `.env.example` if the
   project has one. If no key is available, scaffold the integration with the
   env var wired up and tell the user where to put the key — do not invent a
   placeholder that looks real.

6. **Give the editor a sized container.** The editor fills its container and
   renders an iframe; a container with no height renders as a blank page. The
   wrappers ship their own absolutely-positioned full-viewport div, so mount
   them on a route/page that can own the whole viewport, not inside a small card.

7. **Verify.** Run the project's own checks and report the real output:

   ```bash
   npx tsc --noEmit    # plus the project's lint / build / test scripts
   ```

8. **Report** which package and prerelease version you installed, which editor(s)
   you wired, where the key comes from, and the manual steps below.

## Cannot be automated — always flag these

- **The API key is public.** It ships to the browser by design. Any storage the
  editor talks to must be secured server-side via the `api` and
  `apiAuthorizationHeader` options (`references/options.md`), not by hiding the
  key. Say this out loud; do not let the user assume the key is a secret.
- **Persisting templates.** `onSave` hands you `json` and `html`; where they get
  stored is an application decision. Wire the callback and leave a clearly
  marked TODO rather than inventing an endpoint.
- **Runtime smoke test.** Nothing here proves the editor loads — the loader
  script is fetched at runtime and rejects an invalid key in the browser. Tell
  the user to open the page once and check the console.
- **Prerelease pin.** The user owns the decision to ship a prerelease to
  production. Flag it.

## Rules

- Never edit files under `node_modules/`.
- Never copy an API key out of another project, example, or this repo's docs.
- Do not upgrade unrelated dependencies while installing Topol.
- Do not add a CDN `<script>` tag to an app that has a bundler — use the package.
- Prefer the modern names (`EmailEditor`, `LandingPageEditor`). `TopolPlugin`
  and the `TopolEditor` component are deprecated 0.x aliases that log warnings;
  the only exception is Svelte, where `TopolEditor` is still the real name.

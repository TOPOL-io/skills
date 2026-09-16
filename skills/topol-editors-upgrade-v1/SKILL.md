---
name: topol-editors-upgrade-v1
description: Upgrade a host app from `@topol.io/editor*` 0.x to 1.x (npm major; the Email Editor stays on the v3 runtime, the Landing Page Editor becomes available). Covers `@topol.io/editor`, `-react`, `-vue`, `-svelte`, where the package was split into importable EmailEditor and LandingPageEditor entities. Use when the user wants to upgrade, migrate, or bump Topol editor packages to 1.x, or hits "TopolEditor is not exported" / "no default export" / ITopolOptions type errors after a Topol bump. For a first-time integration use topol-editors-integration; for 2.x (Email Editor v4) use topol-editors-upgrade-v2.
---

# Upgrade `@topol.io/editor*` 0.x → 1.x

> **Versions.** `0.x → 1.x` is the npm major of `@topol.io/editor*`. The Email
> Editor runtime stays on **v3**, so templates and callbacks keep working; the
> Landing Page Editor (**v1**) becomes importable. Moving to Email Editor **v4**
> is `@topol.io/editor*` 2.x — see `topol-editors-upgrade-v2`.

Migrate a host application that embeds the Topol editor. The v1 packages keep the
email editor working, but the entry points, some type names, and one DOM id changed.

## What changed, in one paragraph

v0.x shipped a single editor. v1 splits the package into two importable entities —
`EmailEditor` (the old editor, renamed) and `LandingPageEditor` (new) — and every
package now exports through a real entry file with **named exports only** in the
React wrapper. `TopolPlugin` / `TopolEditor` still exist as deprecated aliases, so
most apps need a small, mechanical diff rather than a rewrite.

## Procedure

1. **Detect what the app uses.** Search the project for Topol imports and the
   installed versions:

   ```bash
   grep -rn "@topol.io/editor" --include="*.{ts,tsx,js,jsx,vue,svelte,json}" . \
     --exclude-dir=node_modules --exclude-dir=dist
   ```

   Note which package(s) appear: core (`@topol.io/editor`), `-react`, `-vue`,
   `-svelte`. An app may use more than one.

2. **Read the reference for each package in use** — do not migrate from memory:
   - `references/vanilla.md` — core `@topol.io/editor` (plain JS/TS)
   - `references/react.md` — `@topol.io/editor-react`
   - `references/vue.md` — `@topol.io/editor-vue`
   - `references/svelte.md` — `@topol.io/editor-svelte`
   - `references/breaking-changes.md` — the full changed-surface list, for
     anything the per-framework files do not cover
   - `references/landing-page-editor.md` — only if the user wants to *add* the
     landing page editor (this is new functionality, not part of the upgrade; for
     a from-scratch integration prefer the `topol-editors-integration` skill)

3. **Bump the dependencies.** All Topol packages must move together — mixing a
   1.x wrapper with a 0.x core (or vice versa) breaks at runtime. Use the app's
   own package manager (check for `pnpm-lock.yaml` / `yarn.lock` / `package-lock.json`):

   ```bash
   npm i @topol.io/editor@1.0.0-alpha.9 @topol.io/editor-react@1.0.0-alpha.9   # example
   ```

   1.x is a prerelease with no dist-tag of its own (`latest` is 0.3.0, `alpha`
   is 2.0.0-alpha.x), so `^1.0.0` and `@alpha` both resolve wrong. List the
   published versions with `npm view @topol.io/editor versions --json` and pin
   the newest `1.0.0-alpha.N` (as of this writing `1.0.0-alpha.9`) on every
   Topol package. 1.x alphas have introduced no breaking changes between each
   other. Tell the user it is a prerelease.

4. **Apply the code changes** from the reference file(s). Prefer the modern names
   (`EmailEditor`) over the deprecated aliases — the aliases log deprecation
   warnings and will be removed in a later major.

5. **Verify.** Run the app's own checks and report the real output:
   ```bash
   npx tsc --noEmit    # or the project's typecheck / lint / build / test scripts
   ```
   Type errors mentioning `ITopolOptions`, `ISaveData`, or a missing default
   export are expected fallout of this migration — fix them using
   `references/breaking-changes.md`, do not silence them with `any`.

6. **Report** what changed, what was left alone, and anything the user must
   verify by hand (see *Cannot be automated* below).

## Cannot be automated — always flag these

- **CSS and DOM selectors.** The wrappers render into a new container id. If the
  app styles or queries `#editor` (React) or `#topol-editor-id` (Vue), those
  selectors must be updated to `#topol-email-editor-id`. Grep for them and list
  every hit for the user; changing a selector can silently break layout, so
  show the diff rather than assuming.
- **`onTestSend` handlers.** `email` widened from `string` to `string | string[]`.
  Any handler that passes it straight into an API call needs a real decision
  (send to all, or take the first) — ask, don't guess.
- **Runtime smoke test.** Nothing here proves the editor still loads; tell the
  user to open the editor screen once.

## Rules

- Never edit files under `node_modules/`.
- Do not "upgrade" unrelated dependencies while you are in the lockfile.
- If the app pins `@topol.io/editor` 0.x deliberately (comment, renovate config),
  stop and ask before bumping.

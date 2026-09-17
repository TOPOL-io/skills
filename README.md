<p align="center">
  <a href="https://topol.io">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://docs.topol.io/topol-logo-white.svg">
      <img src="https://docs.topol.io/topol-logo-dark.svg" alt="Topol" width="180">
    </picture>
  </a>
</p>

<h1 align="center">Topol editors — skills for AI coding agents</h1>

<p align="center">
  Integrate and upgrade the <a href="https://docs.topol.io/email-editor/guide/introduction.html">Email Editor</a>
  and <a href="https://docs.topol.io/landing-page-editor/guide/introduction.html">Landing Page Editor</a>
  from Claude Code, Codex, Cursor, Copilot and any agent that reads skills.
</p>

<p align="center">
  <a href="https://docs.topol.io">Docs</a> ·
  <a href="https://www.npmjs.com/package/@topol.io/editor">npm</a> ·
  <a href="https://docs.topol.io/changelog/about-changelog.html">Changelog</a> ·
  <a href="https://topol.io/contact">Contact</a>
</p>

---

## Skills

The Topol editors (Email Editor, Landing Page Editor, more to come) ship together in the
`@topol.io/editor*` npm packages (`editor`, `editor-react`, `editor-vue`, `editor-svelte`).
Skill names carry the **npm major**; the table maps it to the runtime of each editor.

| Skill | Purpose | npm `@topol.io/editor*` | Email Editor | Landing Page Editor |
| --- | --- | --- | --- | --- |
| [`topol-editors-integration`](skills/topol-editors-integration/SKILL.md) | First-time integration: picks the package for the framework, wires options, callbacks and the API key | 1.x | v3 | v1 |
| [`topol-editors-upgrade-v1`](skills/topol-editors-upgrade-v1/SKILL.md) | Upgrade 0.x → 1.x: split into `EmailEditor` / `LandingPageEditor` exports, renamed types and container ids | 0.x → 1.x | v3 (unchanged) | v1 (new) |
| [`topol-editors-upgrade-v2`](skills/topol-editors-upgrade-v2/SKILL.md) | Upgrade 0.x/1.x → 2.x: instance API, object callback payloads, section renames, v4 loader, optional nonce authentication and Topol Cloud | 0.x/1.x → 2.x | v3 → v4 | not affected |

Each skill covers the core package and the React, Vue and Svelte wrappers, runs the
project's own typecheck, and reports the decisions only a human can make.

> **Prereleases.** 1.x has no dist-tag (`latest` is still 0.3.0, `alpha` is 2.0.0-alpha.x).
> The skills list published versions and pin the exact one; they never trust `latest` or `^`.

## Install

Pick one path. The plugin is a managed, read-only bundle; skills.sh copies editable
markdown into your project.

**Claude Code plugin**

```
/plugin marketplace add TOPOL-io/skills
/plugin install topol-editors@topol
```

**skills.sh** (Claude Code, Codex, Cursor, Copilot, …)

```bash
npx skills@latest add TOPOL-io/skills            # install
npx skills@latest use TOPOL-io/skills@topol-editors-integration   # one-off, no install
```

**Manual.** Point your agent at `skills/<skill>/SKILL.md` and its `references/`
directory. Cursor rules, Copilot instructions and `AGENTS.md` accept the same content.

## Usage

Run inside the app that embeds the editor:

```
> add the Topol email editor to this app
> add a Topol landing page editor screen
> upgrade this app to @topol.io/editor 1.x
> upgrade this app to @topol.io/editor 2.x (Email Editor v4)
```

## Documentation

| | Email Editor | Landing Page Editor |
| --- | --- | --- |
| Getting started | [Guide](https://docs.topol.io/email-editor/guide/getting-started.html) | [Guide](https://docs.topol.io/landing-page-editor/guide/getting-started.html) |
| npm & frameworks | [NPM & Frameworks](https://docs.topol.io/email-editor/guide/js-frameworks.html) · [Next.js](https://docs.topol.io/email-editor/guide/integration-nextjs.html) | [NPM integration](https://docs.topol.io/landing-page-editor/guide/npm-integration.html) |
| Options reference | [Options](https://docs.topol.io/email-editor/reference/topol-options.html) · [Plugin API](https://docs.topol.io/email-editor/reference/topol-plugin.html) | [Options](https://docs.topol.io/landing-page-editor/reference/topol-options.html) · [Plugin API](https://docs.topol.io/landing-page-editor/reference/topol-plugin.html) |
| Callbacks | [Callbacks](https://docs.topol.io/email-editor/guide/callbacks.html) | [API](https://docs.topol.io/landing-page-editor/guide/api.html) |
| Migrations | [NPM packages v1](https://docs.topol.io/email-editor/guide/npm-v1-migration.html) · [Loader URL](https://docs.topol.io/email-editor/guide/new-topol-plugin-loader-url.html) | [Email vs. Landing Page Editor](https://docs.topol.io/landing-page-editor/guide/email-editor-vs-landing-page-editor.html) |

Packages on npm: [`@topol.io/editor`](https://www.npmjs.com/package/@topol.io/editor) ·
[`@topol.io/editor-react`](https://www.npmjs.com/package/@topol.io/editor-react) ·
[`@topol.io/editor-vue`](https://www.npmjs.com/package/@topol.io/editor-vue) ·
[`@topol.io/editor-svelte`](https://www.npmjs.com/package/@topol.io/editor-svelte)

## License

Apache-2.0 · © [Topol.io](https://topol.io)

# Topol skills

Guides and Claude Code skills for integrating and upgrading the [Topol editor](https://topol.io)
in your app.

Everything here is plain markdown. Install it as a Claude Code plugin, pull it
into any agent with [skills.sh](https://skills.sh), or hand a file directly to
your tool. The packaging below is a convenience, not a requirement.

## Available skills

| Skill | What it does |
| --- | --- |
| [`topol-editor-integration`](skills/topol-editor-integration/SKILL.md) | Integrates the Email Editor or Landing Page Editor into an app for the first time, on the v1 prerelease packages. Picks the right npm package for the host framework (React/Next.js, Vue 3, Svelte, or plain JS/TS), wires the options and callbacks, and flags what only a human can decide. |
| [`topol-v1-upgrade`](skills/topol-v1-upgrade/SKILL.md) | Migrates an app from `@topol.io/editor` 0.x to 1.x, where the package was split into importable `EmailEditor` and `LandingPageEditor` entities. Covers the core package and the React, Vue and Svelte wrappers. |

## Install

The two install paths differ in who owns the files. The **Claude Code plugin**
installs the skills as a managed, read-only bundle that updates when we ship.
**[skills.sh](https://skills.sh)** copies the skill files into your project as
editable markdown, on any agent. Pick one; installing both leaves every skill
duplicated.

### Claude Code plugin

This repo is also a Claude Code plugin marketplace:

```
/plugin marketplace add TOPOL-io/skills
/plugin install topol-editor@topol
```

Updates arrive with `/plugin update`. The files are not meant to be edited.

### skills.sh (Claude Code, Codex, Cursor, Copilot, …)

```bash
npx skills@latest add TOPOL-io/skills
```

The installer asks which skills to add and which agents to install them on
(`-a '*'` for all detected agents, `--copy` to copy the files instead of
symlinking them). The files land in your project as **ordinary markdown you own**.
Updates are manual, via `npx skills update`.

To use the skill once without installing anything:

```bash
npx skills@latest use TOPOL-io/skills@topol-editor-integration
```

### Or point your agent at the files

Hand your tool `skills/topol-editor-integration/SKILL.md` (or
`skills/topol-v1-upgrade/SKILL.md`) plus the reference for your framework from
that skill's `references/` directory. Cursor rules, Copilot instructions and
`AGENTS.md` all accept the same content with their own frontmatter.

## Usage

In the repo of the app you want to embed the editor in:

```
> add the Topol email editor to this app
> add a Topol landing page editor screen
```

The integration skill picks the package that matches the framework, installs the
v1 prerelease, and writes the component wiring.

Already on an older Topol version:

```
> upgrade this app to @topol.io/editor v1
```

The upgrade skill detects which Topol packages the app uses, bumps them, applies
the per-framework code changes, runs the project's own typecheck, and reports the
parts that need a human decision.

> **v1 is a prerelease.** The npm `latest` tag still points at `0.3.0`, so both
> skills resolve the newest version on the `alpha` tag and pin it explicitly.

## Related

- [Topol documentation](https://docs.topol.io)
- Integration packages: `@topol.io/editor`, `@topol.io/editor-react`,
  `@topol.io/editor-vue`, `@topol.io/editor-svelte`

## License

Apache-2.0

# Topol skills

Guides and Claude Code skills for integrating and upgrading the [Topol editor](https://topol.io)
in your app.

Everything here is plain markdown. Install it as a Claude Code plugin, pull it
into any agent with [skills.sh](https://skills.sh), or just hand a file to your
tool and read it yourself — the packaging below is a convenience, not a
requirement.

## Available skills

| Skill | What it does |
| --- | --- |
| [`topol-v1-upgrade`](skills/topol-v1-upgrade/SKILL.md) | Migrates an app from `@topol.io/editor` 0.x to 1.x, where the package was split into importable `EmailEditor` and `LandingPageEditor` entities. Covers the core package and the React, Vue and Svelte wrappers. |

## Install

Two ways in, two philosophies. The **Claude Code plugin** installs the skills as a
managed, read-only bundle that updates when we ship — you subscribe rather than
fork. **[skills.sh](https://skills.sh)** copies the skill files into your project,
so you can edit them and make them your own, on any agent. Pick one — installing
both leaves you with every skill twice.

### Claude Code plugin

This repo is also a Claude Code plugin marketplace:

```
/plugin marketplace add TOPOL-io/skills
/plugin install topol-editor-upgrade@topol
```

Updates arrive with `/plugin update`; you don't edit the files.

### skills.sh (Claude Code, Codex, Cursor, Copilot, …)

```bash
npx skills@latest add TOPOL-io/skills
```

Pick the skills you want and which agents to install them on (`-a '*'` for all
detected agents, `--copy` to copy the files instead of symlinking them). They
land in your project as ordinary markdown you own and can change. Nothing updates
behind your back — pull our latest with `npx skills update`.

To use the skill once without installing anything:

```bash
npx skills@latest use TOPOL-io/skills@topol-v1-upgrade
```

### Or just point your agent at the files

Everything here is plain markdown. Hand your tool
`skills/topol-v1-upgrade/SKILL.md` plus the reference for your framework in
`skills/topol-v1-upgrade/references/`. Cursor rules, Copilot instructions and
`AGENTS.md` all accept the same content with their own frontmatter.

## Usage

In the repo of the app that embeds the editor:

```
> upgrade this app to @topol.io/editor v1
```

The skill detects which Topol packages the app uses, bumps them, applies the
per-framework code changes, runs the project's own typecheck, and reports the
parts that need a human decision.

## Related

- [Topol documentation](https://docs.topol.io)
- Integration packages: `@topol.io/editor`, `@topol.io/editor-react`,
  `@topol.io/editor-vue`, `@topol.io/editor-svelte`

## License

Apache-2.0

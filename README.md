# Topol skills

Guides and Claude Code skills for integrating and upgrading the [Topol editor](https://topol.io)
in your app.

Everything here is plain markdown. You can hand a file to any coding agent, or
just read it yourself — the Claude Code packaging below is a convenience, not a
requirement.

## Available skills

| Skill | What it does |
| --- | --- |
| [`topol-v1-upgrade`](skills/topol-v1-upgrade/SKILL.md) | Migrates an app from `@topol.io/editor` 0.x to 1.x, where the package was split into importable `EmailEditor` and `LandingPageEditor` entities. Covers the core package and the React, Vue and Svelte wrappers. |

## Use with Claude Code

This repo is also a Claude Code plugin marketplace:

```
/plugin marketplace add TOPOL-io/skills
/plugin install topol-editor-upgrade@topol
```

Then, in the repo of the app that embeds the editor:

```
> upgrade this app to @topol.io/editor v1
```

The skill detects which Topol packages the app uses, bumps them, applies the
per-framework code changes, runs the project's own typecheck, and reports the
parts that need a human decision.

## Use with any other agent

Point your tool at the files directly — for example
`skills/topol-v1-upgrade/SKILL.md` plus the reference for your framework in
`skills/topol-v1-upgrade/references/`. Cursor rules, Copilot instructions and
`AGENTS.md` all accept the same content with their own frontmatter.

## Related

- [Topol documentation](https://docs.topol.io)
- Integration packages: `@topol.io/editor`, `@topol.io/editor-react`,
  `@topol.io/editor-vue`, `@topol.io/editor-svelte`

## License

Apache-2.0

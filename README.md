# kobweb-skill

An [Agent Skill](https://docs.claude.com/en/docs/claude-code/skills) that gives a coding
agent expert-level knowledge of **[Kobweb](https://github.com/varabyte/kobweb)** — the
opinionated Kotlin framework for full-stack web apps, built on Compose HTML.

The skill follows the open `SKILL.md` convention, so it works in **Claude Code**, the
**Claude apps**, and any other agent that understands the format.

## What it covers

| Reference | Topic |
| --- | --- |
| `SKILL.md` | Mental model, version baseline, non-negotiable rules, decision guide |
| `01-setup-and-build.md` | CLI, project layout, Gradle DSL, version catalog, `conf.yaml` |
| `02-routing.md` | `@Page`, dynamic/catch-all routes, `PageContext`, router, redirects |
| `03-app-layouts-init.md` | `@App`, `@Layout`, `@Init*` hooks, the per-route data store |
| `04-styling.md` | `Modifier`, `CssStyle`, variants, breakpoints, color modes, CSS layers |
| `05-silk-widgets.md` | The Silk widget set, icon packs, palettes and theme overrides |
| `06-fullstack.md` | `@Api` routes, interceptors, API streams, the client HTTP APIs |
| `07-workers.md` | Web worker modules, `WorkerFactory`, attachments |
| `08-markdown.md` | Markdown pages, front matter, embedded Kotlin, callouts, handlers |
| `09-export-and-deploy.md` | Static vs full-stack export, hosting, CI, bundle size |
| `10-interop-and-escape-hatches.md` | Compose HTML interop, raw DOM, JS libraries, own backend |
| `11-capabilities-and-roadmap.md` | **Support matrix**: what works, what doesn't, what's planned |
| `12-versions-and-upgrading.md` | Compatibility table, artifact coordinates, migrations |

The content was derived from the Kobweb source tree, its KSP processors and Gradle
plugins, the official templates, every GitHub release note, and the open issue milestones
— not from the marketing pages. Where an API constraint is enforced by a compiler
warning that is easy to miss, the skill says so.

## Version baseline

Verified against **Kobweb 0.25.1**, **Kobweb CLI 0.9.23**, Kotlin 2.4.10, Compose HTML
1.11.1 / Compose Runtime 1.12.0, Ktor 3.5.0 (September 2026).

Kobweb is pre-1.0 and moving; `11-capabilities-and-roadmap.md` is the section that ages
fastest. Re-check
[COMPATIBILITY.md](https://github.com/varabyte/kobweb/blob/main/COMPATIBILITY.md) and the
[issue milestones](https://github.com/varabyte/kobweb/milestones) before relying on
anything version-specific.

## Installation

### As a plugin (recommended)

This repo is also a Claude Code **plugin marketplace**, so it can be installed and kept
up to date with two commands:

```shell
/plugin marketplace add macsystems/kobweb-skill
/plugin install kobweb@kobweb-skill
```

CLI equivalents:

```bash
claude plugin marketplace add macsystems/kobweb-skill
claude plugin install kobweb@kobweb-skill
```

`/plugin marketplace update kobweb-skill` refreshes the catalog; `/plugin update kobweb`
then applies a new release. Because the plugin declares an explicit `version`, installed
copies only move when that number changes — so **bump `version` in both
`.claude-plugin/plugin.json` and the marketplace entry** whenever the content changes
materially (a new Kobweb baseline, say). `claude plugin tag` creates a matching git tag
and checks that the two manifests agree.

> While this repository is private, installation uses your existing git credentials. Run
> `gh auth setup-git` (or use an SSH remote) so background auto-updates can authenticate
> too — credential helpers are not used for those by default.

### As a plain folder

Skills are just directories; copy or symlink one if you would rather not use the plugin
system.

#### For your user account (available in every project)

```bash
mkdir -p ~/.claude/skills
cp -r /path/to/kobweb-skill/skills/kobweb ~/.claude/skills/
```

Or symlink it so `git pull` keeps it current:

```bash
ln -s /path/to/kobweb-skill/skills/kobweb ~/.claude/skills/kobweb
```

#### Per project (shared with the team)

```bash
# from the root of your project
mkdir -p .claude/skills
cp -r /path/to/kobweb-skill/skills/kobweb .claude/skills/
```

Commit `.claude/skills/` to share it with everyone on the repo.

#### Other agents

Some tools look in `.gemini/skills/` or an equivalent path. The folder is
tool-agnostic — symlink it wherever your agent expects skills to live.

## Verifying it loaded

Ask the agent something only the skill would know, e.g. *"Why would a `CssStyle` I
declared silently not apply?"* — the answer should mention that KSP skips non-public or
locally declared styles with a warning rather than an error.

## Contributing

Corrections welcome, especially where Kobweb has moved on. Two ground rules:

1. **Cite the source.** Prefer the framework's own source tree, KDoc, release notes, or a
   linked issue over documentation prose.
2. **Keep it framework-neutral.** The skill describes Kobweb, not any particular product
   built with it.

## License

The skill text is released under Apache-2.0, matching Kobweb's own license. Kobweb itself
is © Varabyte and is not affiliated with this repository.

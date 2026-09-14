---
name: kobweb
description: Expert knowledge of Kobweb, the opinionated Kotlin/JS web framework built on Compose HTML (pages, layouts, Silk styling, full-stack API routes and streams, web workers, markdown, static and full-stack export). Use when setting up, building, reviewing, debugging, or upgrading a Kobweb site; when deciding whether Kobweb fits a project versus Compose Multiplatform for Web or a JS framework; when writing `@Page`/`@Layout`/`@Api` code, `CssStyle`/Silk theming, or the `kobweb {}` Gradle DSL; and when a question turns on what Kobweb supports today versus what is still on its roadmap.
---

# Kobweb — Kotlin Web Framework Expert

Kobweb is an opinionated framework for building websites and web apps in Kotlin. It sits
on top of **Compose HTML** (JetBrains' DOM-driven Compose renderer) and takes its shape
from Next.js and Chakra UI: file-system routing, a batteries-included widget/styling layer
(**Silk**), a Ktor-based server for API routes and streams, live reloading, and a static
or full-stack export.

Kobweb is **pre-1.0 but production-used**. As of August 2026 the maintainer's own
assessment is that "the APIs are basically stable; we are just adding more and more
missing functionality (CSS properties and UI widgets)". Treat the API as stable-ish, the
feature set as still filling in, and always check the roadmap before promising a
capability — see `references/11-capabilities-and-roadmap.md`.

## Version baseline

Everything in this skill was verified against these versions. Re-check
[COMPATIBILITY.md](https://github.com/varabyte/kobweb/blob/main/COMPATIBILITY.md) before
acting on version-specific advice.

| Component | Version |
| --- | --- |
| Kobweb (library + Gradle plugins) | **0.25.1** (2026-08-19) |
| Kobweb CLI (separate repo, separate version) | **0.9.23** |
| Kotlin | 2.4.10 |
| Compose HTML (`org.jetbrains.compose.html:html-core`) | 1.11.1 |
| Compose Runtime (`androidx.compose.runtime:runtime`) | 1.12.0 |
| Ktor (server) | 3.5.0 |

Since **0.24.0** the Compose runtime comes from **androidx**, not from the JetBrains
Compose Gradle plugin — two separate version refs in the catalog. Do not collapse them.

## The one-paragraph mental model

You write `@Composable` functions in a `pages` package of a Kotlin/JS (`jsMain`) module.
A **KSP processor** scans your source at build time and generates a `main.kt` that
registers every page's route, layout, style, keyframe, and init hook into a `Router` and
a `SilkTheme`. At runtime the whole site is one JS bundle: navigation is client-side, the
DOM is driven by Compose HTML, and styling is real CSS (classes in a generated stylesheet
plus inline styles) rather than a canvas. If you enable a server, a second **JVM
(`jvmMain`)** target compiles `@Api` handlers into a jar that the Kobweb Ktor server
loads and serves at `/api/...` on the same origin. `kobweb export` then either snapshots
every page to static HTML (static layout) or bundles frontend + server (full-stack
layout).

Two consequences follow from "it is HTML/CSS, not a canvas", and they drive most design
decisions:

1. **You are writing CSS.** `Modifier` is a type-safe CSS builder; modifier order does
   *not* matter (unlike Compose Multiplatform). Knowing CSS is an asset, not a liability.
2. **The DOM is yours.** Devtools, SEO crawlers, accessibility tooling, print styles,
   `:visited`, and every JS library on npm keep working.

## When to use this skill

- Creating, structuring, or reviewing a Kobweb project (build scripts, `conf.yaml`,
  module split, page/layout organization).
- Writing or reviewing `@Page`, `@Layout`, `@App`, `@Init*`, `@Api`, `ApiStream`,
  `CssStyle`, Silk widget, or worker code.
- Choosing between Kobweb and alternatives (Compose Multiplatform for Web / Kilua /
  a TS framework), or between static and full-stack export.
- Debugging: routes not found, styles not applied, layout collapse, export failures,
  live-reload not firing, KSP warnings that silently drop a style.
- Upgrading Kobweb / Kotlin / Compose versions.
- Answering "can Kobweb do X?" — answer from
  `references/11-capabilities-and-roadmap.md`, not from intuition.

## Non-negotiable rules

1. **Never promise SSR or hydration.** Kobweb exports *pre-rendered static snapshots*;
   on load, Compose tears the root node down and rebuilds it. Real hydration (#113) and
   server-side rendering (#114) are both open, milestone 1.1, untouched since 2022.
2. **Never suggest a Kotlin/Wasm target.** Kobweb is Kotlin/JS only (`js { browser() }`).
   Wasm support is an open *investigation* (#239), explicitly not planned for 1.0.
3. **Declare `CssStyle`, `CssStyleVariant`, and `Keyframes` as public top-level `val`s**
   (or inside an `object`/companion). KSP silently *skips registration* of local or
   non-public ones — it logs a warning, not an error, so the style just never applies.
4. **A `ComponentKind` interface and its `CssStyle<K>` must live in the same file**, and
   each kind may back exactly one style. Both are hard KSP errors.
5. **`Modifier` order does not matter** in Kobweb. Do not port Compose Multiplatform
   modifier-ordering reasoning into a Kobweb review.
6. **Use the `kobweb` CLI or its Gradle equivalents** — never `./gradlew jsBrowserRun`
   or a hand-rolled static server. Routing, KSP generation, and live reload are wired
   through the Kobweb Gradle tasks.
7. **Do not set `kobweb.kspProcessorDependency`** in a normal project. It has a working
   convention; the value seen in the framework's own `playground` exists only because of
   composite builds.
8. **Prefer a static export** unless the site genuinely needs private backend calls,
   request interception, or client-to-client connections. Static is cheaper to host,
   trivially CDN-able, and keeps every deployment option open.
9. **Check for a type-safe `Modifier`/CSS API before reaching for a raw string.** Kobweb
   has an unusually deep CSS surface; if it really is missing, use `styleModifier` /
   `attrsModifier` / `property()` as the documented escape hatch and consider filing an
   issue.
10. **Every new page is one more chunk of the single bundle.** There is no code splitting
    (see #787); registering a `CssStyle` pulls its whole file into the main chunk. Budget
    bundle size accordingly.

## Decision guide

| Question | Answer |
| --- | --- |
| Static or full-stack export? | Static, unless you need `@Api`/`ApiStream`, secret-holding backend calls, or request interception. |
| Where does shared frontend/backend code go? | `commonMain` of the app module (needs `includeServer = true`), or a `com.varabyte.kobweb.library` module. |
| One big site module or several? | Split reusable UI into `kobweb.library` modules; pages in libraries *are* discovered and routed. |
| Silk or raw Compose HTML? | Silk for layout/theming/widgets; drop to Compose HTML (`Div`, `Text`, `TagElement`) whenever Silk is in the way. They interoperate freely. |
| Need a React/npm library? | Fine — it is a normal Kotlin/JS project. Use `external` declarations or `js()`; see `references/10-interop-and-escape-hatches.md`. |
| Need to share UI with Android/iOS? | Kobweb cannot. Compose HTML's API differs from Compose Multiplatform's. Share *logic* via `commonMain`, not UI. |
| Need auth? | Roll it yourself (API route + cookie/JWT) or use a server plugin. There is no built-in auth (#254, milestone 1.1). |
| Need tests? | No official story yet (#92). Test pure logic in a JVM/common source set; the browser layer stays manual or Playwright-driven. |

## Reference material

Load the file that matches the task; each is self-contained.

| File | Covers |
| --- | --- |
| `references/01-setup-and-build.md` | CLI install & commands, `kobweb create`, project layout, `build.gradle.kts`, version catalog, `settings.gradle.kts`, `conf.yaml`, the whole `kobweb {}` Gradle DSL, snapshots. |
| `references/02-routing.md` | `@Page`, route derivation & overrides, `@PackageMapping`, dynamic / optional / catch-all routes, `PageContext`, `Router`, interceptors, redirects, base path, links. |
| `references/03-app-layouts-init.md` | `@App`, `@Layout` (three forms), `@NoLayout`, `@InitKobweb` / `@InitRoute` / `@InitSilk`, the `ctx.data` store, state that survives navigation. |
| `references/04-styling.md` | `Modifier`, inline vs stylesheet, `CssStyle` (all flavors), `ComponentKind` + variants, breakpoints, color modes, `StyleVariable`, `Keyframes`, CSS layers, naming annotations, escape hatches, element refs. |
| `references/05-silk-widgets.md` | Every Silk widget and built-in icon, the four icon packs, palettes and theme overrides. |
| `references/06-fullstack.md` | `@Api`, `ApiContext`, request/response bodies & multipart, interceptors, `@InitApi`, `ApiStream`, the client `window.api` / `window.http` APIs, server plugins, CORS, logging. |
| `references/07-workers.md` | Kobweb worker modules, `WorkerFactory`/`WorkerStrategy`, naming constraints, `rememberWorker`, attachments. |
| `references/08-markdown.md` | The markdown plugin, front matter, layouts/roots, imports, `process`/generate hooks, callouts, inline Kotlin calls. |
| `references/09-export-and-deploy.md` | Static vs full-stack layout, export commands & browser requirement, `isExporting`, dynamic-route export, clean URLs, hosting, CI. |
| `references/10-interop-and-escape-hatches.md` | Compose HTML interop, `compose-html-ext` / `browser-ext`, raw HTML & SVG, npm/JS libraries, using your own backend server. |
| `references/11-capabilities-and-roadmap.md` | **The support matrix.** What works today, what is explicitly missing, known limitations, the 1.0 / 1.1 roadmap with issue numbers, in-flight PRs. |
| `references/12-versions-and-upgrading.md` | Full compatibility table, artifact coordinates, upgrade procedure, migrations that have bitten users. |

## Authoritative sources

- Source: <https://github.com/varabyte/kobweb> · CLI: <https://github.com/varabyte/kobweb-cli>
- Templates: <https://github.com/varabyte/kobweb-templates>
- Guide: <https://kobweb.varabyte.com/docs> · API reference: <https://varabyte.github.io/kobweb/>
- Issues / roadmap: <https://github.com/varabyte/kobweb/issues> (milestones `1.0`, `1.1`)

When a detail matters and this skill is silent or stale, read the framework source — it is
heavily KDoc'd, and the annotation files (`Page.kt`, `Layout.kt`, `Api.kt`) are the best
specification of the routing rules that exists.

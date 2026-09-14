# Capability matrix and roadmap

State as of **Kobweb 0.25.1 / CLI 0.9.23 (September 2026)**. Issue numbers refer to
<https://github.com/varabyte/kobweb/issues>. Verify before quoting — this is the section
most likely to age.

## Maturity

Kobweb is **pre-1.0 but in production use** by hobbyists and small businesses, including
the maintainers' own sites. In August 2026 the project removed its long-standing
"Can We Kobweb Yet" caveat section, with the assessment: *"The APIs are basically stable;
we are just adding more and more missing functionality (CSS properties and UI widgets)."*

Practical reading: breaking changes are rare, deprecations get `ReplaceWith` hints, and
the residual risk is **feature gaps**, not churn. There is no published 1.0 date; the
maintainer has publicly and repeatedly missed his own estimates and now declines to give
one. ~2.2k GitHub stars, one primary maintainer plus regular community contributors,
Apache-2.0.

## Supported today

### Foundation
- File-system routing with static, dynamic, optional, and catch-all segments; static and
  dynamic siblings; compile-time route validation.
- `@Layout` layouts: per-page, per-package defaults, nesting, `@NoLayout` opt-out, with
  layout state surviving navigation.
- `@App` root, `@InitKobweb` / `@InitRoute` / `@InitSilk` hooks, a typed per-route data
  store.
- Route interceptors, client-side redirects, server-side 301 redirects with regex
  captures, a customizable error page, base-path support.
- Build-time app globals; typed browser storage; persisted color mode.
- Live reloading (`kobweb run` / `kobwebStart -t`).

### Presentation
- `Modifier` covering a very large share of CSS, plus typed gradients, filters,
  `color-mix`, `calc`.
- `CssStyle` with pseudo-classes, pseudo-elements, media queries, arbitrary `cssRule`,
  combinators, ARIA attribute selectors.
- Component styles + typed variants, restricted styles, style extension.
- Five breakpoints, combinable with pseudo-selectors.
- Light/dark color modes with palettes and widget color groups.
- CSS custom properties (`StyleVariable`) with arithmetic and grouping.
- Keyframes/animations, including color-mode-aware ones.
- CSS layers with a defined Silk ordering.
- Rich SVG support and built-in SVG icons; Font Awesome, Material Symbols, Lucide, and
  (legacy) Material Design icon packs.

### Server
- `@Api` routes with dynamic segments, `@InitApi` services, a single `@ApiInterceptor`
  for middleware.
- Request/response bodies including **multipart uploads**, multi-value response headers,
  kotlinx-serialization helpers, redirects.
- API streams (multiplexed over one websocket) with connect/message/disconnect hooks and
  broadcast.
- Configurable logging (with `println` interception), CORS, streaming, build-script
  system properties, remote JVM debugging.
- Kobweb server plugins for raw Ktor access.

### Build & delivery
- Static and full-stack export; parallel export; configurable/reusable browser;
  extra-route export for dynamic pages; export filters and traces.
- Markdown pages with front matter, layouts, embedded Kotlin calls, callouts,
  per-node handlers, and build-time processing/generation.
- Web workers as first-class modules with typed messages and attachments.
- Multi-module projects: Kobweb library modules contribute pages, styles, API handlers,
  and workers.
- Gradle "Isolated Projects" compatibility (0.25.1).

## Not supported

| Capability | Status | Issue |
| --- | --- | --- |
| **Server-side rendering** | Open, milestone 1.1, untouched since 2022. Blocked on hydration. | #114 |
| **Hydration** | Open, milestone 1.1. Today the exported DOM is discarded and rebuilt. | #113 |
| **Kotlin/Wasm target** | Investigation only; explicitly *not* planned for 1.0. Maintainer is still looking for a use case that Compose Web or Kilua does not already serve. | #239 |
| **Sharing UI with Compose Multiplatform** | Not a goal. Compose HTML's API differs by design. | — |
| **Compose Resources** | Not integrated; no code in the repo references it. | #486 |
| **Code splitting / lazy chunks** | Everything lands in one bundle; registering a `CssStyle` pulls its whole file into the main chunk. Post-1.0 goal; the maintainer leans toward user-configurable *page groups* rather than per-style splitting. | #787, #373 |
| **Official testing story** | No published test utilities. An internal `compose-test-utils` module exists but is unpublished. | #92 |
| **Built-in authentication** | Design undecided (server plugin vs. `kobwebx` library). | #254 |
| **Running `@Api` inside your own server** | Open epic. Static export + your own backend is the supported path. | #22 |
| **Theme / design-token API** | Requested, milestone 1.0; hand-rolled palette classes are the current idiom. | #297 |
| **Server-rendered `<meta>` tags** | Open, milestone 1.1 — affects per-route social previews. | #684 |
| **Sitemap generation** | Community PR open. | #701 / PR #738 |
| **PWA support** | Open, no milestone. | #495 |
| **CSS `@container` queries** | Open, milestone 1.0. | #619 |
| **More CSS at-rules in `CssStyle`** | Open, milestone 1.0. | #584 |
| **File-input widget** | Open, milestone 1.0. | #749 |
| **`srcset`/`sizes` on `Image`** | Open, milestone 1.1. | #721 |
| **Element visibility tracking** | Open, milestone 1.1. | #223 |
| **Error pages beyond 404** | Open, milestone 1.0. | #253 |
| **`public/` asset cache headers** | Open, milestone 1.0. | #143 |
| **Docker image** | Open, milestone 1.1. | #99 |
| **Explicit API mode across Kobweb modules** | Open, milestone 1.0. | #227 |

## Known defects worth planning around

- **CSS layers on browsers without `@layer` support** produce bad output (#601, Safari,
  milestone 1.0).
- **`rememberBreakpoint` and exporting** interact badly (#318) — prefer CSS-driven
  responsiveness for anything that must be right in a static snapshot.
- **Cmd+Click on macOS** opens internal links in the same tab rather than a new one
  (#795).
- **`@file:PackageMapping("")`** throws an NPE (#164).
- **Widget accessibility** (aria-labels, high contrast) is incomplete (#673, #295).
- **Palette naming is inconsistent** and slated for a cleanup (#291) — a possible source
  of a future breaking rename.

## Roadmap

### Milestone 1.0 (42 issues open, 116 closed)

Mostly polish and completeness rather than architecture: widget accessibility and
a11y review, `Size`/`ColorScheme` parameters on every widget, a file-input widget,
container queries and more at-rules, a theme/token API, richer error-status handling,
asset caching, explicit API mode, the layer/Safari fix, the modifier-completeness epic
(#200), the string-typed-modifier cleanup (#173), and a large documentation push
(#134 Guide MVP, #133 Reference MVP).

### Milestone 1.1 (13 issues open)

Where the architectural items live: **hydration (#113)** and **SSR (#114)**,
**auth (#254)**, **running under someone else's server (#22)**, server-rendered meta tags
(#684), `srcset` (#721), visibility tracking (#223), a Docker image (#99).

### In flight (open PRs, September 2026)

- **#738** — built-in sitemap generation (implements #701).
- **#741** — `<body>` scripts configurable from the `kobweb {}` block (implements #299).
- **#732** — enhanced markdown rendering support.

### Recent direction (what the last releases actually spent effort on)

- **0.25.1** — HTTP `QUERY` method, parallel exports, Gradle Isolated Projects
  compatibility, `println` log interception, workers honour base path.
- **0.25.0** — Kotlin 2.4.0 / Compose bump only.
- **0.24.1** — Material Symbols and Lucide icon packs; the network-API redesign
  (`bodyOf`/`RequestBody`, web `Response` return values); worker-friendly `self.fetch`;
  backend system properties; reusable export browser.
- **0.24.0** — multipart requests; Compose runtime moved to androidx; markdown HTML
  handling overhaul; compile-time dynamic-route validation; unused-return-value checker.
- **0.23.x** — worker fixes and structured-clone attachments; URL-decoded params.
- **0.22.0** — layouts.

The pattern: steady CSS/widget completion, periodic toolchain bumps, and occasional
focused subsystem work (workers, networking, markdown). No sign yet of hydration or SSR
being started.

## How to answer "can Kobweb do X?"

1. Check the **Not supported** table above.
2. If it is a CSS property or a widget detail, assume it probably exists — search the
   source before saying no; if it truly is missing, the `styleModifier`/`attrsModifier`
   escape hatch covers it.
3. If it is architectural (rendering model, bundling, hosting model), the answer is
   usually governed by "Kobweb owns the server and ships one bundle of pre-rendered
   client-side pages" — reason from there.
4. Never invent a milestone date.

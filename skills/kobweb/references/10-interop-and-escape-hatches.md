# Interop and escape hatches

Kobweb is a normal Kotlin/JS browser project underneath. Nothing it does not support is
therefore blocked — it just means dropping one level down. Knowing the ladder is what
keeps a Kobweb project from getting stuck.

The ladder, from most to least framework:

1. Silk widget → 2. `kobweb-compose` layout (`Box`/`Row`/`Column`) → 3. raw Compose HTML
element (`Div`, `Span`, `Text`, `TagElement`) → 4. `GenericTag` / `RawHtml` →
5. `Modifier.styleModifier`/`attrsModifier`/`property()` → 6. `ref { element -> … }` and
plain DOM calls → 7. `external`/`js()` into a JavaScript library.

## Compose HTML interop

Silk and Compose HTML mix freely in the same composition:

```kotlin
Column(Modifier.gap(1.cssRem)) {
    H1 { Text("Title") }                       // Compose HTML
    SpanText("Subtitle")                       // Silk
    Div(attrs = MyStyle.toAttrs()) { … }       // a Kobweb style on a raw element
}
```

`Modifier.toAttrs()` is the bridge in one direction; `Modifier.attrsModifier { … }` in
the other.

## `compose-html-ext` and `browser-ext`

Two libraries Kobweb extracted so they are usable **without Kobweb** — even without
Compose HTML, in `browser-ext`'s case.

`com.varabyte.kobweb:compose-html-ext` provides:

- type-safe wrappers for a very large number of CSS properties Compose HTML lacks;
- typed CSS functions: gradients (including `repeating-*` and color-interpolation
  modes), filters, `color-mix`, `calc`;
- rich SVG support;
- `GenericTag(name, attrsStr, content)` — an easy wrapper over `TagElement`, with
  namespacing (needed for SVG);
- `RawHtml(htmlString)` — parses and mounts a raw HTML string. Marked **unsafe**, opt-in
  required; never pass user input to it;
- `StyleVariable`, fixing Compose HTML's `CSSStyleVariable` accepting invalid values;
- transition events, which Compose HTML omits.

`com.varabyte.kobweb:browser-ext` (pulled in transitively) provides:

- `window.fetch` / `window.http` / `HttpFetcher` — suspend, typed HTTP with
  `RequestBody`/`bodyOf`;
- `IntersectionObserver` and `ResizeObserver` bindings;
- file I/O: `Document.saveToDisk`, `saveTextToDisk`, `loadFromDisk`,
  `loadTextFromDisk`, `loadMultipleFromDisk`, `File.readBytes()` (suspend);
- typed web storage: `IntStorageKey`/`EnumStorageKey`/… plus
  `Storage.getItem(key)` / `setItem(key, value)`;
- Kotlin-idiomatic `setTimeout` / `setInterval` / `invokeLater` taking a `Duration` and a
  trailing lambda, returning a `CancellableActionHandle`;
- DOM walking (`ElementExtensions`), `Selection`, WebGL2 bindings, coroutine dispatchers
  for window and worker scopes, URI encode/decode.

Using them outside Kobweb:

```kotlin
kotlin {
    js().browser()
    sourceSets.jsMain.dependencies {
        implementation(compose.html.core)
        implementation(compose.runtime)
        implementation("com.varabyte.kobweb:compose-html-ext:0.25.1")
    }
}
```

## Reaching the DOM

```kotlin
Box(ref = disposableRef { element ->
    val observer = ResizeObserver { … }
    observer.observe(element)
    onDispose { observer.disconnect() }
})
```

`ElementTarget` (in `browser-ext`) helps find sibling/ancestor elements for things like
tooltips.

## Using a JavaScript library

Standard Kotlin/JS. Either declare externals:

```kotlin
@JsModule("chart.js/auto")
external class Chart(ctx: Any, config: dynamic)
```

with the npm dependency registered in the build script
(`implementation(npm("chart.js", "4.4.0"))`), or load a script via
`kobweb { app { index { head.add { script { src = "https://…" } } } } }` and bind to the
global with `js("window.Foo")`.

Note `index { interceptUrls { enableSelfHosting() } }` will download those `<head>`
resources at build time and serve them from your own origin — useful for CSP, privacy
rules, or offline builds.

Webpack is the bundler today; replacing or removing it is an open investigation (#330).

## Missing CSS or modifiers

```kotlin
Modifier.styleModifier { property("anchor-name", "--my-anchor") }
Modifier.attrsModifier { attr("data-state", "open") }
```

These always work. If a property is stable and widely supported, also consider filing an
issue — epic #200 tracks filling the remaining gaps, and CSS APIs are the most
frequently contributed area of the project.

## Using your own backend server

If you already run Ktor/Spring/Express, export the site **statically** and serve
`.kobweb/site` from it:

```kotlin
// Ktor
routing {
    staticFiles("/", File(".kobweb/site")) {
        enableAutoHeadResponse()
        extensions("html")      // makes /about resolve to about.html — REQUIRED
        default("index.html")
    }
}
```

Then talk to your own endpoints with `window.http.get("/my/endpoint")`.

What you give up: `@Api` routes, API streams, and live reloading. Making Kobweb work when
it does not own the server is an open epic (#22); the maintainers acknowledge the
limitation and have deprioritized it until after 1.0.

## Compose Multiplatform

You **cannot** share UI code between Kobweb and Compose Multiplatform (Android/iOS/
desktop). Compose HTML's API is deliberately different: it manipulates an HTML/CSS DOM
rather than owning a render pipeline, so `Modifier`, layout, and drawing all diverge.
Compose *Resources* are not integrated either (#486, open since 2024).

What you can share: everything that is not UI — models, validation, networking,
state machines, `@Serializable` DTOs — via a `commonMain` source set.

If a single UI codebase across web and mobile is a hard requirement, Compose
Multiplatform for Web (canvas) is the right tool and Kobweb is not. The trade-offs
Kobweb's author lists for canvas rendering: no crawlable DOM, slower first paint, a large
canvas buffer on wide displays, opaque devtools, no `:visited` styling, no print styles,
degraded accessibility tooling, and a 4–6× larger bundle.

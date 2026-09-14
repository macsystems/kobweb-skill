# Application root, layouts, and init hooks

## `@App` — the root skeleton

Exactly **zero or one** `@App` function per application module (never in a library — that
is a KSP error). It takes a `content: @Composable () -> Unit`.

```kotlin
@App
@Composable
fun AppEntry(content: @Composable () -> Unit) {
    SilkApp {
        val colorMode = ColorMode.current
        LaunchedEffect(colorMode) { colorMode.saveToLocalStorage(COLOR_MODE_KEY) }

        Surface(SmoothColorStyle.toModifier()) {
            Box(Modifier.minHeight(100.vh)) {
                content()
            }
        }
    }
}
```

If you omit `@App`, Kobweb falls back to `KobwebApp` (or `SilkApp` when Silk is on the
classpath). `KobwebApp` installs a small `reset` layer: zero margin/padding on
`html, body` and `box-sizing: border-box` on `*`.

> **The collapsing-height trap (0.22.0+).** `Surface` used to be a `Box` (a CSS grid).
> It is now a plain `<div>`, and plain divs ignore `min-height` when no ancestor has a
> height. If a footer stops sticking to the bottom after upgrading, move `minHeight(100.vh)`
> off the `Surface` and onto a `Box` child, as shown above.

`@App` runs above the router, so `rememberPageContext()` is **not** available in it.

## `@Layout` — shared page scaffolding

Since 0.22.0. A layout is a `@Composable` that takes an optional leading `PageContext`
and a trailing `content: @Composable () -> Unit`.

```kotlin
@Composable
@Layout
fun PageLayout(ctx: PageContext, content: @Composable () -> Unit) {
    Column(Modifier.fillMaxSize(), horizontalAlignment = Alignment.CenterHorizontally) {
        NavHeader()
        content()
    }
}
```

Three ways to apply one:

1. **Per page** — annotate the page with the layout's FQN (a leading `.` means "relative
   to the module's root package"):

   ```kotlin
   @Page
   @Layout(".components.layouts.BlogLayout")
   @Composable
   fun BlogPage() { /* ... */ }
   ```

2. **Per package (default layout)** — a file-level annotation on any file under the pages
   package applies to every page in that package and below. The **most specific** wins.

   ```kotlin
   // pages/Layout.kt
   @file:Layout(".components.layouts.PageLayout")
   package org.example.myproject.pages
   import com.varabyte.kobweb.core.layout.Layout
   ```

3. **Nested layouts** — annotate the layout itself with its parent:

   ```kotlin
   @Layout(".components.layouts.PageLayout")
   @Composable
   fun NestedLayout(content: @Composable () -> Unit) { /* ... */ }
   ```

Opt a single page out with `@NoLayout`. Combining `@Layout` and `@NoLayout` on the same
page is an error, as is a layout cycle.

The resulting composition is `App { Layout { Page() } }`.

**The real benefit is state.** Because the layout composable is not re-entered on every
navigation, state held in a layout (scroll position, an open drawer, a media player)
survives page changes — no local-storage tricks needed. This is the main reason to
migrate from the legacy "page calls `PageLayout { ... }` itself" pattern, which still
works and is not going away.

## Init hooks

Four annotations, all taking a single context parameter, all discovered by KSP.

| Annotation | Runs | Context |
| --- | --- | --- |
| `@InitKobweb` | once, when the page first loads, before nodes are laid out | `InitKobwebContext(config, router)` |
| `@InitSilk` | once at startup, to configure Silk | `InitSilkContext(config, stylesheet, theme)` |
| `@InitRoute` | every time the associated page/layout is about to render | `InitRouteContext(route, data)` |
| `@InitApi` | once at server start (JVM side) | `InitApiContext(data, env, logger, ...)` |

Multiple `@InitKobweb` / `@InitSilk` methods are allowed and may run in any order — never
rely on ordering between them. `@InitRoute` is limited to **one per file**.

### `@InitKobweb`

```kotlin
@InitKobweb
fun initKobweb(ctx: InitKobwebContext) {
    ctx.router.addRouteInterceptor {
        if (path == "/admin") path = "/admin/dashboard"
    }
    ctx.router.setErrorPage { H1 { Text("Page not found!") } }
}
```

### `@InitRoute` and the data store

`@InitRoute` is how a page hands typed data to its layout. Pair each layout with a data
class:

```kotlin
class PageLayoutData(val title: String)

@InitRoute
fun initPageLayout(ctx: InitRouteContext) {
    ctx.data.addIfAbsent { PageLayoutData("(Missing title)") }   // the layout's default
}

@Composable
@Layout
fun PageLayout(ctx: PageContext, content: @Composable () -> Unit) {
    val data = ctx.data.getValue<PageLayoutData>()
    LaunchedEffect(data.title) { document.title = data.title }
    /* ... */
}
```

```kotlin
// in a page file
@InitRoute
fun initStylesPage(ctx: InitRouteContext) {
    ctx.data.add(PageLayoutData("Styles"))
}
```

The `Data` API is a `KClass`-keyed heterogeneous map:

| Call | Meaning |
| --- | --- |
| `data.add(value)` | store, keyed by the reified type |
| `data.addIfAbsent { value }` | store only if that type is not present (layout defaults) |
| `data.get<T>()` | `T?` |
| `data.getValue<T>()` | `T`, throws if missing |

`@InitRoute` methods run **page first, then outward through the layouts** (nearest
ancestor first), which is why `addIfAbsent` in the layout and `add` in the page compose
correctly.

Two behaviours worth knowing:

- The data store is **cleared on every route change** before the init methods run — it is
  per-route scratch space, not app state.
- Init methods only re-run when the **path** changes. Navigating from `/search?q=a` to
  `/search?q=b` does *not* re-run `@InitRoute`; react to query changes inside the page
  composition instead.

`ctx.data` on `PageContext` exposes the read-only view of the same store.

### `@InitSilk`

See `references/04-styling.md`. Typical uses: initial color mode, global element styles,
palette overrides, breakpoint overrides, replacing Silk widget styles.

```kotlin
@InitSilk
fun initColorMode(ctx: InitSilkContext) {
    ctx.config.initialColorMode =
        ColorMode.loadFromLocalStorage(COLOR_MODE_KEY) ?: ColorMode.systemPreference
}
```

`ColorMode.systemPreference` reads `prefers-color-scheme`. Persisting the mode is the
documented way to avoid a flash of the wrong theme on an exported site.

### `@InitApi`

Server-side service initialization; see `references/06-fullstack.md`.

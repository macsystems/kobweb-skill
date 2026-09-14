# Routing

## `@Page`

A `@Composable` function annotated `@Page` inside the pages package becomes a route. The
**file name** decides the slug — the function name is irrelevant.

```kotlin
// site/src/jsMain/kotlin/org/example/myproject/pages/account/Profile.kt
@Page
@Composable
fun ProfilePage() { /* -> /account/profile */ }
```

Rules:

- The file name is converted to kebab-case: `WelcomeIntro.kt` → `/welcome-intro`.
- `Index.kt` is the directory default: `pages/blog/Index.kt` → `/blog`.
- Package segments map to path segments, with underscores removed and camelCase
  hyphenated.

A page function may take **no parameters** or **a single `PageContext`**. Anything else
is a KSP error.

### `routeOverride`

Prefer *not* to set it — it makes routes harder to find. The rules, for
`package pages.a.b.c` in `Slug.kt`:

| Annotation | Resulting route |
| --- | --- |
| `@Page` | `/a/b/c/slug` |
| `@Page("other")` | `/a/b/c/other` |
| `@Page("index")` | `/a/b/c/` |
| `@Page("d/e/f/")` | `/a/b/c/d/e/f/slug` |
| `@Page("d/e/f/other")` | `/a/b/c/d/e/f/other` |
| `@Page("/d/e/f/")` | `/d/e/f/slug` |
| `@Page("/")` | `/slug` |

Leading slash = absolute from the site root. Trailing slash = change the path but keep
the file-derived slug.

## `@PackageMapping`

Rename an intermediate path segment without renaming the package:

```kotlin
// pages/blog/_2024/Post.kt
@file:PackageMapping("2024")

package org.example.myproject.pages.blog._2024
```

`@file:PackageMapping("")` currently triggers an NPE (#164) — avoid it.

## Dynamic routes

Curly braces mark a dynamic segment. An empty `{}` reuses the file name.

| Form | Meaning |
| --- | --- |
| `@Page("{}")` in `Slug.kt` | `/a/b/c/{slug}` |
| `@Page("{user}")` | named capture |
| `@Page("/users/{user}/posts/{post}")` | multiple captures |
| `@Page("{slug?}")` | optional segment — **only valid in the final position** |
| `@Page("{...rest}")` | catch-all: captures `d/e/f` from `/a/b/c/d/e/f` |
| `@Page("{...rest?}")` | optional catch-all (matches the empty remainder too) |

Since 0.24.0 the KSP processor validates these at compile time: a non-final optional
segment, a redundant optional, and conflicting dynamic siblings are all errors.

Static and dynamic siblings at the same level are supported (properly, since 0.20.3) and
the **static route wins**.

## `PageContext`

Obtain it as a page parameter or via `rememberPageContext()` anywhere beneath a `@Page`.
It throws outside a page composition.

```kotlin
@Page
@Composable
fun UserPage() {
    val ctx = rememberPageContext()
    val user = ctx.route.params["user"] ?: "unknown"
    Button(onClick = { ctx.router.navigateTo("/") }) { Text("Home") }
}
```

`ctx.route` is a `RouteInfo`:

| Member | For `https://example.com/a/b/c/slug?x=1&y=2#id` |
| --- | --- |
| `path` | `/a/b/c/slug` |
| `slug` | `slug` |
| `origin` | `https://example.com` |
| `queryParams` | `{x=1, y=2}` |
| `dynamicParams` | captures from the route pattern (URL-decoded) |
| `params` | `queryParams + dynamicParams` — **dynamic wins on collision** |
| `fragment` | `id` |
| `pathQueryAndFragment` | `/a/b/c/slug?x=1&y=2#id` |

All captured and query values are URL-decoded (since 0.23.0): `?arg=hello%20world`
yields `hello world`.

`ctx.router` is the `Router`; `ctx.data` is the per-route data store
(see `references/03-app-layouts-init.md`).

## Navigating

```kotlin
ctx.router.navigateTo(
    pathQueryAndFragment = "/about",
    updateHistoryMode = UpdateHistoryMode.PUSH,      // or REPLACE
    openInternalLinksStrategy = OpenLinkStrategy.IN_PLACE,
    openExternalLinksStrategy = OpenLinkStrategy.IN_NEW_TAB,
)
```

- `tryRoutingTo(...)` returns `false` instead of falling back to the error page.
- Passing a full URL on your own domain (`navigateTo("https://mysite.com/about")`)
  behaves as a hybrid: it opens in place but performs a real server fetch, and the base
  path is *not* applied.
- Navigation snaps to the top of the new page instead of smooth-scrolling (0.20.3+).

### Links

`Link` (Silk, styled) and `Anchor` (kobweb-core, unstyled) both route internally and
respect the base path:

```kotlin
Link("/about", "About us")
```

## Route interceptors

Registered on the router; they rewrite a route before it resolves.

```kotlin
@InitKobweb
fun initRouter(ctx: InitKobwebContext) {
    ctx.router.addRouteInterceptor {
        if (path == "/legacy") path = "/modern"
    }
}
```

Inside the interceptor scope you can read/write `path`, `queryParams`, `fragment`, and
read `pathQueryAndFragment`.

## Redirects

Two mechanisms:

1. **Client-side**, registered with the router
   (`router.registerRedirect(from, to)`), useful for a static export.
2. **Server-side**, in `.kobweb/conf.yaml` under `server.redirects`, which issues a real
   301 with regex capture substitution.

## Base path

Hosting under a subdirectory (GitHub Pages project sites) is configured once:

```yaml
site:
  basePath: "/my-repo"
```

Kobweb then prefixes generated URLs and strips the prefix when resolving routes.
`BasePath.prepend(...)` / `BasePath.remove(...)` exist for manual cases — for example a
raw `<img src>` pointing into `resources/public`. Workers respect the base path as of
0.25.1.

## The error (404) page

Replace the built-in error page from an `@InitKobweb` hook:

```kotlin
@InitKobweb
fun initErrorPage(ctx: InitKobwebContext) {
    ctx.router.setErrorPage {
        H1 { Text("Page not found!") }
    }
}
```

`setErrorPage(layoutId: String? = NO_LAYOUT_FQN, initRouteMethod: InitRouteMethod? = null,
pageMethod: PageMethod)` — by default the error page opts out of layouts; pass a layout
FQN to opt back in. Today only the 404 case is customizable; richer status-code handling
is open (#253).

## Debugging checklist

| Symptom | Likely cause |
| --- | --- |
| Route 404s in dev but the file exists | Page is not under `kobwebBlock.pagesPackage`, or the function is not `@Composable`. |
| Route works in dev, 404 after static export | Host does not rewrite extensionless URLs to `.html`; enable `cleanUrls` or configure the host. |
| Params empty | Reading `queryParams` where the value is a dynamic capture (or vice versa) — use `params`. |
| `PageContext is only valid within a @Page composable` | `rememberPageContext()` called from the `@App` root or a non-page composition. |
| Dynamic route missing from the export | Dynamic routes are skipped by design; add `addExtraRoute(...)`. |

# Exporting and deploying

## The two site layouts

| | **Static layout** | **Full-stack layout** |
| --- | --- | --- |
| Output | plain HTML/JS/assets in `.kobweb/site` | site + server jar + start scripts |
| Hosting | any static host / CDN / GitHub Pages | a machine that runs a JVM |
| `@Api` routes & streams | ✗ | ✓ |
| Secrets on the server | ✗ | ✓ |
| Cost & ops | minimal | a running process |

**Default to static.** Choose full-stack only when the site must talk to private backend
services, intercept sensitive API requests, or connect multiple clients to each other.

## Exporting

```bash
kobweb export --layout static
kobweb export --layout fullstack     # the default if --layout is omitted

# run the exported artifact locally to verify
kobweb run --env prod --layout static
kobweb run --env prod --layout fullstack
```

Since 0.23.0 Kobweb warns when the layout looks wrong — a `static` export of a project
that has backend code, or a `fullstack` export of one that has none. Suppress with
`export { suppressLayoutWarning.set(true) }` if the warning is a false positive.

### Export requires a browser

Export is a *snapshot* process: Kobweb runs your site in a headless browser (Playwright)
and saves the resulting DOM as HTML per route. That is what makes the pages
crawlable/indexable.

Consequences:

- The first export downloads a browser. In CI, or anywhere with a browser already
  present, point at it instead:

  ```bash
  export KOBWEB_EXPORT_BROWSER_PATH=/usr/bin/chromium
  export KOBWEB_EXPORT_BROWSER_TYPE=Chromium     # Chromium | Firefox | WebKit | Edge
  ```

  or in the build script: `kobweb { app { export { browser.set(Browser.Firefox); browserPath.set("…") } } }`.
- Export is parallelized since 0.25.1 (half the cores by default; tune with
  `export { numThreads.set(n) }` or `KOBWEB_EXPORT_NUM_THREADS=max|high|half|<n>`).
- A hanging export can be diagnosed with `export { enableTraces(...) }`, which writes
  Playwright traces.
- An export that produces **no pages** now fails the build with a Gradle exception rather
  than logging an error (0.25.1).

### `AppGlobals.isExporting`

Because your site actually *runs* during export, guard side effects that must not happen
then — analytics pings, auth checks, animations that never settle:

```kotlin
if (!AppGlobals.isExporting) {
    LaunchedEffect(Unit) { loggedInUser = checkForLoggedInUser() }
}
```

It is implemented as a `_kobwebIsExporting` query parameter on the URL the export browser
visits.

### Dynamic routes are skipped

A dynamic route has no single snapshot, so it is not exported. Export representative
instances explicitly:

```kotlin
kobweb {
    app {
        export {
            addExtraRoute("/users/default/posts/0", exportPath = "users/index.html")
        }
    }
}
```

Filter which routes get exported at all with `export { filter.set { … } }`.

### Clean URLs

```kotlin
kobweb { app { cleanUrls.set(true) } }
```

emits `about/index.html` instead of `about.html`, which most static hosts serve at
`/about` without extra configuration. Otherwise configure the host to try a `.html`
extension (Apache/Caddy/Nginx rewrites, Ktor's `extensions("html")`).

Getting this wrong is the single most common "works in dev, 404s in production" report.

## What an exported page actually is

A pre-rendered HTML snapshot plus the JS bundle. On load, Compose **discards the
snapshot's DOM under the root node and rebuilds it**. That means:

- ✅ crawlers, previews, and first paint see real content;
- ✅ SEO works;
- ❌ it is *not* hydration — interactive state is not adopted from the markup, and
- ❌ it is *not* SSR — every visitor gets the same snapshot; there is no per-request
  rendering.

Real hydration (#113) and SSR (#114) are both open on milestone 1.1. Server-side
rendering of `<meta>` tags specifically is also open (#684) — social-preview cards for a
per-route title/description need a workaround today.

## Deploying

**Static:** upload the contents of `.kobweb/site` anywhere — GitHub Pages, Netlify,
Cloudflare Pages, S3, nginx. Remember `site.basePath` if hosting under a subdirectory.

**Full-stack:** the export generates `.kobweb/server/start.sh` / `start.bat`. Package the
whole `.kobweb` folder plus the jars into a container, or run it under a supervisor. An
official Docker image is requested but not published (#99).

## CI

A GitHub Actions export job needs: JDK, the `kobweb` binary (or the Gradle tasks
directly), and a browser. Use `--notty` so output is linear, and pin
`KOBWEB_EXPORT_BROWSER_PATH` to the runner's preinstalled Chrome to skip the download.

```yaml
- run: kobweb export --layout static --notty
  env:
    KOBWEB_EXPORT_BROWSER_PATH: /usr/bin/google-chrome
    KOBWEB_EXPORT_NUM_THREADS: high
```

The official guide for this is at
<https://kobweb.varabyte.com/docs/guides/git-hub-workflow-export>.

## Bundle size expectations

A Compose HTML site's footprint is typically **200–400 KB** for a small site, versus
2–3 MB for a canvas-rendered Compose Multiplatform web app — roughly a 4–6× difference,
and the canvas payload compresses poorly. That gap is the main quantitative argument for
Kobweb over Compose Multiplatform for Web on a public-facing site.

Caveat: there is **no code splitting** (#787). Everything, including every page and every
`CssStyle`'s enclosing file, lands in one chunk. Per-page or per-group splitting is a
post-1.0 goal. If bundle size is critical for a large site, plan for multiple Kobweb apps
rather than one app with many sections.

# Versions, artifacts, and upgrading

## Two version numbers

Kobweb has a **library/plugin version** (`0.25.1`) and a **CLI version** (`0.9.23`). They
move independently and live in different repositories. A `libs.versions.toml` bump
changes the former; `brew upgrade kobweb` / `sdk upgrade kobweb` changes the latter.

## Compatibility matrix

Kobweb is built against, but does not strictly require, these versions. Keeping them in
sync avoids surprises.

| Kobweb | Compose | Kotlin |
| --- | --- | --- |
| 0.25.1+ | 1.11.1 (HTML) / 1.12.0 (Runtime) | 2.4.10 |
| 0.25.0 | 1.11.1 (HTML) / 1.11.2 (Runtime) | 2.4.0 |
| 0.24.0+ | 1.10.0 (HTML) / 1.10.2 (Runtime) | 2.3.10 |
| 0.23.3+ | 1.8.0 | 2.2.20 |
| 0.23.1+ | 1.8.0 | 2.2.10 |
| 0.23.0 | 1.8.0 | 2.2.0 |
| 0.22.0 | 1.8.0 | 2.1.21 |
| 0.21.0+ | 1.7.3 | 2.1.20 |
| 0.20.3+ | 1.7.3 | 2.1.10 |
| 0.20.1+ | 1.7.3 | 2.1.0 |
| 0.20.0 | 1.7.1 | 2.1.0 |
| 0.19.3+ | 1.7.1 | 2.0.20 |
| 0.19.1+ | 1.6.11 | 2.0.20 |
| 0.19.0 | 1.6.11 | 2.0.10 |
| 0.18.0+ | 1.6.2 | 1.9.23 |

Older rows are in
[COMPATIBILITY.md](https://github.com/varabyte/kobweb/blob/main/COMPATIBILITY.md), which
is the authoritative source. Kobweb also tracks Ktor for the server (3.5.0 at 0.25.1).

### The two Compose versions (0.24.0+)

Since 0.24.0 the runtime comes from **androidx**, not from the JetBrains Compose Gradle
plugin:

```toml
[versions]
compose-html = "1.11.1"
compose-runtime = "1.12.0"

[libraries]
compose-html-core = { module = "org.jetbrains.compose.html:html-core", version.ref = "compose-html" }
compose-runtime = { module = "androidx.compose.runtime:runtime", version.ref = "compose-runtime" }
```

They are genuinely different numbers. Do not unify them behind one version ref.

## Published artifacts

Group `com.varabyte.kobweb`:

`kobweb-core`, `kobweb-compose`, `kobweb-silk`, `silk-foundation`, `silk-widgets`,
`silk-widgets-kobweb`, `compose-html-ext`, `browser-ext`, `kobweb-worker`,
`kobweb-worker-interface`, `kobweb-api`, `kobweb-io`, `kobweb-server-plugin`,
`kobweb-common`, `kobweb-client-server-internal`, `kobweb-serialization`,
`framework-annotations`, `kobweb-processor-common`, `kobweb-ksp-ext`,
`kobweb-ksp-site-processors`, `kobweb-ksp-worker-processor`.

Group `com.varabyte.kobwebx`:

`kobwebx-markdown`, `kobwebx-frontmatter`, `kobwebx-serialization-kotlinx`,
`silk-icons-fa`, `silk-icons-ms`, `silk-icons-lucide`, `silk-icons-mdi`.

Gradle plugin IDs: `com.varabyte.kobweb.application`, `com.varabyte.kobweb.library`,
`com.varabyte.kobweb.worker`, `com.varabyte.kobwebx.markdown`.

Everything shares the single `kobweb` version. Libraries are on Maven Central and plugins
on the Gradle Plugin Portal (since 0.20.1).

`compose-html-ext` and `browser-ext` are usable standalone, without Kobweb — see
`references/10-interop-and-escape-hatches.md`.

## Upgrade procedure

1. Bump `kobweb` in `gradle/libs.versions.toml`.
2. Check the compatibility table and bump `kotlin`, `compose-html`, and `compose-runtime`
   to match.
3. Read the release notes between your version and the target — they carry the migration
   notes, and every release page repeats the exact `[versions]` block to copy.
4. Build. Deprecation warnings usually carry `ReplaceWith`, so the IDE's "Replace with"
   quick-fix does much of the work (the 0.24.1 notes warn it does not always do a great
   job).
5. Upgrade the CLI separately if a new one is out; `kobweb` tells you when one is
   available.

## Migrations that have bitten people

| Version | What changed | What to do |
| --- | --- | --- |
| **0.24.1** | Frontend network APIs reworked: bodies are `RequestBody` via `bodyOf(...)`, responses are the web `Response` object. | `window.fetch(POST, bytes)` → `window.fetch(POST, bodyOf(bytes)).bodyAsBytes()`. Deprecated overloads still work. |
| **0.24.1** | Material Design Icons deprecated upstream by Google. | Migrate `silk-icons-mdi` → `silk-icons-ms` (`Mdi…` → `Ms…`). |
| **0.24.0** | Compose runtime moved to androidx; two version refs. | Split the catalog entries as shown above. |
| **0.24.0** | `window.api.bytes(...)` → `window.api.getBytes(...)` etc. | Rename; the old names are being reclaimed for richer return types. |
| **0.23.0** | `Transferables` → `Attachments` in the worker API. | Rename; deprecated alias remains. |
| **0.23.0** | Query/dynamic params are now URL-decoded. | Remove any manual `decodeURIComponent` you had. |
| **0.22.0** | `Surface` is no longer a `Box` — vertical layouts can collapse. | Move `minHeight(100.vh)` from the `Surface` onto a `Box` child. |
| **0.22.0** | Layouts introduced. | Optional migration; the legacy "page calls the layout itself" pattern still works. |
| **0.20.1** | Artifacts moved to Maven Central / Gradle Plugin Portal. | Drop any custom Kobweb repository declaration. |
| **0.19.0** | K2 migration. | See the `docs/k2-migration.md` in that tag if coming from older. |

## Versions to skip

- **0.20.5** and **0.21.0** shipped color-mode bugs; use **0.20.6** / **0.21.1** instead
  (identical plus fixes).
- **0.20.1** has a `tryRoutingTo` bug with query params/fragments that only shows up in
  production builds; use **0.20.2**.
- **0.23.1** broke base paths; **0.23.2** fixes it.
- **0.25.0** broke server log generation; **0.25.1** fixes it.

## Testing a snapshot

Only when the maintainers ask. Snapshots live at
`https://central.sonatype.com/repository/maven-snapshots/` and must be registered for
both plugin and library resolution — see `references/01-setup-and-build.md`.

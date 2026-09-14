# Setup, project layout, and the Gradle DSL

## The CLI

The `kobweb` binary is versioned separately from the framework (CLI `0.9.23` vs library
`0.25.1`). It is a thin wrapper: it delegates to `git` for templates and to Gradle for
everything else.

```bash
brew install varabyte/tap/kobweb        # macOS / Linux
sdk install kobweb                      # SDKMAN!
scoop install varabyte/kobweb           # Windows
yay -S kobweb                           # Arch (AUR)
```

Or download a zip/tar from <https://github.com/varabyte/kobweb-cli/releases> and put
`bin/kobweb` on the PATH. Requires JDK 11+.

### Commands

| Command | Notes |
| --- | --- |
| `kobweb create [template]` | Instantiates a template (prompts if omitted). `--repo`, `--branch`. |
| `kobweb list` | Lists available templates. |
| `kobweb run` | Starts a dev server with live reload. `--env dev\|prod`, `--layout fullstack\|static`, `-f/--foreground` (only with `--notty`), `-o/--once` (disable live reload, dev only), `-p/--path`, `-t/--tty` \| `--notty`. |
| `kobweb export` | Exports the site. `--layout fullstack\|static`, `-p`, tty flags. |
| `kobweb stop` | Stops a running server. |
| `kobweb conf [query]` | Reads a value out of `.kobweb/conf.yaml`, e.g. `kobweb conf server.port`. |
| `kobweb version` | Also `kobweb -v` / `--version`. |

Gradle args can be forwarded per phase: `--gradle`, `--gradle-start`, `--gradle-export`,
`--gradle-stop`.

Use `--notty` in CI and Docker — it logs sequentially and never waits for input.

### Equivalent Gradle tasks

The CLI prints the Gradle command it runs, so you can lift it into an IDE run
configuration:

```bash
./gradlew kobwebStart -t            # dev server, -t = continuous = live reload
./gradlew kobwebStop
./gradlew kobwebExport -PkobwebReuseServer=false -PkobwebEnv=DEV \
  -PkobwebRunLayout=FULLSTACK -PkobwebBuildTarget=RELEASE -PkobwebExportLayout=FULLSTACK
./gradlew kobwebStart -PkobwebEnv=PROD -PkobwebRunLayout=FULLSTACK
```

Swap the layout properties to `STATIC` for a static site.

## Project layout

`kobweb create app` produces:

```
my-project/
├── settings.gradle.kts
├── gradle/libs.versions.toml
└── site/
    ├── build.gradle.kts
    ├── .kobweb/conf.yaml
    └── src/
        ├── jsMain/
        │   ├── kotlin/org/example/myproject/
        │   │   ├── AppEntry.kt          # @App + @InitSilk
        │   │   ├── components/{layouts,sections,widgets}/
        │   │   └── pages/               # @Page functions -> routes
        │   └── resources/
        │       ├── public/              # served verbatim at site root
        │       └── markdown/            # optional, with the markdown plugin
        └── jvmMain/kotlin/org/example/myproject/api/   # @Api handlers (full-stack only)
```

There is no `index.html` and no routing table — both are generated.

Defaults (overridable in the `kobweb {}` block): pages package `.pages`, api package
`.api`, public path `public`, generated code under `build/generated/kobweb`.

## `settings.gradle.kts`

```kotlin
pluginManagement {
    repositories { gradlePluginPortal() }
}

dependencyResolutionManagement {
    repositories {
        mavenCentral()
        google()
    }
}

rootProject.name = "my-project"
include(":site")
```

Kobweb libraries are on **Maven Central** and its plugins on the **Gradle Plugin
Portal** (since 0.20.1), so no custom repository is needed.

### Snapshots (only when the maintainers ask you to test one)

```kotlin
gradle.settingsEvaluated {
    fun RepositoryHandler.kobwebSnapshots() {
        maven("https://central.sonatype.com/repository/maven-snapshots/") {
            mavenContent {
                includeGroupByRegex("com\\.varabyte\\.kobweb.*")
                snapshotsOnly()
            }
        }
    }
    pluginManagement.repositories { kobwebSnapshots() }
    dependencyResolutionManagement.repositories { kobwebSnapshots() }
}
```

The maintainers acknowledge this is not idiomatic Gradle; it is recommended for its
simplicity and for being easy to delete later.

## Version catalog

```toml
[versions]
compose-html = "1.11.1"
compose-runtime = "1.12.0"
kobweb = "0.25.1"
kotlin = "2.4.10"

[libraries]
compose-html-core = { module = "org.jetbrains.compose.html:html-core", version.ref = "compose-html" }
compose-runtime = { module = "androidx.compose.runtime:runtime", version.ref = "compose-runtime" }
kobweb-api = { module = "com.varabyte.kobweb:kobweb-api", version.ref = "kobweb" }
kobweb-core = { module = "com.varabyte.kobweb:kobweb-core", version.ref = "kobweb" }
kobweb-silk = { module = "com.varabyte.kobweb:kobweb-silk", version.ref = "kobweb" }
kobweb-worker = { module = "com.varabyte.kobweb:kobweb-worker", version.ref = "kobweb" }
kobwebx-markdown = { module = "com.varabyte.kobwebx:kobwebx-markdown", version.ref = "kobweb" }
kobwebx-serialization-kotlinx = { module = "com.varabyte.kobwebx:kobwebx-serialization-kotlinx", version.ref = "kobweb" }
silk-icons-fa = { module = "com.varabyte.kobwebx:silk-icons-fa", version.ref = "kobweb" }
silk-icons-lucide = { module = "com.varabyte.kobwebx:silk-icons-lucide", version.ref = "kobweb" }
silk-icons-ms = { module = "com.varabyte.kobwebx:silk-icons-ms", version.ref = "kobweb" }

[plugins]
compose-compiler = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
kobweb-application = { id = "com.varabyte.kobweb.application", version.ref = "kobweb" }
kobweb-library = { id = "com.varabyte.kobweb.library", version.ref = "kobweb" }
kobweb-worker = { id = "com.varabyte.kobweb.worker", version.ref = "kobweb" }
kobwebx-markdown = { id = "com.varabyte.kobwebx.markdown", version.ref = "kobweb" }
kotlin-multiplatform = { id = "org.jetbrains.kotlin.multiplatform", version.ref = "kotlin" }
```

The `kobweb` version drives both `com.varabyte.kobweb:*` and `com.varabyte.kobwebx:*`
artifacts and all Gradle plugins.

## The site `build.gradle.kts`

```kotlin
import com.varabyte.kobweb.gradle.application.util.configAsKobwebApplication

plugins {
    alias(libs.plugins.kotlin.multiplatform)
    alias(libs.plugins.compose.compiler)
    alias(libs.plugins.kobweb.application)
    alias(libs.plugins.kobwebx.markdown)   // optional
}

group = "org.example.myproject"
version = "1.0-SNAPSHOT"

kobweb {
    app {
        index { description.set("Powered by Kobweb") }
    }
}

kotlin {
    // Frontend only. Pass includeServer = true to add a jvm target for @Api handlers.
    configAsKobwebApplication("myproject" /*, includeServer = true */)

    sourceSets {
        jsMain.dependencies {
            implementation(libs.compose.runtime)
            implementation(libs.compose.html.core)
            implementation(libs.kobweb.core)
            implementation(libs.kobweb.silk)
            implementation(libs.kobwebx.markdown)
        }
        // jvmMain.dependencies {
        //     compileOnly(libs.kobweb.api)   // provided by the Kobweb server at runtime
        // }
    }
}
```

`configAsKobwebApplication(moduleName: String? = null, includeServer: Boolean = false,
jsTargetName: String = "js", jvmTargetName: String = "jvm")` sets up `js { browser() }`
plus, optionally, a `jvm` target producing the server jar.

Note `compileOnly` for `kobweb-api`: the Kobweb server supplies it at runtime.

## Library modules

Reusable UI (and even pages and `@Api` handlers) can live in a `kobweb.library` module,
which the application module picks up automatically:

```kotlin
import com.varabyte.kobweb.gradle.library.util.configAsKobwebLibrary

plugins {
    alias(libs.plugins.kotlin.multiplatform)
    alias(libs.plugins.compose.compiler)
    alias(libs.plugins.kobweb.library)
}

kotlin {
    configAsKobwebLibrary(includeServer = true)
    sourceSets {
        jsMain.dependencies { /* kobweb-core, kobweb-silk, ... */ }
        jvmMain.dependencies { implementation(libs.kobweb.api) }
    }
}
```

`@App` is the one thing a library may **not** declare — that is a KSP error.

## The `kobweb {}` Gradle DSL

### Core (all plugin types)

| Property | Default | Purpose |
| --- | --- | --- |
| `pagesPackage` | `.pages` | Package scanned for `@Page`. |
| `apiPackage` | `.api` | Package scanned for `@Api`. |
| `publicPath` | `public` | Resource folder copied to the site root. |
| `baseGenDir` | `generated/kobweb` | Where generated sources land. |
| `kspProcessorDependency` | `com.varabyte.kobweb:kobweb-ksp-site-processors:<version>` | Leave alone. |

### `app { … }`

| Block / property | Purpose |
| --- | --- |
| `globals: MapProperty<String, String>` | Build-time constants readable at runtime via `AppGlobals.getValue("key")`. |
| `cleanUrls: Property<Boolean>` | Emit `about/index.html` instead of `about.html`. |
| `cssPrefix: Property<String>` | Prefix for generated CSS class names. |
| `index { … }` | `description`, `faviconPath`, `lang`, `head: ListProperty<HEAD.() -> Unit>`, `scriptAttributes`, `excludeHtmlForDependencies`, and `interceptUrls { … }`. |
| `index { interceptUrls { … } }` | `enableSelfHosting(excludes)` downloads external `<head>` resources (fonts, CDN CSS) at build time and serves them locally; also `replace(from, to)`, `reject(url)`, `linkRels`. |
| `server { … }` | `systemProperties` (exposed to the backend via `System.getProperty`), `remoteDebugging { enabled, host, port }`. |
| `export { … }` | See below. |

### `app { export { … } }`

| Property | Purpose |
| --- | --- |
| `browser` | `Browser.Chromium` (default), `Firefox`, `WebKit`, `Edge`. |
| `browserPath` | Use an already-installed browser instead of downloading one (env: `KOBWEB_EXPORT_BROWSER_PATH`). |
| `numThreads` | Parallel export workers; default = half the cores. Env `KOBWEB_EXPORT_NUM_THREADS` also accepts `max`, `high`, `half`. |
| `filter` | Predicate deciding which routes get exported. |
| `addExtraRoute(route, exportPath)` | Export a concrete instance of a dynamic route. |
| `timeout`, `includeSourceMap` | Per-page export timeout; ship source maps. |
| `enableTraces(...)` | Emit Playwright traces for debugging export hangs. |
| `suppressLayoutWarning`, `suppressNoRootWarning` | Silence the static/full-stack mismatch and missing-`/` warnings. |

### `markdown { … }` (with the `kobwebx.markdown` plugin)

See `references/08-markdown.md`.

### `worker { … }` (with the `kobweb.worker` plugin)

`name` (the emitted script name) and `fqcn`. See `references/07-workers.md`.

## `.kobweb/conf.yaml`

Runtime configuration for the Kobweb server; not part of the Gradle build.

```yaml
site:
  title: "My Project"
  basePath: ""          # e.g. "/my-repo" when hosting under a subdirectory

server:
  port: 8080
  files:
    dev:
      contentRoot: "build/processedResources/js/main/public"
      script: "build/kotlin-webpack/js/developmentExecutable/myproject.js"
      api: "build/libs/myproject.jar"
    prod:
      script: "build/kotlin-webpack/js/productionExecutable/myproject.js"
      siteRoot: ".kobweb/site"
  logging:
    level: DEBUG              # ALL TRACE DEBUG INFO WARN ERROR OFF
    enableConsoleLogging: true
    enableFileLogging: true
    interceptSystemOutput: true      # println -> info, System.err -> warn
    logRoot: ".kobweb/server/logs"
    clearLogsOnStart: true
    maxFileCount: null
    totalSizeCap: "10mb"
    compressHistory: true
  cors:
    hosts: []
  redirects:
    - from: "/old/([^/]*)"
      to: "/new/$1"
  streaming: { }
  nativeLibraries: []
```

`redirects` supports regexes with `$1`-style capture substitution and issues a 301.

## Common setup mistakes

- Forgetting `alias(libs.plugins.compose.compiler)` — the Compose compiler plugin is
  separate from the Kotlin plugin since Kotlin 2.0.
- Mixing up the two Compose versions (`compose-html` vs `compose-runtime`).
- Copying `kspProcessorDependency.set(...)` out of the framework's `playground` module.
- Using `implementation(libs.kobweb.api)` instead of `compileOnly` in `jvmMain`.
- Expecting `kobweb run` to work without a JDK on the PATH.

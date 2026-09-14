# Web workers

Kobweb wraps the browser's Web Worker API so that off-main-thread work is typed,
`@Serializable`-friendly, and Compose-lifecycle aware. Use it for anything that would
otherwise jank the UI: image processing, parsing, crypto, prime hunting, physics.

A worker is its **own Gradle module** — it compiles to a separate JS script that the
browser loads independently. That separation is a hard requirement, not a style choice.

## The worker module

```kotlin
// workers/sum/build.gradle.kts
import com.varabyte.kobweb.gradle.worker.util.configAsKobwebWorker

plugins {
    alias(libs.plugins.kotlin.multiplatform)
    alias(libs.plugins.kotlinx.serialization)
    alias(libs.plugins.kobweb.worker)
}

kotlin {
    configAsKobwebWorker("sum-worker")      // -> the emitted script name
    sourceSets {
        jsMain.dependencies {
            api(libs.kotlinx.serialization.json)
            implementation(libs.kobweb.worker)
            implementation(libs.kobwebx.serialization.kotlinx)
        }
    }
}
```

The site module just depends on it: `implementation(project(":workers:sum"))`. Kobweb
copies the worker script into the site output automatically — including when the worker
is reached transitively through a Kobweb library module.

## The factory

Exactly **one** `WorkerFactory` implementation per worker module.

```kotlin
@Serializable data class SumInputs(val a: Int, val b: Int)
@Serializable data class SumOutput(val sum: Int)

internal class SumWorkerFactory : WorkerFactory<SumInputs, SumOutput> {
    override fun createStrategy(postOutput: OutputDispatcher<SumOutput>) =
        WorkerStrategy<SumInputs> { input ->
            postOutput(SumOutput(input.a + input.b))
        }

    override fun createIOSerializer() = Json.createIOSerializer<SumInputs, SumOutput>()
}
```

Rules enforced by the worker KSP processor (all errors unless noted):

| Rule | Why |
| --- | --- |
| Exactly one `WorkerFactory` per module | The generated `Worker` class is derived from it. |
| Public, no-arg constructor | The generated wrapper instantiates it. |
| Not `private` | Codegen must see it. `internal` is recommended — being `public` only produces a warning (`@Suppress("PUBLIC_WORKER_FACTORY")` to silence). |
| Input and output types must be `public` | The application module consumes them. |
| Class name must end with `WorkerFactory` (and not *be* `WorkerFactory`) | `SumWorkerFactory` → generated `SumWorker`. Override with `kobweb { worker { fqcn.set(…) } }`. |

`fqcn` supports leading/trailing dots: `".PiWorker"` keeps the factory's package,
`"com.mysite."` keeps the inferred class name.

If you only need strings, implement `WorkerFactory<String, String>` and return
`createPassThroughSerializer()`.

## Using the worker

```kotlin
@Page
@Composable
fun SumPage() {
    var sum by remember { mutableStateOf(0) }
    val worker = rememberWorker { SumWorker { output -> sum = output.sum } }

    var a by remember { mutableStateOf(0) }
    LaunchedEffect(a) { worker.postInput(SumInputs(a, 2)) }

    Text("Sum: $sum")
}
```

`rememberWorker` ties the worker's lifetime to the composition and terminates it on
dispose.

## Attachments (transferables and structured clones)

Beyond the serialized payload you can attach binary data and structured-clonable objects
(`File`, `Blob`, `ArrayBuffer`, `ImageBitmap`, …):

```kotlin
worker.postInput(input, Attachments { add("image", arrayBuffer) })
```

Renamed from `Transferables` to `Attachments` in 0.23.0 when structured-clone support was
added; the old name is deprecated. Raw JSON key/value pairs can also be attached as an
escape hatch for types Kobweb does not wrap.

Serialization failures on either side now surface as exceptions (0.23.0) instead of being
swallowed — the usual cause is a forgotten `@Serializable`.

## Environment differences inside a worker

There is no `window`; touching it throws.

| In a page | In a worker |
| --- | --- |
| `window.fetch(...)` | `self.fetch(...)` |
| `CoroutineScope(window.asCoroutineDispatcher())` | `CoroutineScope(self.asCoroutineDispatcher())` |

(Both added in 0.24.1.)

Workers respect the site's configured base path as of 0.25.1.

## When *not* to use one

Workers cost a separate module, a serialization boundary, and a script download. If the
work is short, or mostly I/O (where a coroutine already yields), keep it on the main
thread. Reach for a worker when a single synchronous computation would block a frame.

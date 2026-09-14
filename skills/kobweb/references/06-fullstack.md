# Full-stack: API routes, streams, the server

## Enabling the backend

```kotlin
kotlin {
    configAsKobwebApplication(includeServer = true)
    sourceSets {
        jvmMain.dependencies {
            compileOnly(libs.kobweb.api)   // the Kobweb server provides it at runtime
            implementation(libs.exposed)   // your own deps: normal implementation
        }
        commonMain.dependencies { /* types shared by js + jvm */ }
    }
}
```

Backend code lives in `src/jvmMain/kotlin/<package>/api/` (configurable via
`kobweb.apiPackage`). Shared `@Serializable` DTOs go in `commonMain`.

The output is a jar (`build/libs/<project>.jar`) that `.kobweb/conf.yaml` points at; the
Kobweb Ktor server loads it and mounts your handlers under `/api/`.

## `@Api` routes

```kotlin
// src/jvmMain/kotlin/org/example/api/user/Fetch.kt   ->  GET /api/user/fetch
@Api
suspend fun fetch(ctx: ApiContext) {
    val id = ctx.req.params["id"] ?: run {
        ctx.res.status = 400
        return
    }
    ctx.res.body = bodyOf("user $id")
}
```

Route derivation mirrors `@Page`: file path + kebab-cased file name, prefixed with
`/api/`. `@Api("override")` follows the same leading-/trailing-slash rules as `@Page`.
Dynamic segments work too — `@Api("/article/{articleId}")` makes
`/api/article/123` readable as `ctx.req.params["articleId"]`; a bare `@Api("{}")` derives
the capture name from the file name.

Handlers may be `suspend` or plain functions; `suspend` is the idiomatic default since
most handlers do I/O.

**A handler that sets nothing returns 404.** Setting `ctx.res.body` implies 200.

### `ApiContext`

| Member | Purpose |
| --- | --- |
| `ctx.req` | `Request`: `method`, `params` (query + dynamic), `queryParams`, `headers`, `cookies`, `body: Body?`, `connection`, `data` (per-request scratch, set by interceptors) |
| `ctx.res` | `Response`: `status`, `body`, `headers` (multi-value since 0.24.1), `data` |
| `ctx.data` | Read-only service registry populated by `@InitApi` |
| `ctx.env` | `Environment.DEV` / `PROD` |
| `ctx.logger` | `trace`/`debug`/`info`/`warn`/`error` into the server log |

Since 0.25.1 plain `println` and `System.err.println` inside a handler are captured as
`info` / `warn` log entries (`server.logging.interceptSystemOutput`).

### Bodies

`Body` is constructed through `bodyOf(...)` factories:

```kotlin
ctx.res.body = bodyOf("hello")                        // text/plain
ctx.res.body = bodyOf(bytes)                          // application/octet-stream
ctx.res.body = bodyOf(inputStream, contentType = "application/pdf")
ctx.res.body = Body.json("""{"ok":true}""")
```

With `kobwebx-serialization-kotlinx` on the classpath:

```kotlin
@Serializable data class User(val id: Int, val name: String)

val incoming: User? = ctx.req.body?.decode<User>()
ctx.res.body = bodyOf(User(1, "Ada"))                 // application/json
```

Redirects: `ctx.res.setAsRedirect("/new-path", status = 307)`.

### Multipart uploads (0.24.0+)

```kotlin
@Api
suspend fun upload(ctx: ApiContext) {
    if (ctx.req.method != HttpMethod.POST) return
    val mp = ctx.req.body?.multipart() ?: return
    mp.forEachPart { part ->
        val name = part.contentDisposition?.name
        (part.extras as? Multipart.Extras.File)?.originalFileName
        val bytes = part.bytes()
        // …
    }
}
```

### `@InitApi` — services

```kotlin
@InitApi
fun initDatabase(ctx: InitApiContext) {
    ctx.data.add<Database>(MutableDatabase(ctx.env))
}

@Api
suspend fun list(ctx: ApiContext) {
    val db = ctx.data.getValue<Database>()
    // …
}
```

Values registered here live for the whole server lifetime. Register a listener for
`DisposeEvent` (`ctx.events`) to clean up on shutdown.

### `@ApiInterceptor` — middleware

```kotlin
@ApiInterceptor
suspend fun intercept(ctx: ApiInterceptorContext): Response {
    if (ctx.path == "/api/health") return Response().apply { body = bodyOf("ok") }
    ctx.req.data.add(RequestId(UUID.randomUUID().toString()))
    return ctx.dispatcher.dispatch()
}
```

One interceptor per project. It sees every API request, can short-circuit it, can mutate
the request before dispatching, and can post-process the returned `Response`. The
`data` property on `Request`/`Response` (0.24.0+) is how an interceptor passes values to
a handler.

## API streams

Streams are a request/response-free, bidirectional channel. **They are not raw
websockets**: the server opens a *single* websocket and multiplexes every named stream
over it — chosen so multiple streams cost one connection and so live reloading can
recreate handlers without tearing down the socket.

Server (`src/jvmMain/.../api/Echo.kt` → stream id `echo`):

```kotlin
val echo = ApiStream { ctx -> ctx.stream.send(ctx.text) }
```

Or the full form for connect/disconnect hooks:

```kotlin
val chat = object : ApiStream() {
    override suspend fun onClientConnected(ctx: ClientConnectedContext) {
        ctx.stream.broadcast("${ctx.stream.id} joined")
    }
    override suspend fun onTextReceived(ctx: TextReceivedContext) {
        ctx.stream.broadcast(ctx.text)
    }
    override suspend fun onClientDisconnected(ctx: ClientDisconnectedContext) { /* … */ }
}
```

A public top-level `ApiStream` property is auto-registered; `@Api` on it is optional and
only needed to override the id. Note the id is **not** URL-prefixed with `/api/`.

Client:

```kotlin
@Page
@Composable
fun ChatPage() {
    val stream = rememberApiStream("chat") { ctx ->
        console.log("received: ${ctx.text}")
    }
    Button(onClick = { stream.send("hello!") }) { Text("Send") }
}
```

### Routes vs streams

| Use `@Api` routes when | Use `ApiStream` when |
| --- | --- |
| simple request/response | high-frequency messaging |
| stateless handlers | the server must push updates |
| you want to scale horizontally | real-time events, chat, presence |
| you want to `curl` the endpoint | a persistent connection is natural |

Streams keep per-connection state on one node, so horizontal scaling needs external
infrastructure (Redis pub/sub or similar). Kobweb provides nothing for that.

## The frontend HTTP API

Two facades on `window`, both `suspend`, both type-safe:

- `window.api.*` — targets **your** Kobweb API routes; prepends `/api/` and the base path.
- `window.http.*` — targets **any** URL.

```kotlin
val users = window.api.get("users").bodyAs<List<User>>()
val created = window.http.post<User>("https://example.com/posts", body = newUser).bodyAs<User>()
```

Verbs: `get`, `post`, `put`, `patch`, `delete`, `head`, `options`, and `query`
(RFC 10008 — a `GET` that may carry a body; added 0.25.1). Each has:

- a `…Bytes` variant returning raw bytes (`getBytes`, `postBytes`, …);
- a `try…` variant that returns `null` instead of throwing on a non-2xx response
  (`tryGet`, `tryPost`, …) — the right default for UI code;
- an optional `redirect` parameter (`RequestRedirect.MANUAL`, …).

Site-wide defaults: `FetchDefaults.Headers = mapOf("X-Api-Key" to key)`.

> **Migration (0.24.1).** Request bodies are now `RequestBody` objects built with
> `bodyOf(...)`, and responses are the standard web `Response` object rather than raw
> bytes. Old: `window.fetch(HttpMethod.POST, bodyBytes)`. New:
> `window.fetch(HttpMethod.POST, bodyOf(bodyBytes)).bodyAsBytes()`. The deprecated
> overloads carry `ReplaceWith` hints.

Inside a **worker** there is no `window` — use `self.fetch(...)` and
`CoroutineScope(self.asCoroutineDispatcher())`.

## Server configuration

Everything runtime-side lives in `.kobweb/conf.yaml` (see
`references/01-setup-and-build.md`): port, logging, CORS hosts, redirects, streaming,
native libraries.

Build-script values can be pushed into the server as system properties:

```kotlin
kobweb { app { server { systemProperties.put("db.url", "…") } } }
```

```kotlin
val dbUrl = System.getProperty("db.url")!!
```

Remote JVM debugging: `kobweb { app { server { remoteDebugging { enabled.set(true); port.set(5005) } } } }`.

## Kobweb server plugins

For behaviour the `@Api` layer cannot express — custom Ktor plugins, extra routes,
authentication filters — write a **server plugin**:

1. A JVM module depending on `com.varabyte.kobweb:kobweb-server-plugin`, implementing
   `KobwebServerPlugin` (hooks into Ktor's `Application` and routing events).
2. Drop the built jar into `.kobweb/server/plugins/`.

Changing a server plugin requires a full server restart — live reload does not pick it
up.

## Missing pieces

- **No built-in authentication.** Whether auth belongs in a server plugin or a `kobwebx`
  library is still an open design question (#254, milestone 1.1). Today: hand-rolled
  cookie/JWT handling in an interceptor, or a server plugin.
- **Kobweb must own the server.** Running `@Api` handlers inside an existing Ktor/Spring
  app is an open epic (#22). The supported workaround is a static export served by your
  own backend — see `references/10-interop-and-escape-hatches.md`.
- **No server-side rendering** (#114) and **no hydration** (#113).

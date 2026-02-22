# Ecosystem and Packages

Effect 4.0 is organized as a monorepo with a core `effect` package and several companion packages for platform-specific capabilities, testing, observability, and framework integration.

---

## Core Package: `effect`

The `effect` package contains the full Effect runtime, data types, and most modules you need for building applications. In v4, `@effect/schema` has been merged into the core package.

### Module Organization

Modules are imported directly from `effect` or from subpaths:

```ts
// Top-level imports
import { Effect, Stream, Schema, Layer, ServiceMap, Ref, Queue } from "effect"

// Testing subpath -- all testing utilities are under "effect/testing"
import { TestClock, TestConsole, TestSchema, FastCheck } from "effect/testing"
// There is no "effect/testing/FastCheck" subpath export; use the above instead
```

### Key Module Categories

| Category | Modules |
|---|---|
| Core | `Effect`, `Exit`, `Cause`, `Scope`, `Function`, `Pipeable` |
| Data Types | `Option`, `Result`, `Array`, `Chunk`, `HashMap`, `HashSet`, `Duration`, `DateTime`, `Data`, `Brand` |
| Services | `ServiceMap`, `Layer`, `Config`, `ConfigProvider`, `ManagedRuntime` |
| References | `Ref`, `SynchronizedRef`, `SubscriptionRef`, `ScopedRef` |
| Concurrency | `Fiber`, `FiberHandle`, `FiberMap`, `FiberSet`, `Deferred`, `Semaphore`, `Latch` |
| Queues & Pub/Sub | `Queue`, `PubSub` |
| Resource Mgmt | `Pool`, `Cache`, `ScopedCache`, `RcRef`, `RcMap`, `LayerMap` |
| Streaming | `Stream`, `Sink`, `Channel`, `Pull` |
| Schema | `Schema`, `SchemaAST`, `SchemaParser`, `SchemaGetter`, `SchemaIssue`, `SchemaTransformation` |
| Transactions | `TxRef`, `TxHashMap`, `TxHashSet`, `TxQueue`, `TxChunk`, `TxSemaphore` |
| Pattern Matching | `Match` |
| Scheduling | `Schedule`, `Cron` |
| Observability | `Logger`, `LogLevel`, `Metric`, `Tracer` |
| Platform Abstractions | `FileSystem`, `Path`, `Terminal`, `Console` |
| Utilities | `Equal`, `Hash`, `Order`, `Predicate`, `Equivalence`, `Encoding`, `Random` |

---

## Platform Packages

Platform packages provide runtime-specific implementations for the abstract services defined in the core `effect` package (e.g., `FileSystem`, `HttpClient`, `HttpServer`).

### @effect/platform-node

For Node.js applications. Provides concrete implementations of:

| Module | Description |
|---|---|
| `NodeRuntime` | `runMain` for Node.js with signal handling, exit codes, error reporting |
| `NodeFileSystem` | Node.js `fs` implementation of `FileSystem` |
| `NodeHttpClient` | HTTP client using Node.js `http`/`https` |
| `NodeHttpServer` | HTTP server implementation |
| `NodeHttpPlatform` | HTTP platform utilities |
| `NodeSocket` / `NodeSocketServer` | TCP socket support |
| `NodeWorker` / `NodeWorkerRunner` | Worker thread support |
| `NodeTerminal` | Terminal I/O |
| `NodeStream` | Bridging Node.js streams with Effect streams |
| `NodeSink` | Stream sinks for Node.js writable streams |
| `NodePath` | Path utilities |
| `NodeStdio` | Standard I/O streams |
| `NodeRedis` | Redis client integration |
| `NodeMultipart` | Multipart form data parsing |
| `NodeChildProcessSpawner` | Child process management |
| `NodeClusterHttp` / `NodeClusterSocket` | Cluster support |
| `NodeServices` | Pre-composed layer of all Node.js services |
| `Undici` | HTTP client via undici |

#### Running an Effect Program

```ts
import { NodeRuntime } from "@effect/platform-node"
import { Effect } from "effect"

const program = Effect.gen(function*() {
  yield* Effect.log("Hello from Node.js!")
})

NodeRuntime.runMain(program)
// Options:
// - disableErrorReporting: boolean
// - disablePrettyLogger: boolean
// - teardown: Runtime.Teardown
```

While v4 has built-in fiber keep-alive (so `Effect.runPromise` will keep the process alive), `NodeRuntime.runMain` is still recommended for production applications because it provides signal handling (SIGINT/SIGTERM), exit code management, and structured error reporting.

### @effect/platform-bun

Same architecture as platform-node but for the Bun runtime:

| Module | Description |
|---|---|
| `BunRuntime` | `runMain` for Bun |
| `BunFileSystem` | Bun file system |
| `BunHttpClient` / `BunHttpServer` | HTTP support |
| `BunSocket` / `BunSocketServer` | Socket support |
| `BunWorker` / `BunWorkerRunner` | Worker support |
| `BunServices` | Pre-composed layer |
| `BunRedis` | Redis integration |
| And more... | |

### @effect/platform-browser

For browser applications:

| Module | Description |
|---|---|
| `BrowserRuntime` | `runMain` for browsers |
| `BrowserHttpClient` | Fetch-based HTTP client |
| `BrowserSocket` | WebSocket support |
| `BrowserStream` | ReadableStream bridging |
| `BrowserWorker` / `BrowserWorkerRunner` | Web Worker support |
| `BrowserKeyValueStore` | localStorage/sessionStorage |
| `Clipboard` | Clipboard API |
| `Geolocation` | Geolocation API |
| `Permissions` | Permissions API |

---

## Testing: @effect/vitest

The `@effect/vitest` package integrates Effect with the Vitest test runner, providing Effect-aware test functions that handle scoping, service provision, and lifecycle management.

### Setup

```ts
import { expect, it, layer } from "@effect/vitest"
import { Effect, Layer, ServiceMap } from "effect"
```

### Key APIs

```ts
// Run an effect as a test
it.effect("test name", () =>
  Effect.gen(function*() {
    const result = yield* Effect.succeed(42)
    expect(result).toBe(42)
  })
)

// Run with the live clock (not TestClock)
it.live("real timing test", () =>
  Effect.gen(function*() {
    yield* Effect.sleep("100 millis")
    // Actually waits 100ms
  })
)

// Property-based testing
it.prop("commutativity", [Schema.Number, Schema.Number], ({ 0: a, 1: b }) => {
  expect(a + b).toBe(b + a)
})
```

### Layer Sharing

Share a `Layer` across multiple tests. The layer is built once and reused:

```ts
import { expect, layer } from "@effect/vitest"
import { Effect, Layer, ServiceMap } from "effect"

class Database extends ServiceMap.Service<Database, {
  readonly query: (sql: string) => Effect.Effect<string>
}>()("Database") {
  static layer = Layer.succeed(this)({
    query: (sql) => Effect.succeed(`result: ${sql}`)
  })
}

layer(Database.layer)("database tests", (it) => {
  it.effect("can query", () =>
    Effect.gen(function*() {
      const db = yield* Database
      const result = yield* db.query("SELECT 1")
      expect(result).toBe("result: SELECT 1")
    })
  )

  // Nested layers are composed with parent layers
  it.layer(AnotherService.layer)("with another service", (it) => {
    it.effect("has both services", () =>
      Effect.gen(function*() {
        const db = yield* Database
        const other = yield* AnotherService
        // Both available
      })
    )
  })
})
```

### TestClock

The `TestClock` allows deterministic testing of time-dependent effects:

```ts
import { Effect, Fiber, Option } from "effect"
import { TestClock } from "effect/testing"

it.effect("timeout test", () =>
  Effect.gen(function*() {
    const fiber = yield* Effect.sleep("5 minutes").pipe(
      Effect.timeout("1 minute"),
      Effect.forkChild
    )

    // Advance time by 1 minute
    yield* TestClock.adjust("1 minute")

    const result = yield* Fiber.join(fiber)
    expect(result).toEqual(Option.none())
  })
)
```

The `TestClock` interface:

```ts
interface TestClock extends Clock {
  adjust(duration: Duration.Input): Effect<void>   // Advance time
  setTime(timestamp: number): Effect<void>          // Set absolute time
  withLive<A, E, R>(effect: Effect<A, E, R>): Effect<A, E, R>  // Use real clock
}
```

### TestConsole

Captures console output for assertions:

```ts
import { TestConsole } from "effect/testing"
```

### FastCheck Integration

Property-based testing via fast-check, integrated with Schema:

```ts
import { FastCheck } from "effect/testing"
// Note: "effect/testing/FastCheck" is NOT a valid subpath export.
// FastCheck is exported as a named export from "effect/testing".

// Use Schema-derived arbitraries or raw fast-check arbitraries
it.prop("reverse is involution",
  [Schema.Array(Schema.Number)],
  ({ 0: arr }) => {
    const reversed = arr.slice().reverse().reverse()
    expect(reversed).toEqual(arr)
  }
)
```

---

## Observability: @effect/opentelemetry

Integrates Effect's tracing and metrics with OpenTelemetry:

| Module | Description |
|---|---|
| `Tracer` | OpenTelemetry tracer implementation |
| `Metrics` | OpenTelemetry metrics exporter |
| `Logger` | OpenTelemetry log exporter |
| `Resource` | OTel resource configuration |
| `NodeSdk` | Node.js SDK setup (traces + metrics + logs) |
| `WebSdk` | Browser SDK setup |

---

## Atom Packages

UI framework integrations for reactive Effect-based state:

| Package | Framework |
|---|---|
| `@effect/atom-react` | React |
| `@effect/atom-solid` | Solid.js |
| `@effect/atom-vue` | Vue.js |

---

## FileSystem Abstraction

The core `effect` package defines abstract `FileSystem` and `Path` services. Platform packages provide implementations.

```ts
import { Effect, FileSystem } from "effect"

const program = Effect.gen(function*() {
  const fs = yield* FileSystem.FileSystem

  yield* fs.makeDirectory("./temp", { recursive: true })
  yield* fs.writeFileString("./temp/hello.txt", "Hello, World!")

  const content = yield* fs.readFileString("./temp/hello.txt")
  console.log(content) // "Hello, World!"

  const stats = yield* fs.stat("./temp/hello.txt")
  console.log(stats.size)
})
```

This effect requires a `FileSystem` service. Provide it via the platform layer:

```ts
import { NodeFileSystem } from "@effect/platform-node"

program.pipe(Effect.provide(NodeFileSystem.layer))
```

---

## Unstable Modules

Effect 4.0 includes a set of **unstable modules** under the `effect/unstable/` subpath. These provide higher-level capabilities built on top of core Effect primitives -- HTTP servers/clients, RPC, SQL, CLI building, AI integration, distributed systems, and more. They are shipped as part of the `effect` package but their APIs may change between minor versions.

```ts
// Import pattern for unstable modules
import { HttpClient, HttpServer, HttpRouter } from "effect/unstable/http"
import { HttpApi, HttpApiBuilder, HttpApiClient } from "effect/unstable/httpapi"
import { Rpc, RpcGroup, RpcClient, RpcServer } from "effect/unstable/rpc"
import { SqlClient, SqlSchema, Migrator } from "effect/unstable/sql"
import { Command, Flag, Argument } from "effect/unstable/cli"
import { LanguageModel, Tool, Toolkit, Chat } from "effect/unstable/ai"
import { Entity, Sharding, Message } from "effect/unstable/cluster"
import { Workflow, Activity } from "effect/unstable/workflow"
```

---

### HTTP (`effect/unstable/http`)

A complete HTTP client and server framework. The HTTP client provides methods for all standard HTTP verbs, request/response processing pipelines, and middleware. The server provides routing, middleware, and static file serving.

**Key exports:** `HttpClient`, `HttpServer`, `HttpRouter`, `HttpClientRequest`, `HttpClientResponse`, `HttpServerRequest`, `HttpServerResponse`, `HttpMiddleware`, `HttpBody`, `Headers`, `Cookies`, `UrlParams`, `Multipart`, `FetchHttpClient`, `HttpEffect`, `Etag`, `Template`, `FindMyWay`, `HttpIncomingMessage`, `HttpMethod`, `HttpPlatform`, `HttpServerRespondable`, `HttpTraceContext`, `Multipasta`

#### HttpClient

The `HttpClient` service provides `.get`, `.post`, `.put`, `.patch`, `.del`, `.head`, and `.options` methods, each returning `Effect<HttpClientResponse, HttpClientError, R>`. The client supports preprocessing (request transforms), postprocessing (response transforms), and retry logic.

```ts
import { Effect } from "effect"
import { HttpClient, HttpClientRequest } from "effect/unstable/http"

const program = Effect.gen(function*() {
  const client = yield* HttpClient.HttpClient

  // Simple GET request
  const response = yield* client.get("https://api.example.com/users")
  const users = yield* response.json

  // POST with JSON body
  // Use HttpClientRequest.bodyJsonUnsafe to set a JSON body on a request,
  // or pass an HttpBody.HttpBody via the `body` option
  const postReq = HttpClientRequest.post("https://api.example.com/users").pipe(
    HttpClientRequest.bodyJsonUnsafe({ name: "Alice" })
  )
  const created = yield* client.execute(postReq)

  return users
})
```

#### HttpRouter

A radix-tree based router (`FindMyWay`) that maps HTTP methods and URL patterns to handlers. Supports path parameters, wildcard routes, and middleware.

```ts
import { Effect } from "effect"
import { HttpRouter, HttpServerResponse } from "effect/unstable/http"

const setupRoutes = Effect.gen(function*() {
  const router = yield* HttpRouter.HttpRouter

  // Path params are NOT on the request object; access them via HttpRouter.params
  // (which yields a Record<string, string | undefined> from RouteContext).
  yield* router.add("GET", "/hello/:name",
    Effect.gen(function*() {
      const { name } = yield* HttpRouter.params
      return HttpServerResponse.text(`Hello, ${name}!`)
    })
  )

  // Handlers can be a static Effect (not a function) or (request) => Effect
  yield* router.add("POST", "/users",
    Effect.gen(function*() {
      // Access the request body via HttpServerRequest service
      return HttpServerResponse.json({ created: true })
    })
  )
})
```

#### HttpServer

The `HttpServer` service provides a `serve` method that takes an HTTP effect and binds it to a server. Platform packages provide concrete implementations.

The recommended high-level API is `HttpRouter.serve(appLayer)`, which takes a `Layer` (built with `HttpRouter.add` / `HttpRouter.use` helpers) and returns a `Layer` that wires everything together with the server, logger, and router.

```ts
import { Effect, Layer } from "effect"
import { HttpRouter, HttpServer, HttpServerResponse } from "effect/unstable/http"

// Build a route layer using HttpRouter.add (returns a Layer, not a function call)
const routeLayer = HttpRouter.add(
  "GET",
  "/",
  Effect.succeed(HttpServerResponse.text("OK"))
)

// HttpRouter.serve(appLayer) composes the router and HttpServer into a Layer.
// Provide a concrete HttpServer implementation (e.g. from NodeHttpServer) to run it.
const serverLayer = HttpRouter.serve(routeLayer)

// Alternatively, use HttpServer.serve directly with an Effect<HttpServerResponse>:
// const serverLayer = HttpServer.serve(router.asHttpEffect())
```

---

### HTTP API (`effect/unstable/httpapi`)

A declarative, schema-first API definition layer built on top of `effect/unstable/http`. Define your API contract with schemas, then derive both server handlers and type-safe clients from the same definition. Includes automatic OpenAPI spec generation and Swagger UI.

**Key exports:** `HttpApi`, `HttpApiGroup`, `HttpApiEndpoint`, `HttpApiBuilder`, `HttpApiClient`, `HttpApiMiddleware`, `HttpApiSecurity`, `HttpApiSchema`, `HttpApiScalar`, `HttpApiSwagger`, `OpenApi`

#### Defining an API

APIs are built from groups of endpoints. Each endpoint specifies its HTTP method, path, payload schemas, success/error response schemas, and middleware.

```ts
import { Schema } from "effect"
import {
  HttpApi,
  HttpApiEndpoint,
  HttpApiGroup,
  HttpApiSchema
} from "effect/unstable/httpapi"

// Define an endpoint using the options object (no chained builder methods)
const getUser = HttpApiEndpoint.get("getUser", "/users/:id", {
  params: Schema.Struct({ id: Schema.NumberFromString }),
  success: Schema.Struct({
    id: Schema.Number,
    name: Schema.String
  })
})

const createUser = HttpApiEndpoint.post("createUser", "/users", {
  payload: Schema.Struct({ name: Schema.String, email: Schema.String }),
  success: Schema.Struct({ id: Schema.Number, name: Schema.String })
})

// Group endpoints
const usersGroup = HttpApiGroup.make("users").add(getUser, createUser)

// Create the API
const myApi = HttpApi.make("myApi").add(usersGroup)
```

#### Implementing handlers

Use `HttpApiBuilder.group` to implement handlers for each endpoint in a group, and `HttpApiBuilder.layer` to register the API with a router.

```ts
import { Effect, Layer } from "effect"
import { HttpApiBuilder } from "effect/unstable/httpapi"

const usersHandlers = HttpApiBuilder.group(myApi, "users", (handlers) =>
  handlers
    .handle("getUser", ({ params }) =>
      Effect.succeed({ id: params.id, name: "Alice" })
    )
    .handle("createUser", ({ payload }) =>
      Effect.succeed({ id: 1, name: payload.name })
    )
)

// Register the API with the router, optionally serving an OpenAPI spec
const apiLayer = HttpApiBuilder.layer(myApi, {
  openapiPath: "/docs/openapi.json"
})
```

#### Derived client

Generate a fully typed HTTP client from the same API definition:

```ts
import { Effect } from "effect"
import { HttpApiClient } from "effect/unstable/httpapi"

const program = Effect.gen(function*() {
  const client = yield* HttpApiClient.make(myApi)
  const user = yield* client.users.getUser({ params: { id: 42 } })
  console.log(user.name)
})
```

---

### RPC (`effect/unstable/rpc`)

A typed Remote Procedure Call system with schema-based serialization, middleware, and transport-agnostic design. Supports HTTP, WebSocket, and Worker transports. The RPC system uses the same `Schema` types for automatic serialization/deserialization.

**Key exports:** `Rpc`, `RpcGroup`, `RpcClient`, `RpcServer`, `RpcMiddleware`, `RpcSerialization`, `RpcSchema`, `RpcMessage`, `RpcClientError`, `RpcTest`, `RpcWorker`

#### Defining RPCs

```ts
import { Schema } from "effect"
import { Rpc, RpcGroup } from "effect/unstable/rpc"

// Define individual procedures
const GetUser = Rpc.make("GetUser")
  .setPayload(Schema.Struct({ id: Schema.Number }))
  .setSuccess(Schema.Struct({ id: Schema.Number, name: Schema.String }))
  .setError(Schema.String)

const ListUsers = Rpc.make("ListUsers")
  .setSuccess(Schema.Array(Schema.Struct({ id: Schema.Number, name: Schema.String })))

// Group procedures together -- RpcGroup.make takes spread Rpc args (no name argument)
const UsersApi = RpcGroup.make(GetUser, ListUsers)
```

#### Implementing handlers

```ts
import { Effect, Layer } from "effect"

// Implement all handlers in one go, returning a Layer
const handlersLayer = UsersApi.toLayer({
  GetUser: ({ id }) => Effect.succeed({ id, name: "Alice" }),
  ListUsers: () => Effect.succeed([{ id: 1, name: "Alice" }])
})
```

#### Client usage

The `RpcClient` type is auto-derived from the group definition, providing a typed record of callable procedures:

```ts
import { RpcClient } from "effect/unstable/rpc"

// RpcClient<typeof UsersApi> gives you:
// { GetUser: (payload) => Effect<...>, ListUsers: (payload) => Effect<...> }
```

RPC servers can be served over HTTP (`RpcServer.layerHttp`, `RpcServer.toHttpEffect`), WebSockets (`RpcServer.layerProtocolWebsocket`, `RpcServer.makeProtocolSocketServer`), or Workers (`RpcServer.layerProtocolWorkerRunner`, `RpcWorker`).

---

### SQL (`effect/unstable/sql`)

An abstract SQL client with tagged template literal queries, transactions, schema-based result mapping, batch resolvers, streaming, and migrations.

**Key exports:** `SqlClient`, `SqlSchema`, `SqlModel`, `SqlResolver`, `SqlConnection`, `SqlError`, `SqlStream`, `Statement`, `Migrator`

#### Basic usage

```ts
import { Effect } from "effect"
import { SqlClient } from "effect/unstable/sql"

const program = Effect.gen(function*() {
  const sql = yield* SqlClient.SqlClient

  // Tagged template literal queries
  const users = yield* sql`SELECT * FROM users WHERE active = ${true}`

  // Transactions
  const result = yield* sql.withTransaction(
    Effect.gen(function*() {
      yield* sql`INSERT INTO users (name) VALUES (${"Alice"})`
      yield* sql`INSERT INTO audit_log (action) VALUES (${"user_created"})`
      return yield* sql`SELECT * FROM users WHERE name = ${"Alice"}`
    })
  )
})
```

#### SqlSchema and SqlModel

`SqlSchema` maps query results through Effect `Schema` types. `SqlModel` provides a higher-level CRUD repository pattern:

```ts
import { Effect } from "effect"
import { SqlClient, SqlModel, SqlSchema } from "effect/unstable/sql"

// SqlModel.makeRepository creates insert, update, findById, delete operations
const repo = Effect.gen(function*() {
  const { insert, findById, update, delete: del } = yield* SqlModel.makeRepository(
    UserModel,
    { tableName: "users", spanPrefix: "UserRepo", idColumn: "id" }
  )

  const user = yield* insert({ name: "Alice", email: "alice@example.com" })
  const found = yield* findById(user.id)
})
```

The `Migrator` module handles schema migrations with a file-based or loader-based approach.

---

### CLI (`effect/unstable/cli`)

A type-safe CLI framework for building command-line applications with automatic help generation, argument parsing, flag handling, shell completions, and interactive prompts.

**Key exports:** `Command`, `Flag`, `Argument`, `Prompt`, `CliError`, `CliOutput`, `HelpDoc`, `Primitive`

#### Building a CLI

```ts
import { Console, Effect } from "effect"
import { Argument, Command, Flag } from "effect/unstable/cli"

// Define flags and arguments
const greet = Command.make("greet", {
  name: Flag.string("name").pipe(Flag.withDescription("Name to greet")),
  loud: Flag.boolean("loud").pipe(Flag.withDefault(false)),
  times: Flag.integer("count").pipe(Flag.optional)
}, (config) =>
  Effect.gen(function*() {
    const msg = config.loud ? config.name.toUpperCase() : config.name
    yield* Console.log(`Hello, ${msg}!`)
  })
)

// Run the CLI
const main = Command.run(greet, { version: "1.0.0" })
```

#### Subcommands

Commands can have subcommands for complex CLI hierarchies:

```ts
import { Command } from "effect/unstable/cli"

const deploy = Command.make("deploy", {
  env: Flag.string("env"),
  force: Flag.boolean("force")
}, (config) => /* ... */)

const rollback = Command.make("rollback", {
  version: Argument.string("version")
}, (config) => /* ... */)

// withSubcommands takes an array, not spread arguments
const app = Command.make("myapp")
  .pipe(Command.withSubcommands([deploy, rollback]))
```

Flag types include `Flag.string`, `Flag.boolean`, `Flag.integer`, `Flag.float`, `Flag.date`, `Flag.file`, `Flag.directory`, `Flag.choice`, and `Flag.redacted` (for `Redacted` values -- note: the function is `Flag.redacted`, not `Flag.secret`). Arguments use `Argument.string`, `Argument.integer`, `Argument.file`, `Argument.directory`, and `Argument.choice`. Both support `.pipe(Flag.optional)`, `.pipe(Flag.withDefault(...))`, and `.pipe(Argument.variadic())`.

---

### AI (`effect/unstable/ai`)

A provider-agnostic AI/LLM integration framework with type-safe tool calling, structured output, streaming, and conversation management.

**Key exports:** `LanguageModel`, `Chat`, `Tool`, `Toolkit`, `Model`, `Prompt`, `Response`, `AiError`, `Tokenizer`, `IdGenerator`, `Telemetry`, `McpServer`, `McpSchema`, `AnthropicStructuredOutput`, `OpenAiStructuredOutput`

#### LanguageModel service

The `LanguageModel` service is the primary interface for AI text generation. Provider-specific packages implement this service.

```ts
import { Effect, Schema } from "effect"
import { LanguageModel } from "effect/unstable/ai"

// Text generation
const textProgram = Effect.gen(function*() {
  const response = yield* LanguageModel.generateText({
    prompt: "Explain quantum computing in simple terms"
  })
  console.log(response.text)
})

// Structured output
const ContactSchema = Schema.Struct({
  name: Schema.String,
  email: Schema.String
})

const structuredProgram = Effect.gen(function*() {
  const response = yield* LanguageModel.generateObject({
    prompt: "Extract contact: John Doe, john@example.com",
    schema: ContactSchema
  })
  console.log(response.value) // { name: "John Doe", email: "john@example.com" }
})
```

#### Tools and Toolkits

Define tools that language models can invoke, with schema-validated parameters:

```ts
import { Effect, Schema } from "effect"
import { LanguageModel, Tool, Toolkit } from "effect/unstable/ai"

const GetWeather = Tool.make("GetWeather", {
  description: "Get weather for a location",
  parameters: Schema.Struct({ location: Schema.String }),
  success: Schema.Struct({ temperature: Schema.Number, condition: Schema.String })
})

const GetCurrentTime = Tool.make("GetCurrentTime", {
  description: "Get the current timestamp",
  success: Schema.Number
})

// Bundle tools into a toolkit and provide handler implementations
const MyToolkit = Toolkit.make(GetWeather, GetCurrentTime)

const MyToolkitLayer = MyToolkit.toLayer({
  GetWeather: ({ location }) =>
    Effect.succeed({ temperature: 72, condition: "sunny" }),
  GetCurrentTime: () =>
    Effect.succeed(Date.now())
})
```

#### Chat sessions

The `Chat` module provides stateful conversation management with history tracking:

```ts
import { Effect, Stream } from "effect"
import { Chat } from "effect/unstable/ai"

const program = Effect.gen(function*() {
  const chat = yield* Chat.empty

  const response = yield* chat.generateText({
    prompt: "Hello! What can you help me with?"
  })
  // response.text extracts concatenated text; response.content is Array<Response.Part>
  console.log(response.text)

  // Streaming -- each emitted item is a Response.StreamPart object (not a plain string).
  // Check part.type to distinguish TextDeltaPart, FinishPart, ToolCallPart, etc.
  yield* chat.streamText({
    prompt: "Tell me a story"
  }).pipe(
    Stream.runForEach((part) =>
      Effect.sync(() => {
        if (part.type === "text-delta") process.stdout.write(part.delta)
      })
    )
  )
})
```

#### Error handling

`AiError` provides semantic error categories (RateLimitError, AuthenticationError, ContentPolicyError, etc.) with `isRetryable` and optional `retryAfter` hints for automatic retry policies.

---

### Cluster (`effect/unstable/cluster`)

A distributed systems framework for building sharded, entity-based architectures. Entities are addressable by ID and communicate via an RPC-based protocol. The cluster manages shard assignment, message routing, and entity lifecycle across multiple runners (nodes).

**Key exports:** `Entity`, `EntityType`, `EntityId`, `EntityAddress`, `EntityProxy`, `EntityProxyServer`, `EntityResource`, `Message`, `Reply`, `Sharding`, `ShardingConfig`, `ShardId`, `Runner`, `Runners`, `RunnerServer`, `RunnerStorage`, `RunnerAddress`, `RunnerHealth`, `Singleton`, `SingleRunner`, `Snowflake`, `Envelope`, `ClusterCron`, `ClusterError`, `ClusterMetrics`, `ClusterSchema`, `ClusterWorkflowEngine`, `MessageStorage`, `HttpRunner`, `SocketRunner`, `K8sHttpClient`, `SqlMessageStorage`, `SqlRunnerStorage`, `TestRunner`

#### Defining entities

```ts
import { Schema } from "effect"
import { Entity, Message } from "effect/unstable/cluster"
import { Rpc, RpcGroup } from "effect/unstable/rpc"

// Define the entity's messaging protocol using RPC
const Increment = Rpc.make("Increment")
  .setPayload(Schema.Struct({ amount: Schema.Number }))
  .setSuccess(Schema.Number)

const GetCount = Rpc.make("GetCount")
  .setSuccess(Schema.Number)

// RpcGroup.make takes spread Rpc args (no name argument)
const CounterProtocol = RpcGroup.make(Increment, GetCount)

// Entity.make takes (type: string, protocol: ReadonlyArray<Rpc>)
// To pass an RpcGroup directly, use Entity.fromRpcGroup instead
const Counter = Entity.make("Counter", [Increment, GetCount])
// or: const Counter = Entity.fromRpcGroup("Counter", CounterProtocol)
```

#### Implementing entity behavior

```ts
import { Effect, Ref } from "effect"

// Register the entity with the sharding system
const counterLayer = Counter.toLayer(
  Effect.gen(function*() {
    const count = yield* Ref.make(0)
    return {
      Increment: ({ amount }) => Ref.updateAndGet(count, (n) => n + amount),
      GetCount: () => Ref.get(count)
    }
  })
)
```

#### Using entity clients

```ts
import { Effect } from "effect"

const program = Effect.gen(function*() {
  const counterClient = yield* Counter.client
  const counter = counterClient("counter-1")

  yield* counter.Increment({ amount: 5 })
  const count = yield* counter.GetCount()
  console.log(count) // 5
})
```

The cluster module also provides `Singleton` for cluster-wide singleton services, `Snowflake` for distributed ID generation, and `ClusterCron` for distributed cron scheduling.

---

### Workflow (`effect/unstable/workflow`)

Durable workflow execution with activities, compensation, and exactly-once semantics. Workflows survive process restarts by persisting their state through a `WorkflowEngine`.

**Key exports:** `Workflow`, `Activity`, `DurableClock`, `DurableDeferred`, `WorkflowEngine`, `WorkflowProxy`, `WorkflowProxyServer`

#### Defining workflows

```ts
import { Effect, Schema } from "effect"
import { Activity, Workflow } from "effect/unstable/workflow"

// Define a workflow with payload and result schemas.
// Workflow.make takes a single options object (not positional name + options).
// `payload` holds the payload schema; `success` and `error` hold result schemas.
// `idempotencyKey` is required: a function that derives a stable string from
// the payload used to deduplicate re-triggered workflow executions.
const OrderWorkflow = Workflow.make({
  name: "OrderWorkflow",
  payload: Schema.Struct({ orderId: Schema.String, amount: Schema.Number }),
  success: Schema.Struct({ status: Schema.String }),
  error: Schema.String,
  idempotencyKey: (payload) => payload.orderId
})

// Define activities (durable, retriable steps)
const chargePayment = Activity.make({
  name: "chargePayment",
  success: Schema.Struct({ transactionId: Schema.String }),
  error: Schema.String,
  execute: Effect.gen(function*() {
    // This step is persisted -- if the workflow crashes, it won't re-execute
    return { transactionId: "txn_123" }
  })
})
```

#### Implementing and executing workflows

```ts
import { Effect } from "effect"

// Register the workflow implementation
const workflowLayer = OrderWorkflow.toLayer((payload, executionId) =>
  Effect.gen(function*() {
    const payment = yield* chargePayment
    // ... more activities
    return { status: "completed" }
  })
)

// Execute a workflow
const program = Effect.gen(function*() {
  const result = yield* OrderWorkflow.execute({ orderId: "order-1", amount: 99.99 })
  console.log(result.status) // "completed"
})
```

Workflows integrate with the cluster module via `ClusterWorkflowEngine` for distributed execution.

---

### Other Unstable Modules

| Module | Import Path | Description | Key Exports |
|---|---|---|---|
| **Socket** | `effect/unstable/socket` | WebSocket and TCP socket abstraction | `Socket`, `SocketServer` |
| **Schema** | `effect/unstable/schema` | Extended schema utilities including Model (CRUD schema sets) and VariantSchema | `Model`, `VariantSchema` |
| **Persistence** | `effect/unstable/persistence` | State persistence with key-value stores, persisted caches, queues, and rate limiters | `KeyValueStore`, `Persistence`, `Persistable`, `PersistedCache`, `PersistedQueue`, `RateLimiter`, `Redis` |
| **Reactivity** | `effect/unstable/reactivity` | Reactive primitives for UI state management (Atom/Signal pattern) | `Atom`, `AtomRef`, `AtomRegistry`, `AtomHttpApi`, `AtomRpc`, `AsyncResult`, `Hydration`, `Reactivity` |
| **Workers** | `effect/unstable/workers` | Worker thread abstraction for offloading computation | `Worker`, `WorkerRunner`, `WorkerError`, `Transferable` |
| **Encoding** | `effect/unstable/encoding` | Encoding formats for streaming data | `Msgpack`, `Ndjson`, `Sse` (Server-Sent Events) |
| **Observability** | `effect/unstable/observability` | Built-in OpenTelemetry export (no external SDK needed) and Prometheus metrics | `Otlp`, `OtlpExporter`, `OtlpTracer`, `OtlpMetrics`, `OtlpLogger`, `OtlpResource`, `OtlpSerialization`, `PrometheusMetrics` |
| **Process** | `effect/unstable/process` | Child process management with tagged template commands and pipe support | `ChildProcess`, `ChildProcessSpawner` |
| **DevTools** | `effect/unstable/devtools` | Development tools for inspecting Effect applications | `DevTools`, `DevToolsClient`, `DevToolsServer`, `DevToolsSchema` |
| **Event Log** | `effect/unstable/eventlog` | Event sourcing with journals, encryption, and remote sync | `Event`, `EventGroup`, `EventJournal`, `EventLog`, `EventLogEncryption`, `EventLogRemote`, `EventLogServer`, `SqlEventLogJournal`, `SqlEventLogServer` |

#### Process example

```ts
import { Effect, Stream } from "effect"
import { ChildProcess } from "effect/unstable/process"

// Tagged template command builder
const command = ChildProcess.make`echo "hello world"`

const program = Effect.gen(function*() {
  const handle = yield* command
  const chunks = yield* Stream.runCollect(handle.stdout)
  const exitCode = yield* handle.exitCode
  return { chunks, exitCode }
}).pipe(Effect.scoped)

// Piping commands
const pipeline = ChildProcess.make`cat package.json`.pipe(
  ChildProcess.pipeTo(ChildProcess.make`grep name`)
)
```

#### Observability example (Prometheus)

```ts
import { Effect, Metric } from "effect"
import { PrometheusMetrics } from "effect/unstable/observability"

const program = Effect.gen(function*() {
  const counter = Metric.counter("http_requests_total", {
    description: "Total HTTP requests"
  })
  yield* Metric.update(counter, 42)

  const output = yield* PrometheusMetrics.format()
  // # HELP http_requests_total Total HTTP requests
  // # TYPE http_requests_total counter
  // http_requests_total 42
})
```

---

## Package Dependency Structure

```
                    effect (core)
                   /      |      \
    @effect/platform-*   @effect/vitest   @effect/opentelemetry
    (node/bun/browser)
```

- All packages depend on the core `effect` package
- Platform packages are interchangeable -- your application code depends on abstract services from `effect`, and you swap platform implementations at the edge
- `@effect/vitest` depends on `effect` and `vitest`
- `@effect/opentelemetry` depends on `effect` and OpenTelemetry SDK packages

---

## Import Conventions

```ts
// Core -- always import from "effect"
import { Effect, Stream, Schema, Layer, ServiceMap } from "effect"

// Testing -- import from "effect/testing" subpath
// Available exports: TestClock, TestConsole, TestSchema, FastCheck
import { TestClock, TestConsole, FastCheck } from "effect/testing"
// Note: "effect/testing/FastCheck" does NOT exist as a package subpath

// Platform -- import from the specific platform package
import { NodeRuntime } from "@effect/platform-node"
import { NodeFileSystem } from "@effect/platform-node"

// Vitest -- import from "@effect/vitest"
import { it, layer, expect } from "@effect/vitest"

// OpenTelemetry -- import from "@effect/opentelemetry"
import { NodeSdk } from "@effect/opentelemetry"
```

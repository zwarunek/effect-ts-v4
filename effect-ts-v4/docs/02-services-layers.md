# Services, Layers, and Configuration (Effect 4.0)

This document covers the dependency injection system in Effect 4.0: defining services with `ServiceMap`, wiring them with `Layer`, reading configuration with `Config`/`ConfigProvider`, and managing runtime with `ManagedRuntime`.

> **Migration note (v3 to v4):** `Context.Tag`, `Context.GenericTag`, `Effect.Tag`, and `Effect.Service` are all replaced by `ServiceMap.Service`. The `Context` data structure is replaced by `ServiceMap`. `FiberRef` is replaced by `ServiceMap.Reference`. See inline migration notes throughout this document.

---

## ServiceMap -- Service Definitions

**Module:** `effect/ServiceMap`

The `ServiceMap` module provides a typed map from service identifiers to their implementations. It is the foundation for dependency injection in Effect 4.0.

### Defining a Service

Use `ServiceMap.Service` to create a service identifier. There are two forms:

**Function syntax** -- for simple, standalone services:

```ts
import { ServiceMap } from "effect"

const Database = ServiceMap.Service<{
  query: (sql: string) => string
}>("Database")
```

**Class syntax** -- for services that carry identity and can attach static members:

```ts
import { ServiceMap } from "effect"

class Database extends ServiceMap.Service<Database, {
  readonly query: (sql: string) => string
}>()("Database") {}
```

> **v3 migration:** `Context.GenericTag<T>(id)` becomes `ServiceMap.Service<T>(id)`. The class form `Context.Tag(id)<Self, Shape>()` becomes `ServiceMap.Service<Self, Shape>()(id)` -- note that type parameters come first and the identifier string is passed to the returned constructor.

#### Type signature

```ts
export const Service: {
  // Function form: ServiceMap.Service<Shape>(key)
  <Identifier, Shape = Identifier>(key: string): Service<Identifier, Shape>

  // Class form with explicit Shape: ServiceMap.Service<Self, Shape>()(key, options?)
  <Self, Shape>(): <
    const Identifier extends string,
    E,
    R = Types.unassigned,
    Args extends ReadonlyArray<any> = never
  >(
    id: Identifier,
    options?: {
      readonly make: ((...args: Args) => Effect<Shape, E, R>) | Effect<Shape, E, R> | undefined
    } | undefined
  ) => ServiceClass<Self, Identifier, Shape>

  // Class form with inferred Shape from make: ServiceMap.Service<Self>()(key, { make })
  <Self>(): <
    const Identifier extends string,
    Make extends Effect<any, any, any> | ((...args: any) => Effect<any, any, any>)
  >(
    id: Identifier,
    options: { readonly make: Make }
  ) => ServiceClass<Self, Identifier, /* Shape inferred from Make */>
    & { readonly make: Make }
}
```

### The `Service` Interface

Every service identifier provides these members:

```ts
export interface Service<in out Identifier, in out Shape> {
  readonly key: string
  readonly Service: Shape    // phantom field for type extraction
  readonly Identifier: Identifier  // phantom field for type extraction
  of(self: Shape): Shape
  serviceMap(self: Shape): ServiceMap<Identifier>
  use<A, E, R>(f: (service: Shape) => Effect<A, E, R>): Effect<A, E, R | Identifier>
  useSync<A>(f: (service: Shape) => A): Effect<A, never, Identifier>
}
```

- **`key`** -- the string identifier used internally to store/retrieve the service.
- **`Service` / `Identifier`** -- phantom fields used for type-level extraction only (not useful at runtime).
- **`of`** -- identity function, validates shape at the type level.
- **`serviceMap`** -- wraps a value into a single-entry `ServiceMap`.
- **`use`** -- access the service inside a callback, returning an Effect that requires the service.
- **`useSync`** -- like `use` but for pure (non-effectful) callbacks.

#### Yielding a service in `Effect.gen`

Service identifiers are `Yieldable`, so you can yield them directly:

```ts
import { Effect, ServiceMap } from "effect"

class Notifications extends ServiceMap.Service<Notifications, {
  readonly notify: (message: string) => Effect.Effect<void>
}>()("Notifications") {}

const program = Effect.gen(function*() {
  const notifications = yield* Notifications
  yield* notifications.notify("hello")
})
```

> **v3 migration:** `Effect.Tag` provided proxy access to service methods as static properties (accessors). In v4, accessors are removed. Use `yield*` in generators or `Service.use()` for one-liners. Prefer `yield*` because it makes dependencies explicit.

### Service with `make`

Attach an effectful constructor to the class:

```ts
class Logger extends ServiceMap.Service<Logger>()("Logger", {
  make: Effect.gen(function*() {
    const config = yield* Config
    return { log: (msg: string) => Effect.log(`[${config.prefix}] ${msg}`) }
  })
}) {
  static layer = Layer.effect(this, this.make).pipe(
    Layer.provide(Config.layer)
  )
}
```

> **v3 migration:** `Effect.Service` auto-generated a `.Default` layer with `dependencies` wired in. In v4, define layers explicitly with `Layer.effect`. The `dependencies` option no longer exists -- use `Layer.provide`. Convention: name layers `.layer` instead of `.Default`.

### Type Utilities

```ts
// Extract the shape (implementation type) from a service
type Shape = ServiceMap.Service.Shape<typeof Database>

// Extract the identifier type
type Id = ServiceMap.Service.Identifier<typeof Database>

// Constraint type for any service
type AnyService = ServiceMap.Service.Any
```

### ServiceMap Data Structure

A `ServiceMap<Services>` is an immutable typed map. Key functions:

```ts
// Create a map with one entry
ServiceMap.make(Database, { query: (sql) => `Result: ${sql}` })

// Create an empty map
ServiceMap.empty()

// Create from a raw JS Map (unsafe -- no type checking)
ServiceMap.makeUnsafe(new Map([["MyKey", impl]]))

// Add a service to an existing map (also supports data-last: ServiceMap.add(key, value)(map))
ServiceMap.add(map, Timeout, { TIMEOUT: 5000 })

// Add or remove a service based on an Option value
ServiceMap.addOrOmit(map, key, Option.some(impl))  // adds
ServiceMap.addOrOmit(map, key, Option.none())       // removes

// Get a service (type-safe at compile time; throws at runtime if not present and not a Reference)
// Note: ServiceMap.get is an alias for ServiceMap.getUnsafe
ServiceMap.get(map, Database)

// Get a service without type constraint -- throws if not found and not a Reference
ServiceMap.getUnsafe(map, Database)

// Get with a fallback function (never throws)
ServiceMap.getOrElse(map, Database, () => fallbackImpl)

// Get as Option (None if not present and not a Reference)
ServiceMap.getOption(map, Database)

// Get or undefined (no fallback, no throw)
ServiceMap.getOrUndefined(map, Database)

// Merge two maps (second map's values win on key conflict)
ServiceMap.merge(map1, map2)

// Merge many maps
ServiceMap.mergeAll(map1, map2, map3)

// Pick/omit specific services (data-last, variadic)
ServiceMap.pick(Port)(map)
ServiceMap.omit(Timeout)(map)

// Type guards
ServiceMap.isServiceMap(value)  // boolean
ServiceMap.isService(value)     // boolean
ServiceMap.isReference(value)   // boolean
```

### References (Services with Default Values)

A `Reference` is a service that has a default value, so it works without explicit provision.

```ts
import { ServiceMap } from "effect"

const LogLevel = ServiceMap.Reference<"info" | "warn" | "error">("LogLevel", {
  defaultValue: () => "info"
})

// Works without provision -- returns "info"
const services = ServiceMap.empty()
const level = ServiceMap.get(services, LogLevel) // "info"

// Override with a custom value
const custom = ServiceMap.make(LogLevel, "error")
ServiceMap.get(custom, LogLevel) // "error"
```

```ts
export interface Reference<in out Shape> extends Service<never, Shape> {
  readonly defaultValue: () => Shape
  [Symbol.iterator](): EffectIterator<Reference<Shape>>
}
```

> **Note:** The default value is **cached** after first computation -- `defaultValue()` is only called once. References are `Yieldable`, so `yield* myReference` works in `Effect.gen`.

> **v3 migration:** `Context.Reference<Self>()(id, { defaultValue })` becomes `ServiceMap.Reference<Shape>(id, { defaultValue })`. `FiberRef` values are now `ServiceMap.Reference` values -- see the References section below.

---

## Layer -- Dependency Injection

**Module:** `effect/Layer`

A `Layer<ROut, E, RIn>` describes how to build services:
- **`ROut`** -- the services this layer provides
- **`E`** -- errors during construction
- **`RIn`** -- services this layer requires as dependencies

Layers are **memoized by default**: if the same layer instance appears multiple times in a dependency graph, it is built only once.

### Creating Layers

#### From a constant value

```ts
const databaseLayer = Layer.succeed(Database)({
  query: (sql: string) => Effect.succeed(`Result: ${sql}`)
})
```

#### Lazily (synchronous)

```ts
const databaseLayer = Layer.sync(Database)(() => ({
  query: (sql: string) => Effect.succeed(`Result: ${sql}`)
}))
```

#### From an Effect (replaces `Layer.scoped` from v3)

```ts
const databaseLayer = Layer.effect(Database)(
  Effect.gen(function*() {
    const pool = yield* connectToDatabase()
    yield* Effect.addFinalizer(() => pool.close())
    return { query: (sql) => pool.query(sql) }
  })
)
```

> **v3 migration:** `Layer.scoped` is replaced by `Layer.effect`. The effect runs in the layer's scope, so `addFinalizer` works. `Layer.scopedDiscard` is replaced by `Layer.effectDiscard`.

#### Type signatures

```ts
export const succeed: {
  <I, S>(service: ServiceMap.Service<I, S>): (resource: S) => Layer<I>
  <I, S>(service: ServiceMap.Service<I, S>, resource: NoInfer<S>): Layer<I>
}

export const effect: {
  // Curried (data-last):
  <I, S>(service: ServiceMap.Service<I, S>): <E, R>(
    effect: Effect<S, E, R>
  ) => Layer<I, E, Exclude<R, Scope>>
  // Data-first (two-argument form):
  <I, S, E, R>(
    service: ServiceMap.Service<I, S>,
    effect: Effect<S, E, R>
  ): Layer<I, E, Exclude<R, Scope>>
}

export const sync: {
  // Curried (data-last):
  <I, S>(service: ServiceMap.Service<I, S>): (evaluate: LazyArg<S>) => Layer<I>
  // Data-first (two-argument form):
  <I, S>(service: ServiceMap.Service<I, S>, evaluate: LazyArg<S>): Layer<I>
}
```

### Composing Layers

#### `Layer.mergeAll` -- combine independent layers

```ts
const appLayer = Layer.mergeAll(databaseLayer, loggerLayer, cacheLayer)
```

All layers are built concurrently.

#### `Layer.provide` -- feed outputs into inputs

```ts
const userServiceLayer = Layer.effect(UserService)(Effect.gen(function*() {
  const db = yield* Database
  const logger = yield* Logger
  return {
    getUser: (id: string) => Effect.gen(function*() {
      yield* logger.log(`Looking up user ${id}`)
      return yield* db.query(`SELECT * FROM users WHERE id = ${id}`)
    })
  }
}))

// Wire dependencies
const fullLayer = userServiceLayer.pipe(
  Layer.provide(Layer.mergeAll(databaseLayer, loggerLayer))
)
// fullLayer: Layer<UserService, never, never>
```

#### `Layer.provideMerge` -- provide and keep dependency outputs

Like `Layer.provide`, but the resulting layer outputs both the consumer's services and the dependency's services.

```ts
const allServicesLayer = userServiceLayer.pipe(
  Layer.provideMerge(Layer.mergeAll(databaseLayer, loggerLayer))
)
// Provides: UserService | Database | Logger
```

### Layer Memoization

In v4, layers are globally memoized across `Effect.provide` calls (the `MemoMap` is shared on the fiber). This means:

```ts
const main = program.pipe(
  Effect.provide(MyServiceLayer),
  Effect.provide(MyServiceLayer) // same layer, built only ONCE
)
```

> **v3 migration:** In v3, each `Effect.provide` had its own memoization scope, so the same layer provided twice would be built twice. In v4, layers are shared automatically.

**Opting out:**

```ts
// Layer.fresh -- bypass the shared cache for a specific layer
Effect.provide(Layer.fresh(MyServiceLayer))

// { local: true } -- build with a local memo map
Effect.provide(MyServiceLayer, { local: true })
```

### Testing with `Layer.mock`

Create partial mocks where unimplemented methods throw:

```ts
const testLayer = Layer.mock(UserService)({
  getUser: (id: string) => Effect.succeed({ id, name: "Test User" })
  // deleteUser, updateUser die with an "UnimplementedError" defect if called
  // (i.e. Effect.die, not a typed failure -- the fiber dies)
})
```

```ts
export const mock: <I, S extends object>(
  service: ServiceMap.Service<I, S>
) => (implementation: PartialEffectful<S>) => Layer<I>
```

### Other Layer Operations

```ts
// Error handling
Layer.orDie(layer)                    // convert errors to defects
Layer.catch(layer, onError)           // recover from all errors
Layer.catchTag("ConfigError", handler)(layer)  // recover from tagged errors
Layer.catchCause(layer, onCause)      // recover from causes

// Dynamic layers
Layer.flatMap(layer, (services) => newLayer)  // build layer from output
Layer.unwrap(effectProducingLayer)            // unwrap Effect<Layer>

// Merging (binary)
Layer.merge(layer1, layer2)           // merge exactly two layers concurrently

// Multi-service constructors
Layer.succeedServices(serviceMap)     // layer from a pre-built ServiceMap
Layer.effectServices(effectOfServiceMap)  // layer from Effect<ServiceMap, E, R>
Layer.effectDiscard(effect)           // layer that runs an Effect for side-effects only (no services provided)

// Mutation helpers
Layer.updateService(service, f)(layer)  // update a service already in the output

// Lifecycle
Layer.launch(layer)  // build and run forever (for server apps)
Layer.fresh(layer)   // always build fresh (no memoization)

// Tracing
Layer.withSpan("name", options)(layer)   // wrap layer construction in a span (data-last)
Layer.withSpan(layer, "name", options)   // same, data-first form
Layer.span("name", options)              // standalone span layer to use as a dependency
Layer.parentSpan(span)                   // set an existing span as the parent
Layer.withParentSpan(layer, span)        // attach a layer to an existing span
```

### Providing Layers to Effects

```ts
const program = Effect.gen(function*() {
  const db = yield* Database
  return yield* db.query("SELECT 1")
}).pipe(
  Effect.provide(databaseLayer)
)
```

---

## Config -- Configuration Values

**Module:** `effect/Config`

A `Config<T>` is a recipe for extracting a typed value from a `ConfigProvider`. Configs are `Yieldable` -- you can yield them in `Effect.gen` and they automatically resolve using the current `ConfigProvider`.

### Convenience Constructors

```ts
Config.string("HOST")           // Config<string>
Config.nonEmptyString("HOST")   // Config<string> (fails if empty)
Config.number("PORT")           // Config<number> (allows NaN, Infinity)
Config.finite("PORT")           // Config<number> (rejects NaN and Infinity)
Config.boolean("DEBUG")         // Config<boolean> -- accepts "true","yes","1","on","y","false","no","0","off","n"
Config.int("WORKERS")           // Config<number> (integers only)
Config.port("PORT")             // Config<number> (1-65535)
Config.url("API_URL")           // Config<URL>
Config.duration("TIMEOUT")      // Config<Duration>
Config.logLevel("LOG_LEVEL")    // Config<LogLevel>
Config.redacted("API_KEY")      // Config<Redacted<string>>
Config.date("CREATED_AT")       // Config<Date>
Config.literal("production", "ENV")  // Config<"production"> (literal value)
Config.succeed(42)              // Config<number> -- always succeeds with a constant
Config.fail(error)              // Config<never> -- always fails with a ConfigError

// Schema utilities for use with Config.schema:
Config.Boolean   // Schema.Codec for boolean strings ("true"/"false"/"yes"/"no"/"on"/"off"/"1"/"0"/"y"/"n")
Config.Duration  // Schema.Codec for Duration strings ("10 seconds", "500 millis")
Config.Port      // Schema.Codec for port numbers (1-65535)
Config.LogLevel  // Schema.Codec for log level strings ("Info", "Debug", etc.)
Config.Record(Schema.String, Schema.String)  // Schema for key-value records (also parses "key=val,key2=val2" strings)
```

### Schema-based Config

For structured configuration, use `Config.schema` with any `Schema.Codec`:

```ts
import { Config, ConfigProvider, Effect, Schema } from "effect"

const AppConfig = Config.schema(
  Schema.Struct({
    host: Schema.String,
    port: Schema.Int
  }),
  "app"   // root path prefix
)

const provider = ConfigProvider.fromEnv({
  env: { app_host: "localhost", app_port: "8080" }
})

// Effect.runSync(AppConfig.parse(provider))
// => { host: "localhost", port: 8080 }
```

### Combinators

```ts
// Default value (only for missing data, not validation errors)
Config.number("port").pipe(Config.withDefault(() => 3000))

// Optional (returns Option; None only on missing data, not validation errors)
Config.option(Config.number("port"))

// Transform (pure function -- cannot fail)
Config.string("name").pipe(Config.map((s) => s.toUpperCase()))

// Transform (effectful function that may produce a ConfigError)
Config.string("name").pipe(
  Config.mapOrFail((s) => s.length > 0 ? Effect.succeed(s.trim()) : Effect.fail(new ConfigError(...)))
)

// Fallback on any ConfigError (including validation errors)
Config.string("HOST").pipe(
  Config.orElse(() => Config.succeed("localhost"))
)

// Combine multiple configs into a struct or tuple
Config.all({
  host: Config.string("host"),
  port: Config.number("port")
})

// Unwrap a Config.Wrap<T> -- accepts either a Config<T> or a record of Configs
Config.unwrap(wrappedConfig)
```

> **Note on `Config.orElse` vs `Config.withDefault`:** `withDefault` only fires on *missing data* errors. `orElse` fires on *any* `ConfigError` including type-validation failures.

### The `Config` Interface

```ts
export interface Config<out T> {
  readonly parse: (provider: ConfigProvider) => Effect<T, ConfigError>
}
```

Every `Config` is `Yieldable`, so in `Effect.gen`:

```ts
const program = Effect.gen(function*() {
  const port = yield* Config.number("PORT")
  console.log(port)
})
```

This automatically uses the current `ConfigProvider` from the service map.

---

## ConfigProvider -- Configuration Sources

**Module:** `effect/ConfigProvider`

A `ConfigProvider` loads raw configuration nodes from a backing store. It is registered as a `ServiceMap.Reference` with a default of `fromEnv()`.

### Built-in Providers

```ts
// Environment variables (default)
ConfigProvider.fromEnv()
ConfigProvider.fromEnv({ env: { HOST: "localhost", PORT: "3000" } })

// Plain JavaScript object
ConfigProvider.fromUnknown({ database: { host: "localhost", port: 5432 } })

// .env file contents (string)
ConfigProvider.fromDotEnvContents("HOST=localhost\nPORT=3000")

// .env file from disk (requires FileSystem)
ConfigProvider.fromDotEnv({ path: ".env" })

// Directory tree (e.g., Kubernetes ConfigMap mounts)
ConfigProvider.fromDir({ rootPath: "/etc/myapp" })

// Custom provider
ConfigProvider.make((path) =>
  Effect.succeed(path.join(".") === "host" ? ConfigProvider.makeValue("localhost") : undefined)
)
```

### Combinators

```ts
// Fallback: try self first; only consults fallback when self returns undefined (path not found).
// SourceError from self is NOT caught -- it propagates immediately.
ConfigProvider.orElse(primaryProvider, fallbackProvider)

// Scope under a prefix (accepts string or Path array)
ConfigProvider.nested(provider, "database")
ConfigProvider.nested(provider, ["database", "primary"])

// Convert camelCase to SCREAMING_SNAKE_CASE (e.g. "databaseHost" -> "DATABASE_HOST")
provider.pipe(ConfigProvider.constantCase)

// Arbitrary path transformation (e.g. uppercase all string segments)
ConfigProvider.mapInput(provider, (path) => path.map(seg =>
  typeof seg === "string" ? seg.toUpperCase() : seg
))
```

### Installing as a Layer

```ts
// Replace the current provider entirely
const TestConfigLayer = ConfigProvider.layer(
  ConfigProvider.fromUnknown({ port: 8080 })
)

// Add as fallback (existing provider is tried first)
const DefaultsLayer = ConfigProvider.layerAdd(
  ConfigProvider.fromUnknown({ HOST: "localhost", PORT: "3000" })
)

// Add as primary (new provider is tried first)
ConfigProvider.layerAdd(overrideProvider, { asPrimary: true })
```

---

## ManagedRuntime -- Runtime Management

**Module:** `effect/ManagedRuntime`

A `ManagedRuntime` converts a `Layer` into a runtime that can execute Effects with pre-built services. Useful at application boundaries (e.g., framework integrations, tests).

```ts
import { Console, Effect, Layer, ManagedRuntime, ServiceMap } from "effect"

class Notifications extends ServiceMap.Service<Notifications, {
  readonly notify: (message: string) => Effect.Effect<void>
}>()("Notifications") {
  static layer = Layer.succeed(this)({
    notify: (message) => Console.log(message)
  })
}

async function main() {
  const runtime = ManagedRuntime.make(Notifications.layer)

  await runtime.runPromise(
    Effect.gen(function*() {
      const n = yield* Notifications
      yield* n.notify("Hello, world!")
    })
  )

  await runtime.dispose()
}
```

### Interface

```ts
export interface ManagedRuntime<in R, out ER> {
  readonly memoMap: Layer.MemoMap
  /** The effect that builds the services. Awaitable. */
  readonly servicesEffect: Effect<ServiceMap<R>, ER>
  /** Returns a Promise that resolves with the built ServiceMap. */
  readonly services: () => Promise<ServiceMap<R>>
  readonly runFork: <A, E>(self: Effect<A, E, R>, options?: RunOptions) => Fiber<A, E | ER>
  readonly runSync: <A, E>(effect: Effect<A, E, R>) => A
  readonly runSyncExit: <A, E>(effect: Effect<A, E, R>) => Exit<A, ER | E>
  readonly runPromise: <A, E>(effect: Effect<A, E, R>, options?: RunOptions) => Promise<A>
  readonly runPromiseExit: <A, E>(effect: Effect<A, E, R>, options?: RunOptions) => Promise<Exit<A, ER | E>>
  readonly runCallback: <A, E>(
    effect: Effect<A, E, R>,
    options?: RunOptions & { readonly onExit: (exit: Exit<A, E | ER>) => void }
  ) => (interruptor?: number) => void
  readonly dispose: () => Promise<void>
  readonly disposeEffect: Effect<void>
}
```

The runtime lazily builds the layer on first use and caches the resulting services.

---

## Resource -- Refreshable Values

**Module:** `effect/Resource`

A `Resource<A, E>` is a value loaded into memory that can be refreshed manually or on a schedule.

```ts
import { Effect, Resource, Schedule, ServiceMap } from "effect"

// Resource.manual and Resource.auto return an Effect that requires a Scope.
// They must be used inside Effect.scoped or provided to a Layer.

// Manual refresh -- must be used within a scoped effect
const program = Effect.scoped(
  Effect.gen(function*() {
    const resource = yield* Resource.manual(
      Effect.sync(() => fetchLatestConfig())
    )

    // Read current value
    const value = yield* Resource.get(resource)

    // Trigger a manual refresh
    yield* Resource.refresh(resource)

    return value
  })
)

// Auto-refresh on a schedule
const autoProgram = Effect.scoped(
  Effect.gen(function*() {
    const resource = yield* Resource.auto(
      Effect.sync(() => fetchLatestConfig()),
      Schedule.fixed("5 minutes")
    )
    return yield* Resource.get(resource)
  })
)
```

> **Important:** `Resource.manual` and `Resource.auto` return `Effect<Resource<A, E>, never, Scope.Scope | R>`, not a `Resource` directly. They must be called inside a scoped Effect so the resource's internal finalizers are properly managed.

```ts
// Type signatures
export const manual: <A, E, R>(
  acquire: Effect<A, E, R>
) => Effect<Resource<A, E>, never, Scope.Scope | R>

export const auto: <A, E, R, Out, E2, R2>(
  acquire: Effect<A, E, R>,
  policy: Schedule<Out, unknown, E2, R2>
) => Effect<Resource<A, E>, never, R | R2 | Scope.Scope>

export const get: <A, E>(self: Resource<A, E>) => Effect<A, E>
export const refresh: <A, E>(self: Resource<A, E>) => Effect<void, E>

export interface Resource<in out A, in out E = never> {
  readonly scopedRef: ScopedRef<Exit<A, E>>
  readonly acquire: Effect<A, E>
}
```

---

## References -- Built-in Runtime Configuration

**Module:** `effect/References`

Built-in `ServiceMap.Reference` values that control runtime behavior. Each has a sensible default and can be overridden per-scope with `Effect.provideService`.

> **v3 migration:** These replace `FiberRef` values. Read with `yield*` instead of `FiberRef.get`. Set with `Effect.provideService` instead of `Effect.locally` or `FiberRef.set`.

| Reference                            | Type                        | Default          | Description                              |
|--------------------------------------|-----------------------------|------------------|------------------------------------------|
| `References.CurrentConcurrency`      | `"unbounded" \| number`     | `"unbounded"`    | Concurrency limit                        |
| `References.CurrentLogLevel`         | `LogLevel`                  | `"Info"`         | Current log level                        |
| `References.MinimumLogLevel`         | `LogLevel`                  | `"Info"`         | Minimum log level threshold              |
| `References.CurrentLogAnnotations`   | `ReadonlyRecord<string, unknown>` | `{}`       | Annotations added to all log entries     |
| `References.CurrentLogSpans`         | `ReadonlyArray<[label: string, timestamp: number]>` | `[]` | Active log spans              |
| `References.Scheduler`               | `Scheduler`                 | `MixedScheduler` | Scheduler implementation                 |
| `References.MaxOpsBeforeYield`       | `number`                    | (runtime default)| Ops before cooperative yield             |
| `References.TracerEnabled`           | `boolean`                   | `true`           | Enable/disable tracing globally          |
| `References.TracerTimingEnabled`     | `boolean`                   | `true`           | Enable/disable span timing               |
| `References.TracerSpanAnnotations`   | `ReadonlyRecord<string, unknown>` | `{}`       | Auto-added span annotations              |
| `References.TracerSpanLinks`         | `ReadonlyArray<SpanLink>`   | `[]`             | Auto-added span links                    |
| `References.UnhandledLogLevel`       | `LogLevel \| undefined`     | `"Error"`        | Log level for unhandled errors           |
| `References.CurrentTraceLevel`       | (re-export from `Tracer`)   | —                | Current trace verbosity level            |
| `References.MinimumTraceLevel`       | (re-export from `Tracer`)   | —                | Minimum trace level threshold            |
| `References.DisablePropagation`      | (re-export from `Tracer`)   | —                | Disable trace context propagation        |

### Reading and Setting References

```ts
import { Effect, References } from "effect"

const program = Effect.gen(function*() {
  // Read (uses default if not overridden)
  const level = yield* References.CurrentLogLevel // "Info"

  // Override for a scoped effect
  yield* Effect.provideService(
    Effect.gen(function*() {
      const level = yield* References.CurrentLogLevel // "Debug"
      yield* Effect.log("verbose logging here")
    }),
    References.CurrentLogLevel,
    "Debug"
  )
})
```

---

## End-to-End Example

Putting it all together -- defining services, composing layers, and running a program:

```ts
import { Config, Effect, Layer, ManagedRuntime, ServiceMap } from "effect"

// 1. Define services
class Database extends ServiceMap.Service<Database, {
  readonly query: (sql: string) => Effect.Effect<string>
}>()("Database") {}

class UserRepo extends ServiceMap.Service<UserRepo, {
  readonly findUser: (id: string) => Effect.Effect<string>
}>()("UserRepo") {}

// 2. Create layers
const DatabaseLayer = Layer.effect(Database)(
  Effect.gen(function*() {
    const port = yield* Config.port("DB_PORT")
    return { query: (sql) => Effect.succeed(`[port=${port}] ${sql}`) }
  })
)

const UserRepoLayer = Layer.effect(UserRepo)(
  Effect.gen(function*() {
    const db = yield* Database
    return {
      findUser: (id) => db.query(`SELECT * FROM users WHERE id = '${id}'`)
    }
  })
)

// 3. Compose layers
const AppLayer = UserRepoLayer.pipe(
  Layer.provide(DatabaseLayer)
)

// 4. Run the program
const program = Effect.gen(function*() {
  const repo = yield* UserRepo
  return yield* repo.findUser("42")
})

const runtime = ManagedRuntime.make(AppLayer)
const result = await runtime.runPromise(program)
// => "[port=5432] SELECT * FROM users WHERE id = '42'"
await runtime.dispose()
```

---

## v3 to v4 Quick Reference

| v3                                    | v4                                         |
| ------------------------------------- | ------------------------------------------ |
| `Context.GenericTag<T>(id)`           | `ServiceMap.Service<T>(id)`                |
| `Context.Tag(id)<Self, Shape>()`      | `ServiceMap.Service<Self, Shape>()(id)`    |
| `Effect.Tag(id)<Self, Shape>()`       | `ServiceMap.Service<Self, Shape>()(id)`    |
| `Effect.Service<Self>()(id, opts)`    | `ServiceMap.Service<Self>()(id, { make })` |
| `Context.Reference<Self>()(id, opts)` | `ServiceMap.Reference<T>(id, opts)`        |
| `Context.make(tag, impl)`            | `ServiceMap.make(tag, impl)`               |
| `Context.get(ctx, tag)`              | `ServiceMap.get(map, tag)`                 |
| `Context.add(ctx, tag, impl)`        | `ServiceMap.add(map, tag, impl)`           |
| `Layer.scoped`                        | `Layer.effect`                             |
| `Layer.scopedDiscard`                 | `Layer.effectDiscard`                      |
| `FiberRef.get(ref)`                   | `yield* reference`                         |
| `Effect.locally(eff, ref, value)`     | `Effect.provideService(eff, ref, value)`   |
| `FiberRef.set(ref, value)`            | `Effect.provideService(eff, ref, value)`   |
| `Logger.Default` / `Service.Default`  | `Service.layer` (define explicitly)        |

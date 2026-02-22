# Common Patterns and Utilities

This document covers higher-level patterns and utility modules in Effect 4.0: reference-counted resources, scoped references, layer maps, managed runtimes, pattern matching, and composition best practices.

---

## RcRef -- Reference-Counted Resource

**Module:** `effect/RcRef`

An `RcRef<A, E>` wraps a resource that can be acquired and released multiple times. The resource is lazily acquired on the first call to `get` and automatically released when the last reference (tracked by `Scope`) is dropped. An optional `idleTimeToLive` keeps the resource alive for a grace period after the reference count reaches zero.

### Type Definition

```ts
interface RcRef<out A, out E = never> extends Pipeable {
  readonly [TypeId]: RcRef.Variance<A, E>
}
```

### Key Functions

```ts
// Create an RcRef with lazy acquisition
RcRef.make<A, E, R>(options: {
  readonly acquire: Effect<A, E, R>
  readonly idleTimeToLive?: Duration.Input
}): Effect<RcRef<A, E>, never, R | Scope>

// Get the resource, incrementing the reference count
RcRef.get<A, E>(self: RcRef<A, E>): Effect<A, E, Scope>

// Invalidate the current resource (force re-acquisition)
RcRef.invalidate<A, E>(self: RcRef<A, E>): Effect<void>
```

### Example

```ts
import { Effect, RcRef } from "effect"

const program = Effect.gen(function*() {
  const ref = yield* RcRef.make({
    acquire: Effect.acquireRelease(
      Effect.succeed("database connection"),
      (conn) => Effect.log(`Closing: ${conn}`)
    ),
    idleTimeToLive: "30 seconds"
  })

  // Multiple gets share the same resource
  const conn1 = yield* RcRef.get(ref)
  const conn2 = yield* RcRef.get(ref)
  console.log(conn1 === conn2) // true
}).pipe(Effect.scoped)
```

### When to Use

- **Singleton resources** shared across multiple consumers (database connections, HTTP clients)
- **Lazy initialization** where you want the resource created only when first needed
- **Idle timeout** to keep the resource alive briefly after all consumers release it, avoiding expensive re-acquisition for bursty access patterns

---

## RcMap -- Reference-Counted Resource Map

**Module:** `effect/RcMap`

An `RcMap<K, A, E>` is a keyed version of `RcRef`. Resources are lazily acquired per-key and automatically released when no references remain. Supports optional `capacity` limits and `idleTimeToLive` per-key.

### Type Definition

```ts
interface RcMap<in out K, in out A, in out E = never> extends Pipeable {
  readonly lookup: (key: K) => Effect<A, E, Scope>
  readonly idleTimeToLive: (key: K) => Duration
  readonly capacity: number
  state: State<K, A, E>
}

// State can be Open (with a MutableHashMap of entries) or Closed
type State<K, A, E> = State.Open<K, A, E> | State.Closed
```

### Key Functions

```ts
// Create an RcMap (no capacity limit)
RcMap.make<K, A, E, R>(options: {
  readonly lookup: (key: K) => Effect<A, E, R>
  readonly idleTimeToLive?: Duration.Input | ((key: K) => Duration.Input)
  readonly capacity?: undefined
}): Effect<RcMap<K, A, E>, never, Scope | R>

// Create an RcMap with a capacity limit
// When capacity is exceeded, get() fails with Cause.ExceededCapacityError
RcMap.make<K, A, E, R>(options: {
  readonly lookup: (key: K) => Effect<A, E, R>
  readonly idleTimeToLive?: Duration.Input | ((key: K) => Duration.Input)
  readonly capacity: number
}): Effect<RcMap<K, A, E | Cause.ExceededCapacityError>, never, Scope | R>

// Get a resource by key (acquired on first access, reference counted)
RcMap.get<K, A, E>(self: RcMap<K, A, E>, key: K): Effect<A, E, Scope>

// Invalidate a specific key (forces re-acquisition on next get)
RcMap.invalidate<K, A, E>(self: RcMap<K, A, E>, key: K): Effect<void>

// Check whether a key currently exists in the map
RcMap.has<K, A, E>(self: RcMap<K, A, E>, key: K): Effect<boolean>

// Returns an iterable of all keys currently held in the map
RcMap.keys<K, A, E>(self: RcMap<K, A, E>): Effect<Iterable<K>>

// Reset the idle-expiration timer for a key (extends its lifetime)
RcMap.touch<K, A, E>(self: RcMap<K, A, E>, key: K): Effect<void>
```

> **Note:** `RcMap.get`, `RcMap.invalidate`, `RcMap.has`, `RcMap.keys`, and `RcMap.touch` all support data-last (piped) form as well: e.g. `RcMap.get(key)(self)`.

> **Capacity:** When `capacity` is set, the error type of the map is widened to include `Cause.ExceededCapacityError`. When `capacity` is absent or `undefined`, the error type is exactly `E`.

### Example

```ts
import { Effect, RcMap } from "effect"

Effect.gen(function*() {
  const connections = yield* RcMap.make({
    lookup: (dbName: string) =>
      Effect.acquireRelease(
        Effect.succeed(`Connection to ${dbName}`),
        (conn) => Effect.log(`Closing ${conn}`)
      ),
    idleTimeToLive: "5 minutes",
    capacity: 10
  })

  // Multiple gets for the same key share the resource
  const conn = yield* RcMap.get(connections, "users-db")
  const conn2 = yield* RcMap.get(connections, "users-db")
  console.log(conn === conn2) // true

  // Different key gets a different resource
  const other = yield* RcMap.get(connections, "analytics-db")
}).pipe(Effect.scoped)
```

### When to Use

- **Connection pools indexed by key** (e.g., multi-tenant database connections)
- **Per-user or per-session resources** that should be lazily created and auto-cleaned
- **Caching with lifecycle management** where cached values hold resources needing cleanup

---

## ScopedRef -- Resource-Backed Mutable Reference

**Module:** `effect/ScopedRef`

A `ScopedRef<A>` is a mutable reference whose value is associated with a `Scope`. When you `set` a new value (via an effectful acquire), the old value's scope is closed (releasing its resources) before the new value takes over.

### Type Definition

```ts
interface ScopedRef<in out A> extends Pipeable {
  readonly backing: SynchronizedRef<readonly [Scope.Closeable, A]>
}
```

### Key Functions

```ts
// Create from a simple value (no resource acquisition)
ScopedRef.make<A>(evaluate: () => A): Effect<ScopedRef<A>, never, Scope>

// Create from an effectful acquisition
ScopedRef.fromAcquire<A, E, R>(
  acquire: Effect<A, E, R>
): Effect<ScopedRef<A>, E, Scope | R>

// Read the current value
ScopedRef.get<A>(self: ScopedRef<A>): Effect<A>
ScopedRef.getUnsafe<A>(self: ScopedRef<A>): A

// Replace the value -- old resources are released, new resources acquired
ScopedRef.set<A, R, E>(
  self: ScopedRef<A>,
  acquire: Effect<A, E, R>
): Effect<void, E, Exclude<R, Scope>>
```

### Example

```ts
import { Effect, ScopedRef } from "effect"

const program = Effect.gen(function*() {
  // Create a scoped ref with a connection
  const ref = yield* ScopedRef.fromAcquire(
    Effect.acquireRelease(
      Effect.succeed("connection-v1"),
      (conn) => Effect.log(`Releasing ${conn}`)
    )
  )

  const v1 = yield* ScopedRef.get(ref)
  console.log(v1) // "connection-v1"

  // Replace -- "connection-v1" resources are released
  yield* ScopedRef.set(ref,
    Effect.acquireRelease(
      Effect.succeed("connection-v2"),
      (conn) => Effect.log(`Releasing ${conn}`)
    )
  )

  const v2 = yield* ScopedRef.get(ref)
  console.log(v2) // "connection-v2"
}).pipe(Effect.scoped)
// When scope closes: "Releasing connection-v2"
```

### When to Use

- **Hot-swappable resources** (e.g., rotating database connections, refreshing credentials)
- **Configuration that requires resource cleanup** when the config value changes

---

## LayerMap -- Dynamic Layer Selection

**Module:** `effect/LayerMap`

A `LayerMap<K, I, E>` maps keys to `Layer`s, backed by an `RcMap`. Use it to dynamically select and build layers based on a runtime key (e.g., tenant ID, environment name).

### Type Definition

```ts
interface LayerMap<in out K, in out I, in out E = never> {
  readonly rcMap: RcMap<K, ServiceMap<I>, E>

  // Get a Layer for a key
  get(key: K): Layer<I, E>

  // Get the services directly (requires Scope)
  services(key: K): Effect<ServiceMap<I>, E, Scope>

  // Invalidate a cached layer
  invalidate(key: K): Effect<void>
}
```

### Key Functions

```ts
// Create from a lookup function
// preloadKeys: optional iterable of keys to eagerly build on creation
LayerMap.make<K, L extends Layer>(
  lookup: (key: K) => L,
  options?: {
    readonly idleTimeToLive?: Duration.Input
    readonly preloadKeys?: Iterable<K>
  }
): Effect<LayerMap<K, Layer.Success<L>, Layer.Error<L>>, ..., Scope | Layer.Services<L>>

// Create from a record of layers
// preload: if true, all layers in the record are eagerly built on creation
LayerMap.fromRecord(
  layers: Record<string, Layer>,
  options?: {
    readonly idleTimeToLive?: Duration.Input
    readonly preload?: boolean
  }
): Effect<LayerMap<...>, ..., Scope>
```

### LayerMap.Service -- Dependency-Injected LayerMap

For long-lived applications, you can wrap a `LayerMap` as a service using `LayerMap.Service`. This gives you a named service that exposes static `.get()`, `.services()`, and `.invalidate()` methods that thread through the service requirement automatically.

```ts
import { Console, Effect, Layer, LayerMap, ServiceMap } from "effect"

const Greeter = ServiceMap.Service<{
  readonly greet: Effect.Effect<string>
}>("Greeter")

class GreeterMap extends LayerMap.Service<GreeterMap>()("GreeterMap", {
  lookup: (name: string) =>
    Layer.succeed(Greeter)({
      greet: Effect.succeed(`Hello, ${name}!`)
    }),
  idleTimeToLive: "5 seconds"
}) {}

// Usage: GreeterMap.get("John") returns a Layer<Greeter, ..., GreeterMap>
const program = Effect.gen(function*() {
  const greeter = yield* Greeter
  yield* Console.log(yield* greeter.greet)
}).pipe(
  Effect.provide(GreeterMap.get("John")),
  Effect.provide(GreeterMap.layer)
)
```

### Example

```ts
import { Effect, Layer, LayerMap, ServiceMap } from "effect"

const Database = ServiceMap.Service<{
  readonly query: (sql: string) => Effect.Effect<string>
}>("Database")

const program = Effect.gen(function*() {
  const layerMap = yield* LayerMap.make(
    (env: string) =>
      Layer.succeed(Database)({
        query: (sql) => Effect.succeed(`${env}: ${sql}`)
      }),
    { idleTimeToLive: "5 seconds" }
  )

  // Provide a layer dynamically by key
  const result = yield* Effect.provide(
    Effect.gen(function*() {
      const db = yield* Database
      return yield* db.query("SELECT * FROM users")
    }),
    layerMap.get("production")
  )

  console.log(result) // "production: SELECT * FROM users"
}).pipe(Effect.scoped)
```

### When to Use

- **Multi-tenant applications** where each tenant gets different service implementations
- **Environment switching** (dev/staging/prod) at runtime
- **Plugin systems** where service implementations are determined by configuration

---

## ManagedRuntime -- Layer-Based Runtime

**Module:** `effect/ManagedRuntime`

A `ManagedRuntime<R, ER>` converts a `Layer<R, ER>` into a runtime that can execute effects with the layer's services pre-provided. It manages the layer's lifecycle (building on first use, disposing when done) and provides familiar run methods (`runFork`, `runSync`, `runPromise`, etc.).

> **v4 Change:** The `Runtime` module in v4 provides `Runtime.makeRunMain` for CLI/server process lifecycle management (SIGINT handling, exit codes). For pre-built service environments, use `ManagedRuntime`. For inline dependency provision, use `Effect.provide`.

### Type Definition

```ts
interface ManagedRuntime<in R, out ER> {
  readonly memoMap: Layer.MemoMap
  readonly servicesEffect: Effect<ServiceMap<R>, ER>
  readonly services: () => Promise<ServiceMap<R>>

  // Run methods -- all accept Effect<A, E, R> and handle service provision
  readonly runFork: <A, E>(self: Effect<A, E, R>, options?: Effect.RunOptions) => Fiber<A, E | ER>
  readonly runSync: <A, E>(effect: Effect<A, E, R>) => A
  readonly runSyncExit: <A, E>(effect: Effect<A, E, R>) => Exit<A, ER | E>
  readonly runCallback: <A, E>(effect: Effect<A, E, R>, options?) => (interruptor?: number) => void
  readonly runPromise: <A, E>(effect: Effect<A, E, R>, options?: Effect.RunOptions) => Promise<A>
  readonly runPromiseExit: <A, E>(effect: Effect<A, E, R>, options?: Effect.RunOptions) => Promise<Exit<A, ER | E>>

  // Lifecycle
  readonly dispose: () => Promise<void>
  readonly disposeEffect: Effect<void>

  // Note: scope and cachedServices are internal fields not intended for external use
}
```

### Constructor

```ts
ManagedRuntime.make<R, ER>(
  layer: Layer<R, ER, never>,
  options?: { readonly memoMap?: Layer.MemoMap }
): ManagedRuntime<R, ER>

// Guard
ManagedRuntime.isManagedRuntime(input: unknown): input is ManagedRuntime<unknown, unknown>
```

### Example

```ts
import { Console, Effect, Layer, ManagedRuntime, ServiceMap } from "effect"

class Notifications extends ServiceMap.Service<Notifications, {
  readonly notify: (message: string) => Effect.Effect<void>
}>()("Notifications") {
  static layer = Layer.succeed(this)({
    notify: (message) => Console.log(message)
  })
}

// Build a managed runtime from the layer
const runtime = ManagedRuntime.make(Notifications.layer)

// Use it to run effects with services pre-provided
async function main() {
  await runtime.runPromise(
    Effect.gen(function*() {
      const n = yield* Notifications
      yield* n.notify("Hello from ManagedRuntime!")
    })
  )
  await runtime.dispose()
}
```

### When to Use

- **Framework integration** (Next.js, Express, etc.) where you need to run effects outside of an Effect context
- **Long-lived server processes** where services are built once at startup
- **Testing** where you want to pre-build a test environment and run multiple test effects against it

---

## Match -- Pattern Matching

**Module:** `effect/Match`

The `Match` module provides exhaustive, type-safe pattern matching. It supports matching on types, values, discriminated unions, and structural patterns.

> **v4 Note:** The Match module lives at `effect/Match` and is imported from the main `"effect"` package. In early v3 it was a separate `@effect/match` package — that package no longer exists in v4.

### Core Workflow

1. **Create a matcher** with `Match.type<T>()` (returns a function) or `Match.value(v)` (returns a value)
2. **Add patterns** with `Match.when`, `Match.not`, `Match.tag`, etc.
3. **Finalize** with `Match.exhaustive`, `Match.orElse`, or `Match.option`

### Key Functions

```ts
// Create a type-level matcher (returns a function when finalized)
Match.type<T>(): TypeMatcher<T, ...>

// Create a value-level matcher (evaluates immediately when finalized)
Match.value<T>(value: T): ValueMatcher<T, ...>

// Pattern combinators
Match.when(pattern, handler)          // Match when pattern matches
Match.not(pattern, handler)           // Match when pattern does NOT match
Match.tag(tag, handler)               // Match discriminated union by _tag (supports multiple tags)
Match.tags({ tagA: handler, ... })    // Match multiple _tag values via an object map (non-exhaustive)
Match.tagsExhaustive({ ... })         // Like tags but enforces all _tags are handled
Match.whenOr(...patterns, handler)    // Match if any of the patterns match
Match.whenAnd(...patterns, handler)   // Match only if all patterns match simultaneously
Match.discriminator(field)(...tags, handler) // Match by an arbitrary discriminant field
Match.discriminators(field)({ tag: handler }) // Discriminant field object-map (non-exhaustive)
Match.discriminatorsExhaustive(field)({ tag: handler }) // Enforces all discriminant values

// Shorthand helpers for discriminated unions
Match.valueTags(input, { tag: handler }) // Match a value's _tag inline, returns result directly
Match.typeTags<T>()({ tag: handler })    // Returns a reusable function over T by _tag

// Finalizers
Match.exhaustive                // Compile-time error if any case is unhandled
Match.orElse(handler)           // Fallback handler for unmatched cases
Match.orElseAbsurd              // Throws at runtime if no pattern matches (use when you believe it unreachable)
Match.option                    // Returns Option.Some(value) on match, Option.None otherwise
Match.result                    // Returns Result.Ok(value) on match, Result.Err(unmatched) otherwise

// Utility
Match.withReturnType<Ret>()     // Constrain all branch return types to Ret (must be first in pipeline)

// Built-in type predicates for use as patterns
Match.string        // matches string values
Match.number        // matches number values
Match.boolean       // matches boolean values
Match.bigint        // matches bigint values
Match.symbol        // matches symbol values
Match.null          // matches null
Match.undefined     // matches undefined
Match.defined       // matches any non-null, non-undefined value
Match.any           // matches any value (catch-all)
Match.nonEmptyString // matches strings with at least one character
Match.record        // matches plain objects (excludes arrays, Dates, etc.)
Match.date          // matches Date instances
Match.is(...literals)           // matches one of the provided literal values
Match.instanceOf(Class)         // matches instances of a class
Match.instanceOfUnsafe(Class)   // like instanceOf but without type narrowing
```

### Example: Discriminated Union

```ts
import { Match } from "effect"

type Shape =
  | { readonly _tag: "Circle"; readonly radius: number }
  | { readonly _tag: "Rectangle"; readonly width: number; readonly height: number }
  | { readonly _tag: "Triangle"; readonly base: number; readonly height: number }

const area = Match.type<Shape>().pipe(
  Match.tag("Circle", ({ radius }) => Math.PI * radius * radius),
  Match.tag("Rectangle", ({ width, height }) => width * height),
  Match.tag("Triangle", ({ base, height }) => 0.5 * base * height),
  Match.exhaustive
)

console.log(area({ _tag: "Circle", radius: 5 }))     // ~78.54
console.log(area({ _tag: "Rectangle", width: 3, height: 4 })) // 12
```

### Example: Value Matching

```ts
import { Match } from "effect"

const input: string | number = "hello"

const result = Match.value(input).pipe(
  Match.when(Match.number, (n) => `number: ${n}`),
  Match.when(Match.string, (s) => `string: ${s}`),
  Match.exhaustive
)

console.log(result) // "string: hello"
```

### Example: Multiple Tags and `Match.tags`

`Match.tag` accepts multiple tags in a single call and matches any of them. `Match.tags` provides an object-map form:

```ts
import { Match } from "effect"

type Event =
  | { readonly _tag: "fetch" }
  | { readonly _tag: "success"; readonly data: string }
  | { readonly _tag: "error"; readonly error: Error }
  | { readonly _tag: "cancel" }

// Multiple tags in one Match.tag call
const describe = Match.type<Event>().pipe(
  Match.tag("fetch", "success", () => "Ok"),
  Match.tag("error", (e) => `Error: ${e.error.message}`),
  Match.tag("cancel", () => "Cancelled"),
  Match.exhaustive
)

// Equivalent using Match.tags (object map, non-exhaustive)
const describe2 = Match.type<Event>().pipe(
  Match.tags({
    fetch: () => "Ok",
    success: () => "Ok",
    error: (e) => `Error: ${e.error.message}`,
    cancel: () => "Cancelled",
  }),
  Match.exhaustive
)
```

### Example: `Match.result` vs `Match.option`

```ts
import { Match } from "effect"

type User = { readonly role: "admin" | "editor" | "viewer" }

const getRole = Match.type<User>().pipe(
  Match.when({ role: "admin" }, () => "Full access"),
  Match.when({ role: "editor" }, () => "Can edit"),
  Match.option  // Option.Some on match, Option.None on no match
)
// getRole({ role: "admin" })  => { _tag: "Some", value: "Full access" }
// getRole({ role: "viewer" }) => { _tag: "None" }

const getRoleResult = Match.type<User>().pipe(
  Match.when({ role: "admin" }, () => "Full access"),
  Match.when({ role: "editor" }, () => "Can edit"),
  Match.result  // Result.Ok on match, Result.Err(unmatched) on no match
)
// getRoleResult({ role: "viewer" }) => { _tag: "Err", err: { role: "viewer" } }
```

### Example: `Match.valueTags` and `Match.typeTags`

Use these when you only need to dispatch on `_tag` and all branches share a type:

```ts
import { Match } from "effect"

type Shape =
  | { readonly _tag: "Circle"; readonly radius: number }
  | { readonly _tag: "Rectangle"; readonly width: number; readonly height: number }

// valueTags: inline match on a specific value
const area1 = Match.valueTags({ _tag: "Circle", radius: 5 } as Shape, {
  Circle: ({ radius }) => Math.PI * radius * radius,
  Rectangle: ({ width, height }) => width * height,
})

// typeTags: create a reusable function
const computeArea = Match.typeTags<Shape>()({
  Circle: ({ radius }) => Math.PI * radius * radius,
  Rectangle: ({ width, height }) => width * height,
})
console.log(computeArea({ _tag: "Circle", radius: 5 })) // ~78.54
```

---

## Composition Best Practices

### Prefer Layer Composition Over Multiple Provides

In v4, layers are automatically memoized across `Effect.provide` calls (a safety improvement over v3). However, composing layers before providing is still the recommended approach:

```ts
// Preferred: compose then provide once
const AppLayer = Layer.mergeAll(DatabaseLayer, CacheLayer, LoggerLayer)
const program = myEffect.pipe(Effect.provide(AppLayer))

// Works but less explicit: multiple provides
const program2 = myEffect.pipe(
  Effect.provide(DatabaseLayer),
  Effect.provide(CacheLayer),
  Effect.provide(LoggerLayer)
)
```

### Use `Effect.gen` for Sequential Logic

Generator syntax is the most readable way to write sequential Effect code:

```ts
const program = Effect.gen(function*() {
  const db = yield* Database
  const user = yield* db.findUser(userId)
  const orders = yield* db.findOrders(user.id)
  return { user, orders }
})
```

### Use Combinators for Data-Parallel Operations

When operations are independent, use `Effect.all` with concurrency:

```ts
const program = Effect.gen(function*() {
  const [users, products, orders] = yield* Effect.all(
    [fetchUsers, fetchProducts, fetchOrders],
    { concurrency: 3 }
  )
  return { users, products, orders }
})
```

### Resource Acquisition Patterns

Always pair acquisition with release using `Effect.acquireRelease`:

```ts
const managedConnection = Effect.acquireRelease(
  Effect.sync(() => createConnection()),
  (conn) => Effect.sync(() => conn.close())
)

// Use within a scope
const program = Effect.scoped(
  Effect.gen(function*() {
    const conn = yield* managedConnection
    return yield* conn.query("SELECT 1")
  })
)
```

### Error Recovery Patterns

```ts
// Retry with policy
const resilientFetch = fetchData.pipe(
  Effect.retry(Schedule.exponential("100 millis").pipe(
    Schedule.compose(Schedule.recurs(3))
  ))
)

// Fallback (Effect.catch handles all typed errors; receives the error value)
const withFallback = primary.pipe(
  Effect.catch((e) => fallbackEffect)
)

// Timeout
const withTimeout = longRunning.pipe(
  Effect.timeout("5 seconds")
)
```

### Service.use vs yield* for Service Access

While `Service.use` is convenient for one-liners, prefer `yield*` in generators for clarity:

```ts
// One-liner: use is fine
const version = Config.useSync((c) => c.version)

// Multi-step: prefer yield*
const program = Effect.gen(function*() {
  const db = yield* Database
  const user = yield* db.findUser(id)
  yield* db.updateLastLogin(user.id)
  return user
})
```

---

## Resource Lifecycle Decision Tree

| Scenario | Module |
|---|---|
| Simple mutable state | `Ref` |
| State with effectful updates | `SynchronizedRef` |
| State with change notifications | `SubscriptionRef` |
| Single shared resource, ref-counted | `RcRef` |
| Keyed shared resources, ref-counted | `RcMap` |
| Resource that gets hot-swapped | `ScopedRef` |
| Dynamic layer selection by key | `LayerMap` |
| Fixed-size resource pool | `Pool` |
| Key-value cache with TTL | `Cache` |
| Key-value cache where values own resources | `ScopedCache` |
| Pre-built layer as runtime | `ManagedRuntime` |

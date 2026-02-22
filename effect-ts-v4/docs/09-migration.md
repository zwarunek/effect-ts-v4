# Migration Guide: Effect v3 to v4

This document consolidates all breaking changes and migration paths from Effect v3 to Effect v4. Use it as a comprehensive reference when upgrading an existing v3 codebase.

---

## Services: Context.Tag to ServiceMap.Service

The service definition system has been completely reworked.

### Defining Services

| v3 | v4 |
|---|---|
| `Context.GenericTag<T>(id)` | `ServiceMap.Service<T>(id)` |
| `Context.Tag(id)<Self, Shape>()` | `ServiceMap.Service<Self, Shape>()(id)` |
| `Effect.Tag(id)<Self, Shape>()` | `ServiceMap.Service<Self, Shape>()(id)` |
| `Effect.Service<Self>()(id, opts)` | `ServiceMap.Service<Self>()(id, { make })` |
| `Context.Reference<Self>()(id, opts)` | `ServiceMap.Reference<T>(id, opts)` |

**v3:**

```ts
import { Context } from "effect"

class Database extends Context.Tag("Database")<Database, {
  readonly query: (sql: string) => string
}>() {}
```

**v4:**

```ts
import { ServiceMap } from "effect"

class Database extends ServiceMap.Service<Database, {
  readonly query: (sql: string) => string
}>()("Database") {}
```

Note the difference in argument order: in v3, the identifier string is passed to `Context.Tag(id)` before the type parameters. In v4, the type parameters come first via `ServiceMap.Service<Self, Shape>()` and the identifier string is passed to the returned constructor `(id)`.

### Service Accessors (Effect.Tag proxy) -- Removed

In v3, `Effect.Tag` provided proxy access to service methods as static properties. In v4, use `Service.use` or `yield*` in a generator:

**v3:**

```ts
// Static proxy access
const program = Notifications.notify("hello")
```

**v4:**

```ts
// use: access the service and call a method in one step
const program = Notifications.use((n) => n.notify("hello"))

// Or more idiomatically:
const program2 = Effect.gen(function*() {
  const n = yield* Notifications
  yield* n.notify("hello")
})
```

### Effect.Service with make

In v3, `Effect.Service` auto-generated a `.Default` layer with wired `dependencies`. In v4, define layers explicitly:

**v3:**

```ts
class Logger extends Effect.Service<Logger>()("Logger", {
  effect: Effect.gen(function*() { ... }),
  dependencies: [Config.Default]
}) {}
// Logger.Default auto-generated
```

**v4:**

```ts
class Logger extends ServiceMap.Service<Logger>()("Logger", {
  make: Effect.gen(function*() { ... })
}) {
  static layer = Layer.effect(this, this.make).pipe(
    Layer.provide(Config.layer)
  )
}
```

The `dependencies` option no longer exists. Wire dependencies via `Layer.provide`.

v4 convention: name layers with `layer` (not `Default` or `Live`).

### Context to ServiceMap Operations

| v3 | v4 |
|---|---|
| `Context.make(tag, impl)` | `ServiceMap.make(tag, impl)` |
| `Context.get(ctx, tag)` | `ServiceMap.get(map, tag)` |
| `Context.add(ctx, tag, impl)` | `ServiceMap.add(map, tag, impl)` |
| `Context.mergeAll(...)` | `ServiceMap.mergeAll(...)` |

---

## Yieldable: No More Effect Subtypes

In v3, `Ref`, `Deferred`, `Fiber`, `FiberRef`, `Config`, `Option`, `Either`, and `Context.Tag` were structural subtypes of `Effect`. In v4, this is removed.

### What Changed

v4 introduces the `Yieldable` trait: a narrower contract that allows `yield*` in generators but does NOT make the type assignable to `Effect`.

Types that implement `Yieldable` (can still `yield*`): `Effect`, `Option`, `Result`, `Config`, `ServiceMap.Service`.

Types that are NO LONGER Effect subtypes: `Ref`, `Deferred`, `Fiber`.

### Migration

| Type | v3 | v4 |
|---|---|---|
| `Ref<A>` | `yield* ref` reads the value | `yield* Ref.get(ref)` |
| `Deferred<A, E>` | `yield* deferred` awaits | `yield* Deferred.await(deferred)` |
| `Fiber<A, E>` | `yield* fiber` joins | `yield* Fiber.join(fiber)` |

When passing to Effect combinators, use `.asEffect()`:

**v3:**

```ts
const program = Effect.map(Option.some(42), (n) => n + 1)
```

**v4:**

```ts
const program = Effect.map(Option.some(42).asEffect(), (n) => n + 1)

// Or use a generator:
const program2 = Effect.gen(function*() {
  const n = yield* Option.some(42)
  return n + 1
})
```

---

## Forking: Renamed Combinators

| v3 | v4 | Description |
|---|---|---|
| `Effect.fork` | `Effect.forkChild` | Fork as a child of the current fiber |
| `Effect.forkDaemon` | `Effect.forkDetach` | Fork detached from parent lifecycle |
| `Effect.forkScoped` | `Effect.forkScoped` | Fork tied to the current Scope (unchanged) |
| `Effect.forkIn` | `Effect.forkIn` | Fork in a specific Scope (unchanged) |
| `Effect.forkAll` | Removed | Use `Effect.all` with concurrency or fork individually |
| `Effect.forkWithErrorHandler` | Removed | Use `Fiber.join` or `Fiber.await` |

### New Fork Options

All fork variants accept an optional options object:

```ts
{
  readonly startImmediately?: boolean   // Begin executing immediately
  readonly uninterruptible?: boolean | "inherit"  // Interruption control
}
```

```ts
const fiber = yield* Effect.forkChild(myEffect, {
  startImmediately: true,
  uninterruptible: true
})
```

---

## Cause: Flattened Structure

In v3, `Cause<E>` was a recursive tree with six variants (`Empty | Fail | Die | Interrupt | Sequential | Parallel`). In v4, it is a flat array of reasons:

```ts
interface Cause<E> {
  readonly reasons: ReadonlyArray<Reason<E>>
}

type Reason<E> = Fail<E> | Die | Interrupt
```

### Removed Variants

`Empty`, `Sequential`, and `Parallel` are removed. An empty cause has an empty `reasons` array. Multiple failures are collected into a flat array.

### Accessing Reasons

**v3:**

```ts
switch (cause._tag) {
  case "Fail": return cause.error
  case "Die": return cause.defect
  case "Empty": return undefined
  case "Sequential": return handle(cause.left)
  case "Parallel": return handle(cause.left)
  case "Interrupt": return cause.fiberId
}
```

**v4:**

```ts
for (const reason of cause.reasons) {
  switch (reason._tag) {
    case "Fail": return reason.error
    case "Die": return reason.defect
    case "Interrupt": return reason.fiberId
  }
}
```

### Predicate Renames

| v3 | v4 |
|---|---|
| `Cause.isFailType(cause)` | `Cause.isFailReason(reason)` |
| `Cause.isDieType(cause)` | `Cause.isDieReason(reason)` |
| `Cause.isInterruptType(cause)` | `Cause.isInterruptReason(reason)` |
| `Cause.isEmptyType(cause)` | `cause.reasons.length === 0` |
| `Cause.isFailure(cause)` | `Cause.hasFails(cause)` |
| `Cause.isDie(cause)` | `Cause.hasDies(cause)` |
| `Cause.isInterrupted(cause)` | `Cause.hasInterrupts(cause)` |
| `Cause.isInterruptedOnly(cause)` | `Cause.hasInterruptsOnly(cause)` |

### Constructor Renames

| v3 | v4 |
|---|---|
| `Cause.sequential(left, right)` | `Cause.combine(left, right)` |
| `Cause.parallel(left, right)` | `Cause.combine(left, right)` |

### Extractor Renames

| v3 | v4 |
|---|---|
| `Cause.failureOption(cause)` | `Cause.findErrorOption(cause)` |
| `Cause.failureOrCause(cause)` | `Cause.findError(cause)` |
| `Cause.dieOption(cause)` | `Cause.findDefect(cause)` |
| `Cause.interruptOption(cause)` | `Cause.findInterrupt(cause)` |
| `Cause.failures(cause)` | `cause.reasons.filter(Cause.isFailReason)` |
| `Cause.defects(cause)` | `cause.reasons.filter(Cause.isDieReason)` |

Note: `findError` and `findDefect` return `Result.Result` instead of `Option`. Use `findErrorOption` for the `Option`-based variant.

### Error Class Renames

All `*Exception` classes renamed to `*Error`:

| v3 | v4 |
|---|---|
| `Cause.NoSuchElementException` | `Cause.NoSuchElementError` |
| `Cause.TimeoutException` | `Cause.TimeoutError` |
| `Cause.IllegalArgumentException` | `Cause.IllegalArgumentError` |
| `Cause.ExceededCapacityException` | `Cause.ExceededCapacityError` |
| `Cause.UnknownException` | `Cause.UnknownError` |
| `Cause.RuntimeException` | Removed |
| `Cause.InterruptedException` | Removed |
| `Cause.InvalidPubSubCapacityException` | Removed |

### New in v4

- `Cause.fromReasons(reasons)` -- construct from an array of `Reason` values
- `Cause.makeFailReason(error)`, `Cause.makeDieReason(defect)`, `Cause.makeInterruptReason(fiberId)`
- `Cause.annotate(cause, annotations)`
- `Cause.Done` -- graceful completion signal for queues and streams
- `Cause.findFail(cause)` -- extract the first `Fail` reason as a `Result`
- `Cause.findDie(cause)` -- extract the first `Die` reason as a `Result`
- `Cause.filterInterruptors(cause)` -- extract interrupting fiber IDs as a `Result`

---

## Error Handling: catch* Renamings

| v3 | v4 |
|---|---|
| `Effect.catchAll` | `Effect.catch` |
| `Effect.catchAllCause` | `Effect.catchCause` |
| `Effect.catchAllDefect` | `Effect.catchDefect` |
| `Effect.catchSome` | `Effect.catchIf` |
| `Effect.catchSomeCause` | `Effect.catchCauseIf` |
| `Effect.catchIf` (predicate-only) | `Effect.catchIf` (merged; now also replaces `catchSome`) |
| `Effect.catchSomeDefect` | Removed |
| `Effect.catchTag` | `Effect.catchTag` (unchanged) |
| `Effect.catchTags` | `Effect.catchTags` (unchanged) |

### catchSome to catchIf

`catchSome` returned `Option`. `catchIf` accepts a predicate, refinement, or `Filter`:

**v3:**

```ts
Effect.catchSome((error) =>
  error === 42
    ? Option.some(Effect.succeed("caught"))
    : Option.none()
)
```

**v4:**

```ts
Effect.catchIf(
  (error: number) => error === 42,
  (error) => Effect.succeed("caught")
)
```

`catchIf` accepts predicates and refinements directly (no need to wrap in `Filter.fromPredicate`), and also accepts a `Filter` value for more advanced use cases.

### New in v4

- `Effect.catchReason(errorTag, reasonTag, handler)` -- catches a specific reason within a tagged error
- `Effect.catchReasons(errorTag, cases)` -- handles multiple reason tags via an object
- `Effect.catchEager(handler)` -- optimization variant that evaluates synchronous recovery immediately

---

## Equality: Structural by Default

In v3, `Equal.equals` used reference equality for plain objects. In v4, it uses **structural equality** by default:

```ts
Equal.equals({ a: 1 }, { a: 1 }) // v3: false, v4: true
Equal.equals([1, 2], [1, 2])     // v3: false, v4: true
Equal.equals(new Map([["a", 1]]), new Map([["a", 1]])) // v4: true
Equal.equals(NaN, NaN)           // v3: false, v4: true
```

### Opting Out

Use `Equal.byReference(obj)` (creates a proxy) or `Equal.byReferenceUnsafe(obj)` (mutates in place) for reference equality.

### Rename

`Equal.equivalence<T>()` renamed to `Equal.asEquivalence<T>()`.

---

## Generators: Passing `this`

In v3, `this` was passed as the first argument. In v4, wrap it in an options object:

**v3:**

```ts
class MyService {
  compute = Effect.gen(this, function*() {
    return this.local + 1
  })
}
```

**v4:**

```ts
class MyService {
  compute = Effect.gen({ self: this }, function*() {
    return this.local + 1
  })
}
```

---

## FiberRef: Removed

`FiberRef`, `FiberRefs`, and `FiberRefsPatch` have been removed. Fiber-local state is now handled by `ServiceMap.Reference`. (`Differ` still exists as a low-level interface.)

### Built-in References

| v3 FiberRef | v4 Reference |
|---|---|
| `FiberRef.currentConcurrency` | `References.CurrentConcurrency` |
| `FiberRef.currentLogLevel` | `References.CurrentLogLevel` |
| `FiberRef.currentMinimumLogLevel` | `References.MinimumLogLevel` |
| `FiberRef.currentLogAnnotations` | `References.CurrentLogAnnotations` |
| `FiberRef.currentLogSpan` | `References.CurrentLogSpans` |
| `FiberRef.currentScheduler` | `References.Scheduler` |
| `FiberRef.currentMaxOpsBeforeYield` | `References.MaxOpsBeforeYield` |
| `FiberRef.currentTracerEnabled` | `References.TracerEnabled` |
| `FiberRef.unhandledErrorLogLevel` | `References.UnhandledLogLevel` |

### Reading

**v3:**

```ts
const level = yield* FiberRef.get(FiberRef.currentLogLevel)
```

**v4:**

```ts
const level = yield* References.CurrentLogLevel
```

### Scoped Updates

**v3:** `Effect.locally(effect, FiberRef.currentLogLevel, LogLevel.Debug)`

**v4:** `Effect.provideService(effect, References.CurrentLogLevel, "Debug")`

---

## Scope: extend to provide

`Scope.extend` has been renamed to `Scope.provide`:

**v3:**

```ts
yield* Scope.extend(myEffect, scope)
```

**v4:**

```ts
yield* Scope.provide(scope)(myEffect)
// or data-first:
yield* Scope.provide(myEffect, scope)
```

---

## Layer Memoization

In v3, layers were memoized only within a single `Effect.provide` call. In v4, layers are automatically memoized across `Effect.provide` calls via a shared `MemoMap`.

### Opting Out

- `Layer.fresh(layer)` -- always builds with a fresh memo map
- `Effect.provide(layer, { local: true })` -- builds with a local memo map (new in v4)

---

## Runtime: Runtime<R> Removed

In v3, `Runtime<R>` bundled `Context<R>`, `RuntimeFlags`, and `FiberRefs`. In v4, this type is removed. Use `ManagedRuntime` or provide services directly.

The `Runtime` module now only contains:
- `Teardown` -- interface for handling process exit
- `defaultTeardown`
- `makeRunMain` -- creates platform-specific main runners

---

## Fiber Keep-Alive

In v3, fibers suspended on async operations (like `Deferred.await`) did not keep the Node.js process alive. You needed `runMain` from `@effect/platform-node`.

In v4, the Effect runtime automatically manages a reference-counted keep-alive timer. `Effect.runPromise` now keeps the process alive while fibers are suspended.

`runMain` is still recommended for signal handling, exit code management, and error reporting.

---

## Data Type Renames

`Either` has been replaced by `Result`. The naming of variants, constructors, and guards has changed completely:

| v3 | v4 |
|---|---|
| `Either<A, E>` | `Result<A, E>` |
| `Either.Left` (variant) | `Result.Failure` (variant) |
| `Either.Right` (variant) | `Result.Success` (variant) |
| `Either.left(e)` | `Result.fail(e)` |
| `Either.right(a)` | `Result.succeed(a)` |
| `Either.isLeft` | `Result.isFailure` |
| `Either.isRight` | `Result.isSuccess` |

The `Result.Failure` variant exposes the error via `.failure` (not `.left`). The `Result.Success` variant exposes the value via `.success` (not `.right`).

---

## Quick Migration Checklist

1. Replace all `Context.Tag` / `Context.GenericTag` / `Effect.Tag` / `Effect.Service` with `ServiceMap.Service`
2. Replace `yield* ref` with `yield* Ref.get(ref)`; `yield* deferred` with `yield* Deferred.await(deferred)`; `yield* fiber` with `yield* Fiber.join(fiber)`
3. Rename `Effect.fork` to `Effect.forkChild`, `Effect.forkDaemon` to `Effect.forkDetach`
4. Rename `Effect.catchAll` to `Effect.catch`, `Effect.catchAllCause` to `Effect.catchCause`, `Effect.catchSome` to `Effect.catchIf`, `Effect.catchSomeCause` to `Effect.catchCauseIf`, etc.
5. Replace `FiberRef` usage with `ServiceMap.Reference` / `References.*`
6. Replace `Scope.extend` with `Scope.provide`
7. Replace `Either` with `Result`: `Left`/`Right` variants become `Failure`/`Success`; `Either.left(e)` → `Result.fail(e)`; `Either.right(a)` → `Result.succeed(a)`; `Either.isLeft` → `Result.isFailure`; `Either.isRight` → `Result.isSuccess`
8. Update `Cause` pattern matching to iterate `cause.reasons` instead of recursive tree matching
9. Rename all `*Exception` to `*Error` (e.g., `NoSuchElementException` to `NoSuchElementError`)
10. Update `Effect.gen(this, function*() { ... })` to `Effect.gen({ self: this }, function*() { ... })`
11. Replace `Effect.provide(layer, { local: true })` where test isolation is needed
12. Remove `@effect/schema` dependency -- Schema is now in `effect`

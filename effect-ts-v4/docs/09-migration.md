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

## Tagged Errors: Data.TaggedError Constructor Change

In v3, `Data.TaggedError` accepted a type parameter on the class extension to define fields. In v4, the constructor is generic at the call site instead, which breaks the `extends Data.TaggedError("Tag")<{ fields }>` pattern at the TypeScript type level.

### Migration

**v3:**

```ts
import { Data } from "effect"

class NotFoundError extends Data.TaggedError("NotFoundError")<{
  readonly id: string
}> {}

new NotFoundError({ id: "123" }) // works
```

**v4 — use `Schema.TaggedErrorClass`:**

```ts
import { Schema } from "effect"

class NotFoundError extends Schema.TaggedErrorClass<NotFoundError>()(
  "NotFoundError",
  { id: Schema.String },
) {}

new NotFoundError({ id: "123" }) // works, with runtime validation
```

For errors **without fields**, `Data.TaggedError` still works fine:

```ts
class EmptyError extends Data.TaggedError("EmptyError") {}
new EmptyError() // works
```

For errors with complex object fields (not just primitives), use `Schema.Any`:

```ts
class ComplexError extends Schema.TaggedErrorClass<ComplexError>()(
  "ComplexError",
  { payload: Schema.Any },
) {}
```

---

## Schema API Changes

### Schema.Union takes an array

**v3:**

```ts
Schema.Union(Schema.String, Schema.Number)
Schema.Union(Schema.Literal("a"), Schema.Literal("b"))
```

**v4:**

```ts
Schema.Union([Schema.String, Schema.Number])
Schema.Union([Schema.Literal("a"), Schema.Literal("b")])
```

### Schema.Literal takes a single value; use Schema.Literals for multiple

**v3:**

```ts
Schema.Literal("a", "b", "c")
```

**v4:**

```ts
Schema.Literal("a")               // single literal
Schema.Literals(["a", "b", "c"])   // union of literals
```

### Schema.decodeUnknown → Schema.decodeUnknownEffect

The function that returns an `Effect` was renamed. `decodeUnknownSync` is unchanged.

| v3 | v4 | Returns |
|---|---|---|
| `Schema.decodeUnknown(S)(input)` | `Schema.decodeUnknownEffect(S)(input)` | `Effect<T, SchemaError>` |
| `Schema.encodeUnknown(S)(input)` | `Schema.encodeUnknownEffect(S)(input)` | `Effect<E, SchemaError>` |
| `Schema.decodeUnknownSync(S)(input)` | `Schema.decodeUnknownSync(S)(input)` | `T` (throws) |
| `Schema.encodeSync(S)(input)` | `Schema.encodeSync(S)(input)` | `E` (throws) |
| `ParseError` | `SchemaError` | Error type |

### Schema.Class.make → Schema.Class.makeUnsafe

```ts
// v3
const user = User.make({ name: "Alice" })

// v4
const user = User.makeUnsafe({ name: "Alice" })
```

### Schema.Record takes positional args

```ts
// v3
Schema.Record({ key: Schema.String, value: Schema.Number })

// v4
Schema.Record(Schema.String, Schema.Number)
```

### Schema.transform → Schema.decodeTo

```ts
// v3
const codec = Schema.transform(FromSchema, ToSchema, {
  decode: (from) => toValue,
  encode: (to) => fromValue,
})

// v4
import { SchemaGetter } from "effect"

const codec = FromSchema.pipe(
  Schema.decodeTo(ToSchema, {
    decode: SchemaGetter.transform((from) => toValue),
    encode: SchemaGetter.transform((to) => fromValue),
  })
)
```

### Schema.optional / Schema.Set

`Schema.optional` works the same. `Schema.Set` has been removed; use `Schema.Array` with uniqueness checks or `Schema.Unknown` for unvalidated sets.

---

## Other Renamed / Removed APIs

| v3 | v4 | Notes |
|---|---|---|
| `Effect.zipRight(a, b)` | `a.pipe(Effect.andThen(b))` | `zipRight` removed |
| `Effect.firstSuccessOf(effects)` | `Effect.raceAll(effects)` | Renamed |
| `Effect.orDieWith(f)` | `Effect.mapError(f), Effect.orDie` | Chain them |
| `Effect.try(() => value)` | `Effect.try({ try: () => value, catch: (e) => error })` | Bare-function overload removed |
| `Effect.tryPromise(async () => v)` | `Effect.tryPromise({ try: async () => v, catch: (e) => e })` | Bare-function overload removed |
| `Effect.tap(voidFn)` | `Effect.tap((x) => Effect.sync(() => voidFn(x)))` | `tap` callback must return `Effect`, not `void` |
| `LogLevel.Info` / `.label` | `"Info"` (string literal) | `LogLevel` is now a string union type |
| `Logger.replace(old, new)` | `Logger.layer([myLogger])` | Provide as a layer |
| `Logger.Logger<I, O>` | `Logger<I, O>` | Same, but `Logger.layer` takes an array |
| `Brand.make(value)` / `UserId.make(v)` | `Schema.decodeSync(BrandSchema)(v)` | For schema brands; `Brand.nominal<T>()` for standalone |
| `Layer.Layer.Success<T>` | Removed entirely | `Layer.Layer` nested namespace gone |
| `Effect.Effect.Error<T>` | `Effect.Error<T>` | Namespace flattened |
| `Effect.Effect.Success<T>` | `Effect.Success<T>` | Namespace flattened |
| `ManagedRuntime.ManagedRuntime.Context<T>` | Removed | Nested namespace gone; use `any` or infer |
| `Schema.Object` | `Schema.Unknown` | `Schema.Object` removed entirely |
| `@effect/platform` imports | `effect/unstable/http` imports | `HttpClient`, `HttpBody`, `HttpClientRequest`, `HttpClientResponse`, `FetchHttpClient` moved |
| `Schedule.upTo(n)` | `Schedule.compose(Schedule.recurs(n))` | `upTo` removed |
| `Span.parent` (yieldable) | `span.parent` (plain property, may be undefined) | No longer yieldable; must null-check |

---

## Package Changes

| v3 | v4 |
|---|---|
| `effect@^3.x` | `effect@4.0.0-beta.x` |
| `@effect/platform` | Removed — use `effect/unstable/http` |
| `@effect/platform-node` | `@effect/platform-node@4.0.0-beta.x` |
| `@effect/opentelemetry` | `@effect/opentelemetry@4.0.0-beta.x` |
| `@effect/vitest` | `@effect/vitest@4.0.0-beta.x` |
| `@effect/schema` | Removed — use `effect` (Schema in core) |
| `@effect/language-service` | No v4 beta yet |

---

## Common Pitfalls (Learned from Real Migrations)

These are issues NOT obvious from the API rename tables. They will bite you during typecheck:

### 1. Config is NOT an Effect subtype

`Config` implements `Yieldable` (so `yield*` works in generators) but is NOT assignable to `Effect`. This means `Effect.orDie(Config.string("X"))` **fails** in v4.

**v3:**
```ts
const envValue = Effect.orDie(Config.string("SERVICE_IDENTIFIER"))
```

**v4:**
```ts
// Config must be yielded inside a generator
const envValue = Effect.gen(function*() {
  return yield* Config.string("SERVICE_IDENTIFIER")
}).pipe(Effect.orDie)
```

### 2. `class extends Schema.Struct({...})` breaks

The pattern `class Foo extends S.Struct({...}) {}` (NOT `Schema.Class`) was valid in v3 for creating lightweight schema-backed classes. In v4, this pattern produces an object that is NOT a valid Schema and cannot be passed to `Schema.decodeUnknownEffect`.

**Fix:** Convert to plain schema values with separate type aliases:
```ts
// v3
export class MySchema extends S.Struct({ name: S.String }) {}

// v4
export const MySchema = S.Struct({ name: S.String })
export type MySchema = S.Schema.Type<typeof MySchema>
```

### 3. Effect.catch error type is `unknown`

When migrating `Effect.catchAll` → `Effect.catch`, the error parameter in the handler may be typed as `unknown` (not the inferred error union). If you access `._tag` on the error, you need a type assertion:

```ts
// May need explicit type annotation
Effect.catch((e: any) => {
  console.log(e._tag) // e is unknown without the annotation
  return Effect.succeed(fallback)
})
```

### 4. Schema.Literals takes an array directly

When you have an array variable, pass it directly — don't wrap in another array:
```ts
const items = ["a", "b", "c"] as const
Schema.Literals(items)      // ✅ correct
Schema.Literals([items])    // ❌ wrong — double-wrapped
```

### 5. Schema.transform → Schema.decodeTo requires SchemaGetter wrappers

The decode/encode functions must be wrapped in `SchemaGetter.transform()`. Plain functions won't work:
```ts
// v4
import { SchemaGetter } from "effect"
FromSchema.pipe(Schema.decodeTo(ToSchema, {
  decode: SchemaGetter.transform((from: FromType): ToType => ...),
  encode: SchemaGetter.transform((to: ToType): FromType => ...),
}))
```

### 6. Logger.make callback must return void

In v3, the logger callback could return `Effect.runPromise(...)`. In v4 it must return `void`. Wrap in a fire-and-forget:
```ts
export const logger = Logger.make(
  ({ logLevel, message }) => {
    Effect.runPromise(logEffect) // fire-and-forget, no return
  }
)
```

### 7. Schema.decode is gone — use Schema.decodeUnknownEffect

`Schema.decode(MySchema)(input)` no longer exists. Use `Schema.decodeUnknownEffect(MySchema)(input)` for the Effect-returning variant.

---

## Quick Migration Checklist

### Phase 1: Package updates
1. Update `effect` to `4.0.0-beta.x`
2. Remove `@effect/platform` — replace imports with `effect/unstable/http`
3. Remove `@effect/schema` — Schema is now in `effect`
4. Update `@effect/platform-node`, `@effect/opentelemetry`, `@effect/vitest` to `4.0.0-beta.x`
5. Run `pnpm install`

### Phase 2: Service definitions (do these first — everything depends on them)
6. Replace all `Context.Tag` / `Effect.Tag` / `Effect.Service` with `ServiceMap.Service`
7. Replace `effect: make` with `make: make` in service options
8. Remove `accessors: true` and `dependencies: [...]` from all services
9. Add explicit `static layer = Layer.effect(this, this.make).pipe(Layer.provide(...))` to each service
10. Rename all `.Default` references to `.layer`
11. Replace all accessor calls (`ServiceName.fieldName`) with `yield*` pattern: `const svc = yield* ServiceName; svc.fieldName`

### Phase 3: Error classes
12. Replace `Data.TaggedError("Tag")<{ fields }>` with `Schema.TaggedErrorClass<Self>()("Tag", { schemaFields })`
13. Errors without fields: keep `Data.TaggedError("Tag")` as-is

### Phase 4: API renames (use find-and-replace)
14. `Effect.catchAll` → `Effect.catch`
15. `Effect.catchAllCause` → `Effect.catchCause`
16. `Effect.catchSome` → `Effect.catchIf`
17. `Effect.zipRight` → `Effect.andThen`
18. `Effect.firstSuccessOf` → `Effect.raceAll`
19. `Effect.fork` → `Effect.forkChild`
20. `Effect.forkDaemon` → `Effect.forkDetach`
21. `NoSuchElementException` → `NoSuchElementError`
22. `Schema.decodeUnknown(S)(input)` → `Schema.decodeUnknownEffect(S)(input)`
23. `Schema.encodeUnknown(S)(input)` → `Schema.encodeUnknownEffect(S)(input)`
24. `Schema.Class.make(...)` → `Schema.Class.makeUnsafe(...)`
25. `Schema.Union(A, B, C)` → `Schema.Union([A, B, C])`
26. `Schema.Literal("a", "b")` → `Schema.Literals(["a", "b"])`
27. `Schema.Record({ key: K, value: V })` → `Schema.Record(K, V)`
28. `Schema.Object` → `Schema.Unknown`
29. `Schema.transform(From, To, opts)` → `From.pipe(Schema.decodeTo(To, { decode: SchemaGetter.transform(...), encode: SchemaGetter.transform(...) }))`
30. `Effect.tryPromise(async () => v)` → `Effect.tryPromise({ try: async () => v, catch: (e) => e })`
31. `Effect.try(() => v)` → `Effect.try({ try: () => v, catch: (e) => e })`
32. `Effect.orDieWith(f)` → `Effect.mapError(f)` then `Effect.orDie`
33. `ParseError` (from `effect/ParseResult`) → `SchemaError` (from `effect/Schema`)
34. `class Foo extends S.Struct({...}) {}` → `const Foo = S.Struct({...}); type Foo = S.Schema.Type<typeof Foo>`
35. `Effect.Effect.Success<T>` → `Effect.Success<T>`
36. `Effect.Effect.Error<T>` → `Effect.Error<T>`

### Phase 5: Logger and runtime
27. Replace `Logger.replace(Logger.defaultLogger, myLogger)` with `Logger.layer([myLogger])`
28. `LogLevel` is now a string (`"Info"`, `"Error"`, etc.), not an object with `.label`
29. `Logger.make` callback must return `void` (not `Effect`) — use fire-and-forget for async logging

### Phase 6: Verify
30. Run typecheck (`tsc --noEmit`)
31. Run tests
32. Check bundle size — v4 typically reduces bundle due to `@effect/platform` consolidation

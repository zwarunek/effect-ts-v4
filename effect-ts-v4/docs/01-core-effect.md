# Core Effect

> The foundation of Effect 4.0: the `Effect` type for modeling composable, type-safe computations; `Result` for synchronous success/failure values (replacing `Either` from v3); `Exit` and `Cause` for inspecting outcomes; `Scope` for resource management; and the `pipe`/`flow` utility functions that tie everything together.

---

## Function Utilities (pipe, flow, identity, dual)

### Overview

Core utility functions for composition. `pipe` threads a value through a pipeline of functions. `flow` composes functions left-to-right. `identity` returns its argument unchanged. `dual` creates functions usable in both data-first and data-last styles.

### pipe

Threads a value through a sequence of unary functions, left to right.

```ts
function pipe<A>(a: A): A
function pipe<A, B>(a: A, ab: (a: A) => B): B
function pipe<A, B, C>(a: A, ab: (a: A) => B, bc: (b: B) => C): C
// ... up to 20 functions
```

```ts
import { pipe } from "effect"

const result = pipe(
  5,
  (x) => x * 2,   // 10
  (x) => x + 1,   // 11
  (x) => x.toString() // "11"
)
```

### flow

Composes functions left-to-right, returning a new function. The first function may have any arity; the rest must be unary.

```ts
function flow<A extends ReadonlyArray<unknown>, B>(ab: (...a: A) => B): (...a: A) => B
function flow<A extends ReadonlyArray<unknown>, B, C>(
  ab: (...a: A) => B, bc: (b: B) => C
): (...a: A) => C
// ... up to 9 functions
```

```ts
import { flow } from "effect"

const len = (s: string): number => s.length
const double = (n: number): number => n * 2
const f = flow(len, double)

f("aaa") // 6
```

### identity

Returns its input argument unchanged. Useful as a default transformation.

```ts
const identity = <A>(a: A): A => a
```

### dual

Creates a function usable in both data-first (`f(data, arg)`) and data-last (`pipe(data, f(arg))`) styles. The first parameter is the arity of the uncurried form.

```ts
import { dual, pipe } from "effect"

const sum: {
  (that: number): (self: number) => number
  (self: number, that: number): number
} = dual(2, (self: number, that: number): number => self + that)

sum(2, 3)        // 5 (data-first)
pipe(2, sum(3))  // 5 (data-last)
```

### Pipeable Interface

All Effect types implement `Pipeable`, which adds a `.pipe()` method:

```ts
interface Pipeable {
  pipe<A>(this: A): A
  pipe<A, B>(this: A, ab: (_: A) => B): B
  // ... up to 20 functions
}
```

```ts
const program = Effect.succeed(1).pipe(
  Effect.map((x) => x + 1),
  Effect.flatMap((x) => Effect.succeed(x * 2))
)
```

---

## Effect\<A, E, R\>

### Overview

The `Effect` module is the core of the Effect library. An `Effect<A, E, R>` represents a computation that:
- May succeed with a value of type `A`
- May fail with an error of type `E` (defaults to `never`)
- Requires a context/environment of type `R` (defaults to `never`)

Effects are lazy and immutable -- they describe computations that are executed later. Key features include type-safe error handling, resource management, structured concurrency, composability, and interruptibility.

### Key Types

```ts
interface Effect<out A, out E = never, out R = never>
  extends Pipeable, Yieldable<Effect<A, E, R>, A, E, R> {
  readonly [TypeId]: Variance<A, E, R>
}

// Type-level extractors
type Effect.Success<T>  // extracts A from Effect<A, E, R>
type Effect.Error<T>    // extracts E
type Effect.Services<T> // extracts R
```

### Constructors

#### succeed / fail

```ts
const succeed: <A>(value: A) => Effect<A>
const fail: <E>(error: E) => Effect<never, E>
```

```ts
const success = Effect.succeed(42)       // Effect<number>
const failure = Effect.fail("error")     // Effect<never, string>
```

#### sync / suspend

```ts
const sync: <A>(thunk: () => A) => Effect<A>
const suspend: <A, E, R>(effect: () => Effect<A, E, R>) => Effect<A, E, R>
```

`sync` wraps a synchronous side-effectful thunk. `suspend` defers effect creation (useful for recursion and type unification).

```ts
const log = (msg: string) => Effect.sync(() => { console.log(msg) })

// Recursive -- use suspend to avoid stack overflow
const fib = (n: number): Effect.Effect<number> =>
  n < 2
    ? Effect.succeed(1)
    : Effect.zipWith(
        Effect.suspend(() => fib(n - 1)),
        Effect.suspend(() => fib(n - 2)),
        (a, b) => a + b
      )
```

#### promise / tryPromise

```ts
const promise: <A>(evaluate: (signal: AbortSignal) => PromiseLike<A>) => Effect<A>

const tryPromise: <A, E = Cause.UnknownError>(
  options: ((signal: AbortSignal) => PromiseLike<A>)
    | { readonly try: (signal: AbortSignal) => PromiseLike<A>; readonly catch: (error: unknown) => E }
) => Effect<A, E>
```

`promise` wraps a Promise that should never reject (rejections become defects). `tryPromise` catches rejections and maps them to typed errors.

```ts
const fetchData = Effect.tryPromise({
  try: () => fetch("https://api.example.com/data"),
  catch: (error) => new Error(`Fetch failed: ${error}`)
})
```

#### callback

```ts
const callback: <A, E = never, R = never>(
  register: (
    resume: (effect: Effect<A, E, R>) => void,
    signal: AbortSignal
  ) => void | Effect<void, never, R>
) => Effect<A, E, R>
```

Wraps callback-based APIs. The `register` function receives a `resume` callback to complete the effect and an `AbortSignal` for interruption. The `register` function may optionally return a cleanup `Effect` that runs if the fiber is interrupted before `resume` is called.

#### gen (Generator syntax)

```ts
const gen: {
  <Eff extends Yieldable<any, any, any, any>, AEff>(
    f: () => Generator<Eff, AEff, never>
  ): Effect<AEff, /* E extracted from Eff */, /* R extracted from Eff */>
  // Overload with `self` binding for class methods:
  <Self, Eff extends Yieldable<any, any, any, any>, AEff>(
    options: { readonly self: Self },
    f: (this: Self) => Generator<Eff, AEff, never>
  ): Effect<AEff, /* E extracted from Eff */, /* R extracted from Eff */>
}
```

The primary way to compose effects. Use `yield*` to unwrap effects inside a generator -- if any yields fail, the generator short-circuits. The error type `E` and requirements type `R` are inferred automatically from the yielded effects. Use the `{ self }` overload to bind a `this` context (useful in class methods).

```ts
import { Effect } from "effect"

const program = Effect.gen(function*() {
  const a = yield* Effect.succeed(10)
  const b = yield* Effect.succeed(20)
  return a + b
})

Effect.runPromise(program).then(console.log) // 30
```

#### Other Constructors

```ts
const die: (defect: unknown) => Effect<never>           // unrecoverable defect
const failCause: <E>(cause: Cause<E>) => Effect<never, E>
const never: Effect<never>                               // runs forever
const void: Effect<void>                                 // succeeds with undefined
const fromResult: <A, E>(result: Result<A, E>) => Effect<A, E>
const fromOption: <A>(option: Option<A>) => Effect<A, Cause.NoSuchElementError>
```

### Transformations

#### map

Transforms the success value. Does not affect failures.

```ts
const map: {
  <A, B>(f: (a: A) => B): <E, R>(self: Effect<A, E, R>) => Effect<B, E, R>
  <A, E, R, B>(self: Effect<A, E, R>, f: (a: A) => B): Effect<B, E, R>
}
```

```ts
// pipe style
pipe(Effect.succeed(3), Effect.map((n) => n * 2))
// data-first style
Effect.map(Effect.succeed(3), (n) => n * 2)
```

#### flatMap

Chains effects sequentially. The function receives the success value and must return an Effect.

```ts
const flatMap: {
  <A, B, E1, R1>(f: (a: A) => Effect<B, E1, R1>): <E, R>(self: Effect<A, E, R>) => Effect<B, E | E1, R | R1>
  <A, E, R, B, E1, R1>(self: Effect<A, E, R>, f: (a: A) => Effect<B, E1, R1>): Effect<B, E | E1, R | R1>
}
```

```ts
pipe(
  Effect.succeed(100),
  Effect.flatMap((amount) => applyDiscount(amount, 5))
)
```

#### andThen

A more flexible variant of `flatMap`. The second argument can be an Effect, a function returning an Effect, a plain value, or a function returning a plain value. Also works with `Result` and `Option` types. Replaces `Effect.zipRight` from v3.

```ts
pipe(
  Effect.succeed(1),
  Effect.andThen((n) => n + 1),          // plain value function
  Effect.andThen(Effect.succeed("done")), // constant Effect
)
```

#### tap

Runs a side effect on the success value without changing it. Useful for logging/debugging. Replaces `Effect.zipLeft` from v3. Accepts either a function returning an `Effect` or a constant `Effect`.

```ts
const tap: {
  <A, B, E2, R2>(f: (a: A) => Effect<B, E2, R2>): <E, R>(self: Effect<A, E, R>) => Effect<A, E | E2, R | R2>
  <B, E2, R2>(f: Effect<B, E2, R2>): <A, E, R>(self: Effect<A, E, R>) => Effect<A, E | E2, R | R2>
  // data-first overloads omitted for brevity
}
```

#### Other Transformations

```ts
const as: <B>(value: B) => <A, E, R>(self: Effect<A, E, R>) => Effect<B, E, R>
const asVoid: <A, E, R>(self: Effect<A, E, R>) => Effect<void, E, R>
const asSome: <A, E, R>(self: Effect<A, E, R>) => Effect<Option<A>, E, R>
const flatten: <A, E, R, E2, R2>(self: Effect<Effect<A, E, R>, E2, R2>) => Effect<A, E | E2, R | R2>
const flip: <A, E, R>(self: Effect<A, E, R>) => Effect<E, A, R>   // swap success/error
const zip: { /* combines two effects into a tuple */ }
const zipWith: { /* combines two effects with a function */ }
const all: { /* collects an array/struct/iterable of effects */ }
```

### Error Handling

#### result (replaces Effect.either from v3)

Wraps both success and failure into a `Result<A, E>`, removing the error channel.

```ts
const result: <A, E, R>(self: Effect<A, E, R>) => Effect<Result<A, E>, never, R>
```

```ts
const program = Effect.result(Effect.fail("oops"))
// Effect<Result<never, string>, never, never>
```

#### exit

Wraps both success and failure into an `Exit<A, E>`, capturing the full `Cause` on failure.

```ts
const exit: <A, E, R>(self: Effect<A, E, R>) => Effect<Exit<A, E>, never, R>
```

Use `exit` when you need to inspect the full `Cause` (including defects and interruptions), not just typed errors. Use `result` when you only care about typed errors.

#### catch (replaces Effect.catchAll from v3)

Handles all recoverable errors by providing a fallback effect.

```ts
const catch: {
  <E, A2, E2, R2>(f: (e: E) => Effect<A2, E2, R2>): <A, R>(self: Effect<A, E, R>) => Effect<A | A2, E2, R | R2>
  <A, E, R, A2, E2, R2>(self: Effect<A, E, R>, f: (e: E) => Effect<A2, E2, R2>): Effect<A | A2, E2, R | R2>
}
```

#### catchTag / catchTags

Catches errors by their `_tag` discriminator field.

```ts
class NetworkError { readonly _tag = "NetworkError"; constructor(readonly message: string) {} }
class ValidationError { readonly _tag = "ValidationError"; constructor(readonly message: string) {} }

const program = pipe(
  task,
  Effect.catchTag("NetworkError", (e) => Effect.succeed(`Recovered: ${e.message}`))
)

// Or handle multiple tags at once:
const handled = Effect.catchTags(task, {
  NetworkError: (e) => Effect.succeed(`Network: ${e.message}`),
  ValidationError: (e) => Effect.succeed(`Validation: ${e.message}`)
})
```

#### catchCause / sandbox

`catchCause` handles the full `Cause` (including defects and interruptions). `sandbox` exposes the full `Cause<E>` as the error channel.

```ts
const catchCause: {
  <E, A2, E2, R2>(f: (cause: Cause<E>) => Effect<A2, E2, R2>): <A, R>(self: Effect<A, E, R>) => Effect<A | A2, E2, R | R2>
}

const sandbox: <A, E, R>(self: Effect<A, E, R>) => Effect<A, Cause<E>, R>
```

#### mapError / mapBoth

```ts
const mapError: {
  <E, E2>(f: (e: E) => E2): <A, R>(self: Effect<A, E, R>) => Effect<A, E2, R>
}
```

#### match

Pattern-match on both success and failure, producing a pure value.

```ts
const match: {
  <E, A2, A, A3>(options: {
    readonly onFailure: (error: E) => A2
    readonly onSuccess: (value: A) => A3
  }): <R>(self: Effect<A, E, R>) => Effect<A2 | A3, never, R>
}
```

#### Other Error Handling

```ts
const orDie: <A, E, R>(self: Effect<A, E, R>) => Effect<A, never, R>  // typed errors become defects
const eventually: <A, E, R>(self: Effect<A, E, R>) => Effect<A, never, R>  // retry until success
const retry: { /* retry with Schedule */ }
const ignore: { /* discard both success and failure */ }
const tapError: { /* side-effect on failure */ }
const tapCause: { /* side-effect on full Cause */ }
const catchDefect: { /* catch unrecoverable defects */ }
```

### Collecting & Iteration

#### all

Combines multiple effects from an array, tuple, struct, or iterable into a single effect. Supports concurrency control.

```ts
const all: <Arg extends Iterable<Effect<any, any, any>> | Record<string, Effect<any, any, any>>, O extends {
  readonly concurrency?: Concurrency
  readonly discard?: boolean
  readonly mode?: "default" | "result"
}>(arg: Arg, options?: O) => Effect<...>
```

```ts
import { Effect } from "effect"

// Tuple -- preserves types
const tuple = Effect.all([Effect.succeed(1), Effect.succeed("two")])
// Effect<[number, string]>

// Struct -- preserves keys
const struct = Effect.all({ x: Effect.succeed(1), y: Effect.succeed("two") })
// Effect<{ x: number; y: string }>

// Concurrent execution
const concurrent = Effect.all(
  [fetchUser(1), fetchUser(2), fetchUser(3)],
  { concurrency: "unbounded" }
)
```

#### forEach

Applies an effectful function to each element in an iterable, returning an array of results.

```ts
const forEach: {
  <A, B, E, R>(
    f: (a: A, i: number) => Effect<B, E, R>,
    options?: { readonly concurrency?: Concurrency; readonly discard?: boolean }
  ): (elements: Iterable<A>) => Effect<Array<B>, E, R>
}
```

```ts
const program = Effect.forEach(
  [1, 2, 3, 4, 5],
  (n) => Effect.succeed(n * 2),
  { concurrency: 2 }
)
// Effect<number[]> -- values: [2, 4, 6, 8, 10]
```

#### partition

Applies an effectful function to each element and separates failures from successes.

```ts
const partition: {
  <A, B, E, R>(
    f: (a: A, i: number) => Effect<B, E, R>,
    options?: { readonly concurrency?: Concurrency }
  ): (elements: Iterable<A>) => Effect<[excluded: Array<E>, satisfying: Array<B>], never, R>
}
```

```ts
const program = Effect.partition([0, 1, 2, 3], (n) =>
  n % 2 === 0 ? Effect.fail(`${n} is even`) : Effect.succeed(n)
)
Effect.runPromise(program).then(console.log)
// [ ["0 is even", "2 is even"], [1, 3] ]
```

### Delays & Timeouts

```ts
const sleep: (duration: Duration.Input) => Effect<void>
const delay: {
  (duration: Duration.Input): <A, E, R>(self: Effect<A, E, R>) => Effect<A, E, R>
}
const timeout: {
  (duration: Duration.Input): <A, E, R>(self: Effect<A, E, R>) => Effect<A, E | Cause.TimeoutError, R>
}
const timeoutOption: {
  (duration: Duration.Input): <A, E, R>(self: Effect<A, E, R>) => Effect<Option<A>, E, R>
}
const timeoutOrElse: {
  <A2, E2, R2>(options: {
    readonly duration: Duration.Input
    readonly onTimeout: () => Effect<A2, E2, R2>
  }): <A, E, R>(self: Effect<A, E, R>) => Effect<A | A2, E | E2, R | R2>
}
const timed: <A, E, R>(self: Effect<A, E, R>) => Effect<[duration: Duration, result: A], E, R>
```

```ts
import { Effect } from "effect"

// Wait 1 second
yield* Effect.sleep("1 second")

// Timeout after 5 seconds, failing with TimeoutError
const result = yield* pipe(
  longRunningTask,
  Effect.timeout("5 seconds")
)

// Timeout returning Option instead of failing
const maybeResult = yield* pipe(
  longRunningTask,
  Effect.timeoutOption("5 seconds")
)
```

### Concurrency & Racing

```ts
const race: {
  <A2, E2, R2>(that: Effect<A2, E2, R2>): <A, E, R>(self: Effect<A, E, R>) => Effect<A | A2, E | E2, R | R2>
}
const raceAll: <Eff extends Effect<any, any, any>>(effects: Iterable<Eff>) => Effect<...>
const raceFirst: {
  <A2, E2, R2>(that: Effect<A2, E2, R2>): <A, E, R>(self: Effect<A, E, R>) => Effect<A | A2, E | E2, R | R2>
}
const withConcurrency: {
  (concurrency: Concurrency): <A, E, R>(self: Effect<A, E, R>) => Effect<A, E, R>
}
```

`race` returns whichever effect succeeds first (loser is interrupted). `raceFirst` returns whichever effect completes first, even if it fails. `withConcurrency` sets the default concurrency for combinators like `all` and `forEach`.

### Running Effects

```ts
// Async execution (effect must have R = never; provide services first if needed)
const runPromise: <A, E>(effect: Effect<A, E>, options?: RunOptions) => Promise<A>
const runPromiseExit: <A, E>(effect: Effect<A, E>, options?: RunOptions) => Promise<Exit<A, E>>
const runFork: <A, E>(effect: Effect<A, E, never>, options?: RunOptions) => Fiber<A, E>
const runCallback: <A, E>(
  effect: Effect<A, E, never>,
  options?: RunOptions & { readonly onExit: (exit: Exit<A, E>) => void }
) => (interruptor?: number) => void

// Sync execution (throws on async effects or on failure)
const runSync: <A, E>(effect: Effect<A, E>) => A
const runSyncExit: <A, E>(effect: Effect<A, E>) => Exit<A, E>

// With provided services (new in v4) — curried: provide services first, then run
const runPromiseWith: <R>(services: ServiceMap<R>) => <A, E>(effect: Effect<A, E, R>, options?: RunOptions) => Promise<A>
const runSyncWith: <R>(services: ServiceMap<R>) => <A, E>(effect: Effect<A, E, R>) => A
const runForkWith: <R>(services: ServiceMap<R>) => <A, E>(effect: Effect<A, E, R>, options?: RunOptions) => Fiber<A, E>

interface RunOptions {
  readonly signal?: AbortSignal
  readonly scheduler?: Scheduler
  readonly uninterruptible?: boolean
}
```

```ts
// Basic usage
Effect.runPromise(Effect.succeed(42)).then(console.log) // 42
Effect.runSync(Effect.succeed(42)) // 42

// With services
const services = ServiceMap.make(MyService, { ... })
Effect.runPromiseWith(services)(myProgram)
```

### Resource Management

```ts
const acquireRelease: <A, E, R>(
  acquire: Effect<A, E, R>,
  release: (a: A, exit: Exit<unknown, unknown>) => Effect<unknown>
) => Effect<A, E, R | Scope>

const scoped: <A, E, R>(self: Effect<A, E, R>) => Effect<A, E, Exclude<R, Scope>>

// addFinalizer registers a cleanup that receives the scope's Exit value
const addFinalizer: <R>(
  finalizer: (exit: Exit<unknown, unknown>) => Effect<void, never, R>
) => Effect<void, never, Scope | R>
```

```ts
const resource = Effect.acquireRelease(
  Effect.succeed({ handle: "file.txt" }),
  (handle) => Effect.sync(() => console.log(`Closing ${handle.handle}`))
)

const program = Effect.scoped(
  Effect.gen(function*() {
    const file = yield* resource
    return file.handle
  })
)
```

---

## Result\<A, E\>

### Overview

A synchronous, pure type for representing computations that can succeed (`Success<A>`) or fail (`Failure<E>`). Unlike `Effect`, `Result` is evaluated eagerly and carries no side effects. `Result` replaces `Either` from Effect 3.x.

- `Result<A, E>` is a discriminated union: `Success<A, E> | Failure<A, E>`
- `Success` wraps a value accessed via `.success`
- `Failure` wraps an error accessed via `.failure`
- `E` defaults to `never`, so `Result<number>` means a result that cannot fail
- `Result` is yieldable in `Effect.gen`, producing the inner value or short-circuiting on failure

### Key Types

```ts
type Result<A, E = never> = Success<A, E> | Failure<A, E>

interface Success<out A, out E> extends Pipeable, Inspectable, Yieldable<Result<A, E>, A, E> {
  readonly _tag: "Success"
  readonly _op: "Success"
  readonly success: A
}

interface Failure<out A, out E> extends Pipeable, Inspectable, Yieldable<Result<A, E>, A, E> {
  readonly _tag: "Failure"
  readonly _op: "Failure"
  readonly failure: E
}

// Type-level extractors
type Result.Success<T extends Result<any, any>>  // extracts A from Result<A, E>
type Result.Failure<T extends Result<any, any>>  // extracts E from Result<A, E>
```

### Constructors

```ts
const succeed: <A>(value: A) => Result<A>           // replaces Either.right
const fail: <E>(error: E) => Result<never, E>       // replaces Either.left
const void: Result<void>                             // pre-built success with undefined
const failVoid: Result<never, void>                  // pre-built failure with undefined
const fromNullishOr: {
  <A, E>(onNullish: (a: A) => E): (self: A) => Result<NonNullable<A>, E>
  <A, E>(self: A, onNullish: (a: A) => E): Result<NonNullable<A>, E>
}
const fromOption: {
  <E>(onNone: () => E): <A>(self: Option<A>) => Result<A, E>
  <A, E>(self: Option<A>, onNone: () => E): Result<A, E>
}
const try: {
  <A, E>(options: { readonly try: () => A; readonly catch: (error: unknown) => E }): Result<A, E>
  <A>(evaluate: () => A): Result<A, unknown>
}
const liftPredicate: {
  <A, B extends A, E>(refinement: Refinement<A, B>, orFailWith: (a: A) => E): (a: A) => Result<B, E>
  <A, E>(predicate: Predicate<A>, orFailWith: (a: A) => E): (a: A) => Result<A, E>
}
```

### Transformations & Sequencing

```ts
const map: {
  <A, A2>(f: (ok: A) => A2): <E>(self: Result<A, E>) => Result<A2, E>
  <A, E, A2>(self: Result<A, E>, f: (ok: A) => A2): Result<A2, E>
}
const mapError: {
  <E, E2>(f: (err: E) => E2): <A>(self: Result<A, E>) => Result<A, E2>
}
const mapBoth: {
  <E, E2, A, A2>(options: { onFailure: (e: E) => E2; onSuccess: (a: A) => A2 }): (self: Result<A, E>) => Result<A2, E2>
}
const flatMap: {
  <A, A2, E2>(f: (a: A) => Result<A2, E2>): <E>(self: Result<A, E>) => Result<A2, E | E2>
}
const andThen: { /* accepts Result, function=>Result, value, or function=>value */ }
const tap: {
  <A>(f: (a: A) => void): <E>(self: Result<A, E>) => Result<A, E>
}
const all: <I extends Iterable<Result<any, any>> | Record<string, Result<any, any>>>(input: I) => Result<...>
const flip: <A, E>(self: Result<A, E>) => Result<E, A>
```

### Pattern Matching & Getters

```ts
const match: {
  <E, B, A, C>(options: { onFailure: (e: E) => B; onSuccess: (a: A) => C }): (self: Result<A, E>) => B | C
}
const getOrElse: {
  <E, A2>(onFailure: (err: E) => A2): <A>(self: Result<A, E>) => A | A2
}
const getOrNull: <A, E>(self: Result<A, E>) => A | null
const getOrUndefined: <A, E>(self: Result<A, E>) => A | undefined
const getOrThrow: <A, E>(self: Result<A, E>) => A        // throws E directly
const getOrThrowWith: {
  <E>(onFailure: (err: E) => unknown): <A>(self: Result<A, E>) => A
}
const merge: <A, E>(self: Result<A, E>) => A | E
const getSuccess: <A, E>(self: Result<A, E>) => Option<A>
const getFailure: <A, E>(self: Result<A, E>) => Option<E>
```

### Type Guards

```ts
const isResult: (input: unknown) => input is Result<unknown, unknown>
const isSuccess: <A, E>(self: Result<A, E>) => self is Success<A, E>
const isFailure: <A, E>(self: Result<A, E>) => self is Failure<A, E>
```

### Error Handling

```ts
const orElse: {
  <E, A2, E2>(that: (err: E) => Result<A2, E2>): <A>(self: Result<A, E>) => Result<A | A2, E2>
}
const filterOrFail: {
  <A, E2>(predicate: Predicate<A>, orFailWith: (a: A) => E2): <E>(self: Result<A, E>) => Result<A, E | E2>
}
```

### Generator Syntax

```ts
const gen: (f: () => Generator) => Result

const result = Result.gen(function*() {
  const a = yield* Result.succeed(1)
  const b = yield* Result.succeed(2)
  return a + b
})
// Result<number> -- { _tag: "Success", success: 3 }
```

### Do Notation

An alternative to `gen` for building up a record of named fields sequentially. Each `bind` step runs a `Result`-producing function and short-circuits on `Failure`.

```ts
const Do: Result<{}>
const bind: {
  <N extends string, A extends object, B, E2>(
    name: N,
    f: (a: A) => Result<B, E2>
  ): <E>(self: Result<A, E>) => Result<A & Record<N, B>, E | E2>
}
const bindTo: {
  <N extends string>(name: N): <A, E>(self: Result<A, E>) => Result<Record<N, A>, E>
}
// `let` adds a pure (non-Result) field
```

```ts
import { pipe, Result } from "effect"

const validated = pipe(
  Result.Do,
  Result.bind("name", () => validateName("Alice")),
  Result.bind("age", () => validateAge(30))
)
// Result<{ name: string; age: number }> on success
```

### Gotchas

- `E` defaults to `never`, so `Result<number>` means a result that cannot fail
- `andThen` accepts a `Result`, a function returning a `Result`, a plain value, or a function returning a plain value; `flatMap` only accepts a function returning a `Result`
- `all` short-circuits on the first `Failure` -- later elements are not inspected
- `getOrThrow` throws the raw failure value `E`; use `getOrThrowWith` for custom error objects
- `tap` runs a side-effect but does not change the result; its return value is ignored

---

## Exit\<A, E\>

### Overview

Represents the outcome of an Effect computation as a plain, synchronously inspectable value. `Exit<A, E>` is a union of `Success<A, E>` (wrapping a value `A`) and `Failure<A, E>` (wrapping a `Cause<E>`). Exit is also an Effect, so you can yield it inside `Effect.gen`.

A `Failure` wraps a `Cause<E>`, not a bare `E`. Use Cause utilities to drill into it.

### Key Types

```ts
type Exit<A, E = never> = Success<A, E> | Failure<A, E>

interface Success<out A, out E = never> extends Effect<A, E> {
  readonly _tag: "Success"
  readonly value: A
}

interface Failure<out A, out E> extends Effect<A, E> {
  readonly _tag: "Failure"
  readonly cause: Cause<E>
}
```

### Constructors

```ts
const succeed: <A>(a: A) => Exit<A>
const fail: <E>(e: E) => Exit<never, E>
const failCause: <E>(cause: Cause<E>) => Exit<never, E>
const die: (defect: unknown) => Exit<never>
const interrupt: (fiberId?: number) => Exit<never>
const void: Exit<void>
```

### Guards & Inspection

```ts
const isExit: (u: unknown) => u is Exit<unknown, unknown>
const isSuccess: <A, E>(self: Exit<A, E>) => self is Success<A, E>
const isFailure: <A, E>(self: Exit<A, E>) => self is Failure<A, E>
const hasFails: <A, E>(self: Exit<A, E>) => self is Failure<A, E>     // has typed errors?
const hasDies: <A, E>(self: Exit<A, E>) => self is Failure<A, E>      // has defects?
const hasInterrupts: <A, E>(self: Exit<A, E>) => self is Failure<A, E> // was interrupted?
```

### Pattern Matching & Accessors

```ts
const match: {
  <A, E, X1, X2>(options: {
    readonly onSuccess: (a: A) => X1
    readonly onFailure: (cause: Cause<E>) => X2
  }): (self: Exit<A, E>) => X1 | X2
}

const getSuccess: <A, E>(self: Exit<A, E>) => Option<A>
const getCause: <A, E>(self: Exit<A, E>) => Option<Cause<E>>
const findErrorOption: <A, E>(self: Exit<A, E>) => Option<E>
```

### Combinators

```ts
const map: { <A, B>(f: (a: A) => B): <E>(self: Exit<A, E>) => Exit<B, E> }
const mapError: { <E, E2>(f: (e: E) => E2): <A>(self: Exit<A, E>) => Exit<A, E2> }
const mapBoth: { /* transform both channels */ }
const asVoid: <A, E>(self: Exit<A, E>) => Exit<void, E>
const asVoidAll: <I extends Iterable<Exit<any, any>>>(exits: I) => Exit<void, /* E inferred from I */>
```

---

## Cause\<E\>

### Overview

Structured representation of how an Effect can fail. A `Cause<E>` holds a flat array of `Reason` values in its `.reasons` property. Each reason is one of:

- **Fail\<E\>** -- a typed, expected error (created by `Effect.fail`), accessed via `.error`
- **Die** -- an untyped defect (`unknown`) from `Effect.die` or uncaught throws, accessed via `.defect`
- **Interrupt** -- a fiber interruption, optionally carrying the interrupting fiber's ID via `.fiberId`

Causes are always flat: concurrent and sequential failures are stored together. An empty `reasons` array means the cause is empty. `Cause` implements `Equal`.

### Key Types

```ts
interface Cause<out E> extends Pipeable, Inspectable, Equal {
  readonly reasons: ReadonlyArray<Reason<E>>
}

type Reason<E> = Fail<E> | Die | Interrupt

// Each Reason also extends ReasonProto which adds:
//   annotations: ReadonlyMap<string, unknown>  -- tracing metadata
//   annotate(annotations, options?): this       -- attach metadata
interface Fail<out E> { readonly _tag: "Fail"; readonly error: E }
interface Die { readonly _tag: "Die"; readonly defect: unknown }
interface Interrupt { readonly _tag: "Interrupt"; readonly fiberId: number | undefined }
```

### Constructors

```ts
// Cause constructors (each wraps one reason in a Cause)
const fail: <E>(error: E) => Cause<E>
const die: (defect: unknown) => Cause<never>
const interrupt: (fiberId?: number) => Cause<never>
const empty: Cause<never>
const fromReasons: <E>(reasons: ReadonlyArray<Reason<E>>) => Cause<E>
const combine: {
  <E2>(that: Cause<E2>): <E>(self: Cause<E>) => Cause<E | E2>
  <E, E2>(self: Cause<E>, that: Cause<E2>): Cause<E | E2>
}

// Standalone Reason constructors (not wrapped in a Cause; for use with fromReasons)
const makeFailReason: <E>(error: E) => Fail<E>
const makeDieReason: (defect: unknown) => Die
const makeInterruptReason: (fiberId?: number) => Interrupt
```

### Predicates & Extractors

```ts
const hasFails: <E>(self: Cause<E>) => boolean
const hasDies: <E>(self: Cause<E>) => boolean
const hasInterrupts: <E>(self: Cause<E>) => boolean
const hasInterruptsOnly: <E>(self: Cause<E>) => boolean

// Type guards for individual reasons
const isFailReason: <E>(self: Reason<E>) => self is Fail<E>
const isDieReason: <E>(self: Reason<E>) => self is Die
const isInterruptReason: <E>(self: Reason<E>) => self is Interrupt

// Extract first matching value (returns Result where Failure = not found)
const findError: <E>(self: Cause<E>) => Result<E, Cause<never>>         // first typed error E
const findFail: <E>(self: Cause<E>) => Result<Fail<E>, Cause<never>>    // first Fail reason (with annotations)
const findDefect: <E>(self: Cause<E>) => Result<unknown, Cause<E>>      // first defect value
const findDie: <E>(self: Cause<E>) => Result<Die, Cause<E>>             // first Die reason (with annotations)
const findErrorOption: <E>(self: Cause<E>) => Option<E>                  // Option-based variant of findError
```

### Transformations

```ts
// Transform typed errors; Die and Interrupt reasons pass through unchanged
const map: {
  <E, E2>(f: (error: E) => E2): (self: Cause<E>) => Cause<E2>
  <E, E2>(self: Cause<E>, f: (error: E) => E2): Cause<E2>
}
```

### Rendering

```ts
const squash: <E>(self: Cause<E>) => unknown       // lossy: first Fail, then Die, then generic
const pretty: <E>(cause: Cause<E>) => string        // human-readable string
const prettyErrors: <E>(self: Cause<E>) => Array<Error>  // each reason as an Error
```

### Iterating Reasons

```ts
import { Cause } from "effect"

const cause = Cause.fail("error")
const errors = cause.reasons.filter(Cause.isFailReason).map((r) => r.error)
const defects = cause.reasons.filter(Cause.isDieReason).map((r) => r.defect)
```

### Built-in Error Classes

All implement `YieldableError` and can be yielded directly in `Effect.gen`:

```ts
class NoSuchElementError       // thrown by Array.head, Option.getOrThrow, etc.
class TimeoutError             // produced by Effect.timeout
class IllegalArgumentError     // contract violations
class ExceededCapacityError    // queue/pool overflow
class UnknownError             // wraps non-Error thrown values
```

### Gotchas

- `squash` is lossy -- use `prettyErrors` or iterate `reasons` when you need all failures
- `findError`/`findDefect` return a `Result` where the **Failure case** (`Result.fail(cause)`) means no match was found (not `Option.none()`); use `findErrorOption` if you need an `Option`
- `findError` returns `Result<E, Cause<never>>` -- check `Result.isFailure(result)` to know if no typed error was found
- `findFail` and `findDie` are lower-level variants that return the full `Fail`/`Die` reason (including annotations) rather than the bare value

---

## Scope

### Overview

The `Scope` module provides resource lifecycle management. A `Scope` represents a context where resources can be acquired and automatically cleaned up when the scope is closed. Scopes support both sequential and parallel finalization strategies. Finalizers run in reverse order of registration.

### Key Types

```ts
interface Scope {
  readonly strategy: "sequential" | "parallel"
  state: State.Open | State.Closed | State.Empty
}

interface Closeable extends Scope { }

// States
namespace State {
  type Empty = { readonly _tag: "Empty" }
  type Open = { readonly _tag: "Open"; readonly finalizers: Map<{}, (exit: Exit<any, any>) => Effect<void>> }
  type Closed = { readonly _tag: "Closed"; readonly exit: Exit<any, any> }
}
```

### Constructors & Operations

```ts
const Scope: ServiceMap.Service<Scope, Scope>  // service tag for dependency injection
const make: (finalizerStrategy?: "sequential" | "parallel") => Effect<Closeable>
const makeUnsafe: (finalizerStrategy?: "sequential" | "parallel") => Closeable

const addFinalizer: (scope: Scope, finalizer: Effect<unknown>) => Effect<void>
const addFinalizerExit: (scope: Scope, finalizer: (exit: Exit<any, any>) => Effect<unknown>) => Effect<void>
const close: (self: Scope, exit: Exit<any, any>) => Effect<void>
const fork: (scope: Scope, finalizerStrategy?: "sequential" | "parallel") => Effect<Closeable>
const forkUnsafe: (scope: Scope, finalizerStrategy?: "sequential" | "parallel") => Closeable
const provide: {
  (value: Scope): <A, E, R>(self: Effect<A, E, R>) => Effect<A, E, Exclude<R, Scope>>
  <A, E, R>(self: Effect<A, E, R>, value: Scope): Effect<A, E, Exclude<R, Scope>>
}
const use: {
  (scope: Closeable): <A, E, R>(self: Effect<A, E, R>) => Effect<A, E, Exclude<R, Scope>>
  <A, E, R>(self: Effect<A, E, R>, scope: Closeable): Effect<A, E, Exclude<R, Scope>>
}
```

### Typical Usage

Scopes are usually used indirectly through `Effect.acquireRelease` and `Effect.scoped`:

```ts
import { Console, Effect } from "effect"

const resource = Effect.acquireRelease(
  Console.log("Acquiring").pipe(Effect.as("resource")),
  () => Console.log("Releasing")
)

const program = Effect.scoped(
  Effect.gen(function*() {
    const res = yield* resource
    yield* Console.log(`Using ${res}`)
    return res
  })
)

Effect.runFork(program)
// Output: Acquiring
// Output: Using resource
// Output: Releasing
```

---

## Comprehensive Examples

### Full Program with Error Handling

```ts
import { Console, Effect, Result } from "effect"

class HttpError {
  readonly _tag = "HttpError"
  constructor(readonly status: number, readonly message: string) {}
}

class ParseError {
  readonly _tag = "ParseError"
  constructor(readonly input: string) {}
}

const fetchUser = (id: number) =>
  Effect.tryPromise({
    try: () => fetch(`/api/users/${id}`).then((r) => r.json()),
    catch: () => new HttpError(500, "Network failure")
  })

const parseAge = (data: unknown): Effect.Effect<number, ParseError> => {
  const age = (data as any)?.age
  return typeof age === "number"
    ? Effect.succeed(age)
    : Effect.fail(new ParseError(JSON.stringify(data)))
}

const program = Effect.gen(function*() {
  const data = yield* fetchUser(1)
  const age = yield* parseAge(data)
  yield* Console.log(`User age: ${age}`)
  return age
}).pipe(
  Effect.catchTags({
    HttpError: (e) => Effect.succeed(-1),
    ParseError: (e) => Effect.succeed(0)
  })
)

Effect.runPromise(program).then(console.log)
```

### Resource Management Pattern

```ts
import { Console, Effect, Exit, Scope } from "effect"

// A managed database connection
const makeDbConnection = Effect.acquireRelease(
  Effect.gen(function*() {
    yield* Console.log("Opening database connection")
    return { query: (sql: string) => Effect.succeed(`Result: ${sql}`) }
  }),
  (db, exit) =>
    Console.log(
      `Closing connection (${Exit.isSuccess(exit) ? "success" : "failure"})`
    )
)

// A managed file handle
const makeFileHandle = (path: string) =>
  Effect.acquireRelease(
    Effect.gen(function*() {
      yield* Console.log(`Opening file: ${path}`)
      return { path, write: (data: string) => Effect.succeed(data.length) }
    }),
    (handle) => Console.log(`Closing file: ${handle.path}`)
  )

// Compose resources -- both are cleaned up when the scope closes
const program = Effect.scoped(
  Effect.gen(function*() {
    const db = yield* makeDbConnection
    const file = yield* makeFileHandle("/tmp/output.csv")
    const data = yield* db.query("SELECT * FROM users")
    const bytes = yield* file.write(data)
    yield* Console.log(`Wrote ${bytes} bytes`)
    return bytes
  })
)
// Closing file: /tmp/output.csv
// Closing connection (success)
```

### Using Result for Synchronous Validation

```ts
import { pipe, Result } from "effect"

interface User {
  name: string
  email: string
  age: number
}

const validateName = (name: string): Result.Result<string, string> =>
  name.length >= 2 ? Result.succeed(name) : Result.fail("Name too short")

const validateEmail = (email: string): Result.Result<string, string> =>
  email.includes("@") ? Result.succeed(email) : Result.fail("Invalid email")

const validateAge = (age: number): Result.Result<number, string> =>
  age >= 0 && age <= 150 ? Result.succeed(age) : Result.fail("Invalid age")

// Using gen for sequential validation
const validateUser = (input: { name: string; email: string; age: number }) =>
  Result.gen(function*() {
    const name = yield* validateName(input.name)
    const email = yield* validateEmail(input.email)
    const age = yield* validateAge(input.age)
    return { name, email, age } satisfies User
  })

// Using Do notation
const validateUserDo = (input: { name: string; email: string; age: number }) =>
  pipe(
    Result.Do,
    Result.bind("name", () => validateName(input.name)),
    Result.bind("email", () => validateEmail(input.email)),
    Result.bind("age", () => validateAge(input.age))
  )

console.log(validateUser({ name: "Alice", email: "alice@example.com", age: 30 }))
// { _tag: "Success", success: { name: "Alice", email: "alice@example.com", age: 30 } }

console.log(validateUser({ name: "A", email: "alice@example.com", age: 30 }))
// { _tag: "Failure", failure: "Name too short" }
```

### Inspecting Exit and Cause

```ts
import { Cause, Effect, Exit } from "effect"

const program = Effect.gen(function*() {
  // Run an effect and inspect the full Exit
  const exit = yield* Effect.exit(
    Effect.fail(new Error("something broke"))
  )

  if (Exit.isFailure(exit)) {
    const cause = exit.cause

    // Check what kind of failure occurred
    if (Cause.hasFails(cause)) {
      const errors = cause.reasons
        .filter(Cause.isFailReason)
        .map((r) => r.error)
      console.log("Typed errors:", errors)
    }

    if (Cause.hasDies(cause)) {
      const defects = cause.reasons
        .filter(Cause.isDieReason)
        .map((r) => r.defect)
      console.log("Defects:", defects)
    }

    // Pretty-print the cause for debugging
    console.log(Cause.pretty(cause))
  }
})
```

### Dual Calling Convention

Most Effect combinators support both data-first and data-last (pipe) styles:

```ts
import { Effect, pipe } from "effect"

// Data-first (direct call)
const a = Effect.map(Effect.succeed(1), (n) => n + 1)

// Data-last (pipe)
const b = pipe(
  Effect.succeed(1),
  Effect.map((n) => n + 1)
)

// Method-style (.pipe)
const c = Effect.succeed(1).pipe(
  Effect.map((n) => n + 1)
)

// All three produce the same Effect<number>
```

### Key Migration Notes from v3

| Effect 3.x            | Effect 4.0           |
|------------------------|----------------------|
| `Either<E, A>`         | `Result<A, E>`       |
| `Either.right(a)`      | `Result.succeed(a)`  |
| `Either.left(e)`       | `Result.fail(e)`     |
| `Effect.either`        | `Effect.result`      |
| `Effect.catchAll`      | `Effect.catch`       |
| `Effect.zipRight`      | `Effect.andThen`     |
| `Effect.zipLeft`       | `Effect.tap`         |
| `Scope.extend`         | `Scope.provide`      |
| `Cause.Sequential/Parallel` | `Cause.reasons` (flat array) |

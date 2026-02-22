# Streaming (Stream, Sink, Channel, Pull)

Effect 4.0 provides a complete pull-based streaming system built on four
interconnected modules. Streams are approximately **20x faster** in v4 thanks to
a redesigned internal architecture that eliminates the old continuation-based
channel interpreter in favor of direct pull functions.

## Architecture Overview

```
Pull  -->  Channel  -->  Stream / Sink
(low-level)  (foundation)   (high-level)
```

- **Pull** -- a low-level `Effect` that yields values, fails, or signals
  completion with `Cause.Done`. This is the primitive everything else compiles
  down to.
- **Channel** -- bidirectional I/O primitive. Both Stream and Sink are
  implemented on top of channels.
- **Stream** -- a lazy, potentially infinite sequence of values with
  back-pressure. Emits chunks (non-empty arrays) to amortize effect overhead.
- **Sink** -- a stream consumer that folds/reduces a stream into a single result.

---

## Stream

```ts
// effect/Stream
interface Stream<out A, out E = never, out R = never>
```

A `Stream<A, E, R>` describes a program that can emit zero or more `A` values,
fail with `E`, and require services `R`. Streams are pull-based with
back-pressure and emit chunks (`NonEmptyReadonlyArray<A>`) to amortize effect
evaluation. They support monadic composition and error handling similar to
`Effect`, adapted for multiple values.

### Constructors

#### make / succeed / empty / never

```ts
// Create from explicit values
const s = Stream.make(1, 2, 3)

// Single value
const one = Stream.succeed(42)

// Empty stream (completes immediately)
const none = Stream.empty // Stream<never>

// Never-ending stream (never emits, never fails)
const inf = Stream.never // Stream<never>
```

#### fromIterable / fromArray

```ts
const fromList = Stream.fromIterable([1, 2, 3])
const fromArr = Stream.fromArray([1, 2, 3])

// Multiple arrays concatenated
const multi = Stream.fromArrays([1, 2], [3, 4])
// emits: 1, 2, 3, 4
```

#### fromEffect / fromEffectRepeat / fromEffectSchedule / fromEffectDrain

```ts
// Single element from an effect
const single = Stream.fromEffect(Effect.succeed(42))
// emits: 42, then ends

// Run an effect for its side effects only -- emits no elements
const drain = Stream.fromEffectDrain(Console.log("hello"))
// emits nothing, then ends

// Repeat an effect forever (use take/takeUntil to stop)
const repeated = Stream.fromEffectRepeat(Random.nextInt).pipe(
  Stream.take(5)
)
// Note: fromEffectRepeat replaces the v3 Stream.repeatEffect

// Repeat an effect on a schedule (replaces v3 Stream.repeatEffectWithSchedule)
const scheduled = Stream.fromEffectSchedule(
  Effect.succeed("ping"),
  Schedule.recurs(2)
)
// emits: "ping", "ping", "ping" (initial + 2 recurrences)
```

#### unfold / iterate / range

> **v4 note:** `Stream.unfold` is always effectful -- the callback returns `Effect<[A, S] | undefined>`. There is no separate `unfoldEffect`. Return `undefined` to signal the end of the stream.

```ts
// unfold signature: (initial: S, f: (s: S) => Effect<[A, S] | undefined, E, R>) => Stream<A, E, R>
const naturals = Stream.unfold(1, (n) =>
  Effect.succeed([n, n + 1] as const)
)

// Paginated API example -- use Stream.paginate when each step yields an array
// of items plus an optional next-page cursor.
// paginate emits each item from the array as individual stream elements.
// Return Option.none() when there is no next page.
const pages = Stream.paginate(null as string | null, (cursor) =>
  fetchPage(cursor).pipe(
    Effect.map((res) => [
      res.items,  // ReadonlyArray<Item> -- each item is emitted individually
      res.nextCursor !== null
        ? Option.some(res.nextCursor)
        : Option.none()
    ] as const)
  )
)
// pages: Stream<Item>  -- no extra flattening needed

// unfold: each step emits exactly ONE A.  Return undefined to stop.
const bounded = Stream.unfold(1, (n) =>
  n > 5
    ? Effect.succeed(undefined)
    : Effect.succeed([n, n + 1] as const)
)
// bounded emits: 1, 2, 3, 4, 5

// iterate: simpler when the value IS the state
const counting = Stream.iterate(1, (n) => n + 1).pipe(Stream.take(5))

// range: inclusive integer range
const r = Stream.range(1, 100)
// emits: 1, 2, 3, ..., 100
```

#### callback (replaces v3 async/asyncEffect/asyncPush)

```ts
const s = Stream.callback<number>((queue) =>
  Effect.sync(() => {
    Queue.offerUnsafe(queue, 1)
    Queue.offerUnsafe(queue, 2)
    Queue.offerUnsafe(queue, 3)
    Queue.endUnsafe(queue) // signal completion
  })
)
```

Accepts an optional second argument for `bufferSize` and `strategy`
(`"sliding"`, `"dropping"`, or `"suspend"`).

#### fromAsyncIterable / fromReadableStream / fromQueue / fromPubSub

These constructors bridge JavaScript ecosystem types into Effect streams.

#### tick

```ts
// Emit void at regular intervals
const heartbeat = Stream.tick("200 millis").pipe(Stream.take(3))
```

#### repeat (stream-level)

```ts
// Repeat the entire stream on a schedule
// Schedule.recurs(4) repeats 4 additional times after the first run (5 total)
const pingStream = Stream.make("ping").pipe(
  Stream.repeat(Schedule.recurs(4))
)
// emits: "ping", "ping", "ping", "ping", "ping"

// Collect to an array:
const result: Effect<Array<string>> = Stream.runCollect(pingStream)
// Effect resolves to: ["ping", "ping", "ping", "ping", "ping"]
```

### Transformations

#### map

```ts
// Receives element AND index
Stream.make(1, 2, 3).pipe(
  Stream.map((n, i) => `${i}:${n}`)
)
// "0:1", "1:2", "2:3"
```

#### mapEffect

```ts
// Effectful mapping with optional concurrency
Stream.make(1, 2, 3).pipe(
  Stream.mapEffect(
    (n) => Effect.succeed(n * 2),
    { concurrency: "unbounded", unordered: true }
  )
)
```

#### flatMap

```ts
// Monadic bind -- each element expands into a sub-stream
Stream.make(1, 2, 3).pipe(
  Stream.flatMap((n) => Stream.make(n, n * 10))
)
// 1, 10, 2, 20, 3, 30

// With concurrency for parallel inner streams
Stream.make(1, 2, 3).pipe(
  Stream.flatMap((n) => fetchItems(n), { concurrency: 4 })
)
```

#### switchMap

```ts
// Switch to the latest inner stream, cancelling previous ones
Stream.make(1, 2, 3).pipe(
  Stream.switchMap((n) =>
    n === 3 ? Stream.make(n) : Stream.never
  )
)
// emits: 3
```

#### flatten

```ts
// Flatten a stream of streams (sequential by default)
const nested = Stream.make(
  Stream.make(1, 2),
  Stream.make(3, 4)
)
Stream.flatten(nested)
// 1, 2, 3, 4

// Flatten with concurrency
Stream.flatten(nested, { concurrency: "unbounded" })
```

#### tap

```ts
Stream.make(1, 2, 3).pipe(
  Stream.tap((n) => Console.log(`saw: ${n}`))
)
```

#### mapAccum / scan

```ts
// scan: emits the initial value, then each accumulated state after each element
Stream.make(1, 2, 3, 4).pipe(
  Stream.scan(0, (acc, n) => acc + n)
)
// emits: 0, 1, 3, 6, 10  (initial + one per element)
```

#### concat / merge / zip

```ts
// Sequential concatenation
Stream.concat(Stream.make(1, 2), Stream.make(3, 4))
// 1, 2, 3, 4

// Concurrent merge -- elements from both as they arrive
Stream.merge(streamA, streamB)
// merge accepts { haltStrategy: "left" | "right" | "both" | "either" }

// Pairwise zip
Stream.zip(Stream.make(1, 2), Stream.make("a", "b"))
// [1, "a"], [2, "b"]
```

### Filtering

#### filter

```ts
Stream.make(1, 2, 3, 4).pipe(
  Stream.filter((n) => n % 2 === 0)
)
// 2, 4
```

Also supports `Refinement` for type narrowing and `Filter` objects from the
`Filter` module.

#### take / drop

```ts
Stream.range(1, 100).pipe(Stream.take(5))
// 1, 2, 3, 4, 5

Stream.range(1, 100).pipe(Stream.drop(95))
// 96, 97, 98, 99, 100
```

#### takeUntil / takeWhile / dropUntil / dropWhile

```ts
Stream.range(1, 10).pipe(
  Stream.takeUntil((n) => n === 5)
)
// 1, 2, 3, 4, 5

Stream.range(1, 10).pipe(
  Stream.takeUntil((n) => n === 5, { excludeLast: true })
)
// 1, 2, 3, 4

Stream.range(1, 10).pipe(
  Stream.takeWhile((n) => n < 4)
)
// 1, 2, 3
```

### Chunking / Grouping

```ts
// Expose chunks as stream elements
Stream.make(1, 2, 3, 4).pipe(
  Stream.rechunk(2),
  Stream.chunks
)
// [[1, 2], [3, 4]]

// Batch by count
Stream.range(1, 10).pipe(Stream.grouped(3))
// [[1,2,3], [4,5,6], [7,8,9], [10]]

// Batch by count or time window
Stream.range(1, 10).pipe(
  Stream.groupedWithin(3, "100 millis")
)
```

### Rate Limiting

```ts
// Debounce: emit only after quiet period
stream.pipe(Stream.debounce("200 millis"))

// Throttle: shape throughput
// cost receives the entire chunk; units is the bucket size
stream.pipe(
  Stream.throttle({ cost: (chunk) => chunk.length, units: 10, duration: "1 second" })
)

// Space elements with a schedule
Stream.make(1, 2, 3).pipe(
  Stream.schedule(Schedule.spaced("10 millis"))
)
```

### Error Handling

#### catch (replaces v3 catchAll)

```ts
Stream.make(1, 2).pipe(
  Stream.concat(Stream.fail("Oops!")),
  Stream.catch(() => Stream.make(999))
)
// 1, 2, 999
```

#### catchTag

```ts
class HttpError extends Data.TaggedError("HttpError")<{
  message: string
}> {}

Stream.fail(new HttpError({ message: "timeout" })).pipe(
  Stream.catchTag("HttpError", (err) =>
    Stream.make(`Recovered: ${err.message}`)
  )
)
```

#### catchIf (replaces v3 catchSome)

```ts
stream.pipe(
  Stream.catchIf(
    (error): error is TimeoutError => error._tag === "TimeoutError",
    () => Stream.make("fallback")
  )
)
```

#### tapError / tapCause

```ts
stream.pipe(
  Stream.tapError((error) => Console.log(`Error: ${error}`)),
  Stream.catch(() => Stream.empty)
)
```

### Consumers (Destructors)

All consumers return `Effect` values.

#### runCollect

```ts
const arr: Effect<Array<number>> = Stream.make(1, 2, 3).pipe(
  Stream.runCollect
)
// [1, 2, 3]
```

#### runForEach

```ts
Stream.make(1, 2, 3).pipe(
  Stream.runForEach((n) => Console.log(`Processing: ${n}`))
)
```

#### runFold

```ts
Stream.make(1, 2, 3).pipe(
  Stream.runFold(() => 0, (acc, n) => acc + n)
)
// 6
```

#### runDrain

```ts
// Run for side effects, discard elements
Stream.make(1, 2, 3).pipe(
  Stream.mapEffect((n) => Console.log(n)),
  Stream.runDrain
)
```

#### runHead / runLast / runCount / runSum

```ts
Stream.runHead(Stream.make(1, 2, 3)) // Effect<Option<number>>  => Some(1)
Stream.runLast(Stream.make(1, 2, 3)) // Effect<Option<number>>  => Some(3)
Stream.runCount(Stream.make(1, 2, 3)) // Effect<number>          => 3
Stream.runSum(Stream.make(1, 2, 3))   // Effect<number>          => 6
```

#### run (with a Sink)

```ts
Stream.make(1, 2, 3).pipe(
  Stream.run(Sink.sum)
)
// 6
```

#### toPull (manual consumption)

```ts
const program = Effect.scoped(
  Effect.gen(function*() {
    const pull = yield* Stream.toPull(Stream.make(1, 2, 3))
    const chunk = yield* pull
    // chunk: NonEmptyReadonlyArray<number>
  })
)
```

The pull fails with `Cause.Done` when the stream completes.

---

## Sink

```ts
// effect/Sink
interface Sink<out A, in In = unknown, out L = never, out E = never, out R = never>
```

A `Sink<A, In, L, E, R>` consumes a variable number of `In` elements, may
fail with `E`, and produces a final value `A` together with any unconsumed
leftovers `L`. Sinks are used with `Stream.run` or `Stream.transduce`.

### Key type: End

```ts
type End<A, L = never> = readonly [value: A, leftover?: NonEmptyReadonlyArray<L> | undefined]
```

### Common Sinks

#### drain

```ts
Sink.drain // Sink<void, unknown>
// Consumes everything, produces void
```

#### collect

```ts
Sink.collect<number>() // Sink<Array<number>, number>
// Accumulates all elements into an array
```

#### forEach / forEachArray

```ts
// Per-element side effect
Sink.forEach((n: number) => Console.log(n))
// Sink<void, number, never, never>

// Per-chunk side effect
Sink.forEachArray((chunk) => Console.log(chunk))
```

#### fold

```ts
// Fold with a continuation predicate (stops early when predicate is false)
Sink.fold(
  () => 0,           // initial state
  (s) => s < 100,    // continue while true
  (s, n) => Effect.succeed(s + n)
)
```

#### reduce

```ts
// Like fold but always runs to completion (no predicate)
Sink.reduce(() => 0, (acc, n: number) => acc + n)
```

#### take

```ts
Sink.take<number>(3) // Sink<Array<number>, number, number>
// Collects first 3 elements; leftovers are the remaining elements
```

#### head / last

```ts
Sink.head<number>() // Sink<Option<number>, number, number>
Sink.last<number>() // Sink<Option<number>, number>
```

#### sum / count

```ts
Sink.sum   // Sink<number, number>
Sink.count // Sink<number, unknown>
```

#### every / some

```ts
Sink.every((n: number) => n > 0) // Sink<boolean, number, number>
Sink.some((n: number) => n > 5)  // Sink<boolean, number, number>
```

### Sink Combinators

```ts
// Transform result
Sink.map(sink, (a) => a.toString())

// Transform input
Sink.mapInput(sink, (s: string) => parseInt(s))

// Effectful result transform
Sink.mapEffect(sink, (a) => Effect.succeed(a * 2))

// Sequential composition
Sink.flatMap(firstSink, (result) => secondSink(result))

// Timing
Sink.withDuration(sink) // Sink<[A, Duration], ...>
Sink.timed             // Sink<Duration, unknown>
```

### Building Sinks from Streams (v4)

`Sink.make<In>()` returns a curried constructor. The functions you pass are
applied in sequence with `pipe`. The **last** function must return
`Effect<Result, E, R>`; all earlier functions return non-Effect values (e.g.
another `Stream`).

```ts
// Single-step: the one function receives Stream<In> and returns an Effect
Sink.make<number>()(
  (stream) => stream.pipe(
    Stream.filter((n) => n > 0),
    Stream.runFold(() => 0, (acc, n) => acc + n)
  )
)
// Sink<number, number, never, never, never> -- sums positive numbers

// Collect first N items (single step)
const first5 = Sink.make<string>()(
  (stream) => stream.pipe(
    Stream.take(5),
    Stream.runCollect
  )
)
// Sink<Array<string>, string, never, never, never>

// Multi-step pipeline: intermediate steps return non-Effect values,
// only the last step returns Effect.  Each step is applied via pipe.
Sink.make<string>()(
  (stream) => stream.pipe(Stream.take(5)), // returns Stream<string>
  (stream) => Stream.runCollect(stream)    // returns Effect<Array<string>>
)
// Sink<Array<string>, string, never, never, never>
```

---

## Channel

```ts
// effect/Channel
interface Channel<
  out OutElem,
  out OutErr = never,
  out OutDone = void,
  in InElem = unknown,
  in InErr = unknown,
  in InDone = unknown,
  out Env = never
>
```

A Channel is the bidirectional I/O primitive underlying Stream and Sink. Most
users should prefer Stream and Sink directly. Use channels when building custom
operators or doing highly specialized work.

### Type Parameters

| Parameter | Direction | Description                           |
|-----------|-----------|---------------------------------------|
| OutElem   | out       | Elements the channel outputs          |
| OutErr    | out       | Errors the channel can produce        |
| OutDone   | out       | Final value when channel completes    |
| InElem    | in        | Elements the channel reads            |
| InErr     | in        | Errors the channel can receive        |
| InDone    | in        | Final value from upstream             |
| Env       | out       | Required environment/services         |

### Core Constructors

```ts
Channel.succeed(42)          // Channel<42>
Channel.fail("oops")        // Channel<never, string>
Channel.empty                // Channel<never>
Channel.never                // Channel<never, never, never>
Channel.fromEffect(effect)   // Channel from a single Effect
Channel.fromIterable([1,2])  // Channel that emits elements
Channel.fromArray(arrays)    // Channel from arrays
```

### fromTransform / fromPull

The v4 channel is fundamentally a **transform function** from an upstream
`Pull` to a downstream `Pull`:

```ts
Channel.fromTransform((upstream, scope) =>
  Effect.succeed(
    Effect.map(upstream, (value) => value * 2)
  )
)

Channel.fromPull(
  Effect.succeed(Effect.succeed(42))
)
```

### Composition

```ts
// Pipe output of one channel as input to another
Channel.pipeTo(source, transform)

// Concurrent merge
Channel.merge(left, right, { haltStrategy: "both" })
```

### Execution

```ts
Channel.runDrain(channel)          // Effect<OutDone, OutErr, Env>
Channel.runCollect(channel)        // Effect<Array<OutElem>, OutErr, Env>
Channel.runFold(channel, init, f)  // Effect<Z, OutErr, Env>
Channel.runForEach(channel, f)     // Effect<OutDone, OutErr, Env>
Channel.runDone(channel)           // Effect<OutDone, OutErr, Env>
```

---

## ChannelSchema

```ts
// effect/ChannelSchema
```

Integrates `Schema` encoding/decoding with channels for type-safe data
transformation pipelines.

### encode / decode

```ts
import * as ChannelSchema from "effect/ChannelSchema"
import * as Schema from "effect/Schema"

// Create a channel that encodes elements using a schema
const encoder = ChannelSchema.encode(Schema.Number)()
// Channel that transforms Type -> Encoded

const decoder = ChannelSchema.decode(Schema.Number)()
// Channel that transforms Encoded -> Type
```

### duplex

Wraps a channel with both input encoding and output decoding:

```ts
const channel = rawChannel.pipe(
  ChannelSchema.duplex({
    inputSchema: MyInputSchema,
    outputSchema: MyOutputSchema
  })
)
```

---

## Pull

```ts
// effect/Pull
interface Pull<out A, out E = never, out Done = void, out R = never>
  extends Effect<A, E | Cause.Done<Done>, R>
```

A `Pull` is a specialized `Effect` that models one step of a stream. It either:
- Succeeds with a value of type `A`
- Fails with an error of type `E`
- Signals completion by failing with `Cause.Done<Done>`

### Type Extractors

```ts
Pull.Success<P>   // Extract the success type
Pull.Error<P>     // Extract errors, excluding Done
Pull.Leftover<P>  // Extract the Done value type
Pull.Services<P>  // Extract service requirements
```

### Key Operations

#### catchDone

```ts
// Handle stream completion
Pull.catchDone(pull, (leftover) =>
  Effect.succeed("stream ended")
)
```

#### matchEffect

```ts
Pull.matchEffect(pull, {
  onSuccess: (value) => Effect.succeed(`Got: ${value}`),
  onFailure: (cause) => Effect.succeed(`Error: ${cause}`),
  onDone: (leftover) => Effect.succeed(`Done: ${leftover}`)
})
```

#### Done-related utilities

```ts
Pull.isDoneCause(cause)    // boolean -- does the Cause contain Done?
Pull.isDoneFailure(reason) // boolean -- is this specific reason a Done?
Pull.filterDone(cause)     // Extract Done from a Cause
Pull.doneExitFromCause(cause) // Convert to Exit<Done, E>
```

---

## Common Patterns

### Paginated API Consumption

`Stream.paginate` takes a seed state `S` and a function that returns
`Effect<[ReadonlyArray<A>, Option<S>]>`. Each call emits the items in the
returned array as individual stream elements. Return `Option.none()` to end
the stream after emitting the current page's items.

```ts
const fetchPages: Stream<Item> = Stream.paginate(1, (page) =>
  Effect.gen(function*() {
    const response = yield* fetchPage(page)
    const items = response.items  // ReadonlyArray<Item>
    const next = response.hasMore
      ? Option.some(page + 1)
      : Option.none()
    return [items, next] as const
    // paginate emits each Item individually -- no further flattening needed
  })
)
```

### Stream to Queue Bridge

`Stream.toQueue` converts a stream into a scoped `Queue.Dequeue` (when called
in curried/pipe form) that emits elements as they arrive and ends with
`Cause.Done` when the stream completes.

```ts
const program = Effect.scoped(
  Effect.gen(function*() {
    const queue = yield* Stream.make(1, 2, 3).pipe(
      Stream.toQueue({ capacity: 16 })
    )
    // queue: Queue.Dequeue<number, never | Cause.Done>
    // consume from queue...
  })
)

// To run a stream directly into an existing queue use Stream.runIntoQueue:
const program2 = Effect.gen(function*() {
  const q = yield* Queue.bounded<number, Cause.Done>(16)
  yield* Stream.runIntoQueue(Stream.make(1, 2, 3), q)
})
```

### Transduction (Chunked Processing)

```ts
// Process in batches using a Sink
Stream.range(1, 100).pipe(
  Stream.transduce(Sink.take(10)),
  Stream.runForEach((batch) => processBatch(batch))
)
```

### Broadcasting

`Stream.broadcast` returns a single shared stream backed by a PubSub. Every
subscriber to that shared stream receives all elements. Use `Stream.share`
when you want re-subscription support with an optional idle TTL.

```ts
// Fan out a stream to multiple consumers
Effect.scoped(
  Effect.gen(function*() {
    // broadcast returns Effect<Stream<A, E>> -- subscribe by consuming the
    // returned stream from multiple concurrent fibers
    const shared = yield* Stream.make(1, 2, 3).pipe(
      Stream.broadcast({ capacity: 16 })
    )

    const [r1, r2] = yield* Effect.all(
      [
        Stream.runCollect(shared),
        Stream.runCollect(shared)
      ],
      { concurrency: "unbounded" }
    )
    // r1 and r2 each receive all elements
  })
)
```

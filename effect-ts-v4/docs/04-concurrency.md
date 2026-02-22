# Concurrency Primitives

Effect 4.0 provides a rich set of concurrency primitives for building safe, structured concurrent programs. This document covers fibers, mutable references, synchronization primitives, queues, pub/sub, resource pooling, caching, and scheduling.

> **v4 Breaking Change:** `Fiber`, `Ref`, and `Deferred` are **no longer Effect subtypes**. In v3 you could `yield* ref` or `yield* fiber` directly. In v4 you must use explicit operations: `Ref.get(ref)`, `Deferred.await(deferred)`, `Fiber.join(fiber)`.

---

## Fiber -- Lightweight Concurrency

**Module:** `effect/Fiber`

Fibers are the fundamental unit of concurrency in Effect -- lightweight, user-space threads with structured concurrency guarantees. They are cooperative (yielding at effect boundaries), cancellation-safe, and traceable via numeric IDs.

### Type Definition

```ts
interface Fiber<out A, out E = never> extends Pipeable {
  readonly id: number
  readonly currentOpCount: number
  readonly addObserver: (cb: (exit: Exit<A, E>) => void) => () => void
  readonly interruptUnsafe: (fiberId?: number, annotations?: ServiceMap<never>) => void
  readonly pollUnsafe: () => Exit<A, E> | undefined
}
```

### Forking Effects into Fibers

> **v4 Breaking Change:** `Effect.fork` is renamed to `Effect.forkChild`. `Effect.forkDaemon` is renamed to `Effect.forkDetach`. `Effect.forkAll` and `Effect.forkWithErrorHandler` are removed.

| v3 | v4 | Description |
|---|---|---|
| `Effect.fork` | `Effect.forkChild` | Fork as a child of the current fiber |
| `Effect.forkDaemon` | `Effect.forkDetach` | Fork detached from parent lifecycle |
| `Effect.forkScoped` | `Effect.forkScoped` | Fork tied to the current Scope |
| `Effect.forkIn` | `Effect.forkIn` | Fork in a specific Scope |

All fork variants accept an optional options object:

```ts
{
  readonly startImmediately?: boolean   // Begin executing immediately
  readonly uninterruptible?: boolean | "inherit"  // Interruption control
}
```

### Key Functions

```ts
// Wait for a fiber to complete, getting the Exit value
Fiber.await: <A, E>(self: Fiber<A, E>) => Effect<Exit<A, E>>

// Wait for all fibers in an iterable, getting an array of Exit values
Fiber.awaitAll: <A extends Fiber<any, any>>(self: Iterable<A>) => Effect<Array<Exit<...>>>

// Wait for a fiber and unwrap the result (re-raises errors)
Fiber.join: <A, E>(self: Fiber<A, E>) => Effect<A, E>

// Join multiple fibers, unwrapping results (re-raises first failure)
Fiber.joinAll: <A extends Iterable<Fiber<any, any>>>(self: A) => Effect<Array<...>, ...>

// Interrupt a fiber
Fiber.interrupt: <A, E>(self: Fiber<A, E>) => Effect<void>

// Interrupt a fiber, attributing the interruption to a specific fiber ID
Fiber.interruptAs: (fiberId: number) => <A, E>(self: Fiber<A, E>) => Effect<void>

// Interrupt all fibers in a collection
Fiber.interruptAll: <A extends Iterable<Fiber<any, any>>>(fibers: A) => Effect<void>

// Interrupt all fibers attributing interruption to a specific fiber ID
Fiber.interruptAllAs: (fiberId: number) => <A extends Iterable<Fiber<any, any>>>(fibers: A) => Effect<void>

// Type guard
Fiber.isFiber: (u: unknown) => u is Fiber<unknown, unknown>

// Get the current fiber (may return undefined)
Fiber.getCurrent: () => Fiber<any, any> | undefined

// Link a fiber's lifetime to a Scope (new in v4)
Fiber.runIn: (scope: Scope) => <A, E>(self: Fiber<A, E>) => Fiber<A, E>
```

### Example

```ts
import { Console, Effect, Fiber } from "effect"

const program = Effect.gen(function*() {
  // Fork a background task
  const fiber = yield* Effect.forkChild(
    Effect.gen(function*() {
      yield* Effect.sleep("2 seconds")
      return "background result"
    })
  )

  yield* Console.log("Doing other work...")
  yield* Effect.sleep("1 second")

  // Wait for the fiber to complete
  const result = yield* Fiber.join(fiber)
  yield* Console.log(`Fiber result: ${result}`)
})
```

---

## FiberHandle -- Single Fiber Management

**Module:** `effect/FiberHandle`

A `FiberHandle` stores and manages a single fiber. When the associated Scope closes, the contained fiber is interrupted. Running a new effect replaces (and interrupts) the previous fiber.

```ts
interface FiberHandle<out A = unknown, out E = unknown> extends Pipeable {
  readonly deferred: Deferred<void, unknown>
  state: { _tag: "Open"; fiber: Fiber<A, E> | undefined } | { _tag: "Closed" }
}
```

### Key Functions

```ts
FiberHandle.make<A, E>(): Effect<FiberHandle<A, E>, never, Scope>
FiberHandle.run(handle, effect, options?): Effect<Fiber<XA, XE>, never, R>
FiberHandle.set(handle, fiber, options?): Effect<void>            // Add an existing fiber
FiberHandle.get(handle): Effect<Fiber<A, E> | undefined>          // Retrieve the current fiber
FiberHandle.clear(handle): Effect<void>                           // Interrupt and remove the fiber
FiberHandle.join(handle): Effect<void, E>   // Suspend until failure propagation
FiberHandle.awaitEmpty(handle): Effect<void, E>                   // Wait for fiber to complete
FiberHandle.runtime(handle)<R>(): Effect<RunFunction, never, R>   // Capture runtime for out-of-Effect use
FiberHandle.makeRuntime<R>(): Effect<RunFunction, never, Scope | R> // Shorthand: make + runtime
```

### Example

```ts
import { Effect, FiberHandle } from "effect"

Effect.gen(function*() {
  const handle = yield* FiberHandle.make()

  yield* FiberHandle.run(handle, Effect.never)
  // Running another effect interrupts the previous one
  yield* FiberHandle.run(handle, Effect.never)

  yield* Effect.sleep(1000)
}).pipe(Effect.scoped)
```

---

## FiberHandle, FiberMap, FiberSet -- Runtime Helpers

All three modules (`FiberHandle`, `FiberMap`, `FiberSet`) also provide `runtime` and `runtimePromise` helpers that capture the current service environment and return a function for forking effects outside of the generator context. This is useful for bridging with imperative or callback-based APIs.

```ts
// FiberHandle: runtime captures services; returns a function that runs effects and replaces the fiber
const run = yield* FiberHandle.runtime(handle)<MyService>()
run(someEffect) // returns Fiber synchronously, replaces previous fiber

// FiberMap: runtime captures services; returns a keyed function
const run = yield* FiberMap.runtime(map)<MyService>()
run("task-key", someEffect) // returns Fiber, keyed by "task-key"

// FiberSet: runtime captures services; returns an unkeyed function
const run = yield* FiberSet.runtime(set)<MyService>()
run(someEffect) // returns Fiber, added to set

// All three also have runtimePromise variants that return Promise instead of Fiber:
const runPromise = yield* FiberHandle.runtimePromise(handle)<MyService>()
runPromise(someEffect) // returns Promise<A>
```

---

## FiberMap -- Map of Managed Fibers

**Module:** `effect/FiberMap`

A `FiberMap<K>` is a key-indexed collection of fibers. When the Scope closes, all fibers are interrupted. Setting a fiber for an existing key interrupts the previous one. Fibers are automatically removed when they complete.

### Key Functions

```ts
FiberMap.make<K, A, E>(): Effect<FiberMap<K, A, E>, never, Scope>
FiberMap.run(map, key, effect, options?): Effect<Fiber<XA, XE>, never, R>
FiberMap.set(map, key, fiber, options?): Effect<void>     // Add an existing fiber at a key
FiberMap.get(map, key): Effect<Fiber<A, E> | undefined>  // Returns undefined if key absent
FiberMap.has(map, key): Effect<boolean>
FiberMap.remove(map, key): Effect<void>   // Interrupts the fiber at that key
FiberMap.clear(map): Effect<void>         // Interrupts all fibers
FiberMap.size(map): Effect<number>
FiberMap.join(map): Effect<void, E>       // Suspends until first failure
FiberMap.awaitEmpty(map): Effect<void, E> // Waits for all fibers to complete
```

### Example: Named Worker Pool

```ts
import { Effect, FiberMap } from "effect"

Effect.gen(function*() {
  const workers = yield* FiberMap.make<string>()

  // Start named workers -- each key holds exactly one fiber
  yield* FiberMap.run(workers, "poller", Effect.forever(
    Effect.gen(function*() {
      yield* Effect.sleep("5 seconds")
      yield* Effect.log("Polling...")
    })
  ))

  yield* FiberMap.run(workers, "heartbeat", Effect.forever(
    Effect.gen(function*() {
      yield* Effect.sleep("30 seconds")
      yield* Effect.log("Heartbeat")
    })
  ))

  // Replace a worker -- the old "poller" fiber is interrupted
  yield* FiberMap.run(workers, "poller", Effect.never)

  // Check status
  const count = yield* FiberMap.size(workers)
  console.log(`Active workers: ${count}`) // 2
}).pipe(Effect.scoped)
```

---

## FiberSet -- Set of Managed Fibers

**Module:** `effect/FiberSet`

A `FiberSet` is an unkeyed collection of fibers. Fibers are automatically removed when they complete. Scope closure interrupts all fibers.

### Key Functions

```ts
FiberSet.make<A, E>(): Effect<FiberSet<A, E>, never, Scope>
FiberSet.run(set, effect, options?): Effect<Fiber<XA, XE>, never, R>
FiberSet.add(set, fiber, options?): Effect<void>   // Add an existing fiber to the set
FiberSet.clear(set): Effect<void>                  // Interrupts all fibers in the set
FiberSet.size(set): Effect<number>
FiberSet.join(set): Effect<void, E>                // Suspends until first failure
FiberSet.awaitEmpty(set): Effect<void>             // Waits for all fibers to complete (no E -- does not propagate failure)
```

### Example: Dynamic Task Spawner

```ts
import { Effect, FiberSet } from "effect"

Effect.gen(function*() {
  const tasks = yield* FiberSet.make()

  // Spawn multiple tasks into the set
  for (const i of [1, 2, 3, 4, 5]) {
    yield* FiberSet.run(tasks, Effect.gen(function*() {
      yield* Effect.sleep(`${i * 100} millis`)
      yield* Effect.log(`Task ${i} done`)
    }))
  }

  // Wait for all tasks to finish
  yield* FiberSet.awaitEmpty(tasks)
  yield* Effect.log("All tasks completed")
}).pipe(Effect.scoped)
```

---

## Ref -- Mutable Reference

**Module:** `effect/Ref`

A `Ref<A>` is a thread-safe mutable reference providing atomic read, write, and update operations.

> **v4 Breaking Change:** `Ref` is no longer an Effect subtype. Use `Ref.get(ref)` instead of `yield* ref`.

```ts
interface Ref<in out A> extends Pipeable {
  readonly ref: MutableRef<A>
}
```

### Key Functions

```ts
Ref.make<A>(value: A): Effect<Ref<A>>
Ref.makeUnsafe<A>(value: A): Ref<A>         // New in v4
Ref.get<A>(self: Ref<A>): Effect<A>          // Was implicit via yield* in v3
Ref.getUnsafe<A>(self: Ref<A>): A            // Sync read
Ref.set<A>(self: Ref<A>, value: A): Effect<void>
Ref.update<A>(self: Ref<A>, f: (a: A) => A): Effect<void>
Ref.modify<A, B>(self: Ref<A>, f: (a: A) => readonly [B, A]): Effect<B>
Ref.getAndSet<A>(self: Ref<A>, value: A): Effect<A>
Ref.getAndUpdate<A>(self: Ref<A>, f: (a: A) => A): Effect<A>
Ref.updateAndGet<A>(self: Ref<A>, f: (a: A) => A): Effect<A>
```

### Example

```ts
import { Effect, Ref } from "effect"

const program = Effect.gen(function*() {
  const counter = yield* Ref.make(0)

  yield* Ref.update(counter, (n) => n + 1)
  yield* Ref.update(counter, (n) => n * 2)

  const value = yield* Ref.get(counter)
  console.log(value) // 2

  const previous = yield* Ref.getAndSet(counter, 100)
  console.log(previous) // 2
})
```

---

## SynchronizedRef -- Ref with Effectful Updates

**Module:** `effect/SynchronizedRef`

A `SynchronizedRef<A>` extends `Ref<A>` with a semaphore-guarded lock, allowing effectful update functions. All operations are serialized so effectful computations inside updates are safe. Use this when your update logic needs to perform side effects (API calls, database queries, etc.) while holding the lock.

```ts
interface SynchronizedRef<in out A> extends Ref<A> {
  readonly backing: Ref<A>
  readonly semaphore: Semaphore
}
```

### Key Functions

All standard `Ref` operations plus effectful variants:

```ts
SynchronizedRef.make<A>(value: A): Effect<SynchronizedRef<A>>
SynchronizedRef.makeUnsafe<A>(value: A): SynchronizedRef<A>

// Effectful operations -- the key differentiator from Ref
SynchronizedRef.modifyEffect(self, f: (a: A) => Effect<readonly [B, A], E, R>): Effect<B, E, R>
SynchronizedRef.updateEffect(self, f: (a: A) => Effect<A, E, R>): Effect<void, E, R>
SynchronizedRef.getAndUpdateEffect(self, f: (a: A) => Effect<A, E, R>): Effect<A, E, R>
SynchronizedRef.updateAndGetEffect(self, f: (a: A) => Effect<A, E, R>): Effect<A, E, R>
```

### Example: Effectful State Updates

```ts
import { Effect, SynchronizedRef } from "effect"

interface AppState {
  count: number
  lastUpdated: number
}

const program = Effect.gen(function*() {
  const state = yield* SynchronizedRef.make<AppState>({
    count: 0,
    lastUpdated: 0
  })

  // Update requires calling an effect (e.g., getting the current time)
  yield* SynchronizedRef.updateEffect(state, (s) =>
    Effect.sync(() => ({
      count: s.count + 1,
      lastUpdated: Date.now()
    }))
  )

  const current = yield* SynchronizedRef.get(state)
  console.log(current.count) // 1
})
```

### When to Use Which Ref

| Ref Type | Use Case |
|---|---|
| `Ref` | Simple synchronous state (counters, flags, accumulators) |
| `SynchronizedRef` | State updates that require effects (API calls, timestamps) |
| `SubscriptionRef` | State that others need to observe via streams |

---

## SubscriptionRef -- Ref with Change Notifications

**Module:** `effect/SubscriptionRef`

A `SubscriptionRef<A>` combines `Ref` semantics with a `PubSub` so that subscribers receive a stream of all value changes.

```ts
interface SubscriptionRef<in out A> extends Pipeable {
  readonly backing: Ref<A>
  readonly semaphore: Semaphore
  readonly pubsub: PubSub<A>
}
```

### Key Functions

```ts
SubscriptionRef.make<A>(value: A): Effect<SubscriptionRef<A>>
SubscriptionRef.get(self): Effect<A>
SubscriptionRef.set(self, value): Effect<void>
SubscriptionRef.update(self, f): Effect<void>
SubscriptionRef.changes(self): Stream<A>   // Stream of current + all future values
// Plus effectful variants: updateEffect, modifyEffect, etc.
```

### Example

```ts
import { Effect, Stream, SubscriptionRef } from "effect"

const program = Effect.gen(function*() {
  const ref = yield* SubscriptionRef.make(0)
  const stream = SubscriptionRef.changes(ref)

  const fiber = yield* Stream.runForEach(
    stream,
    (value) => Effect.sync(() => console.log("Value:", value))
  ).pipe(Effect.forkScoped)

  yield* SubscriptionRef.set(ref, 1)
  yield* SubscriptionRef.set(ref, 2)
})
```

---

## Deferred -- One-Shot Async Variable

**Module:** `effect/Deferred`

A `Deferred<A, E>` is an asynchronous variable that can be set exactly once. Multiple fibers can await the same Deferred and all are notified when it completes.

> **v4 Breaking Change:** `Deferred` is no longer an Effect subtype. Use `Deferred.await(deferred)` instead of `yield* deferred`.

```ts
interface Deferred<in out A, in out E = never> extends Pipeable {
  effect?: Effect<A, E>
  resumes?: Array<(effect: Effect<A, E>) => void>
}
```

### Key Functions

```ts
Deferred.make<A, E = never>(): Effect<Deferred<A, E>>
Deferred.makeUnsafe<A, E = never>(): Deferred<A, E>

// Awaiting
Deferred.await(self): Effect<A, E>         // Suspends until completed
Deferred.isDone(self): Effect<boolean>
Deferred.poll(self): Effect<Effect<A, E> | undefined>

// Completing
Deferred.succeed(self, value): Effect<boolean>
Deferred.fail(self, error): Effect<boolean>
Deferred.done(self, exit): Effect<boolean>
Deferred.complete(self, effect): Effect<boolean>  // Memoizes the effect
Deferred.completeWith(self, effect): Effect<boolean>  // Faster, no memoization
Deferred.interrupt(self): Effect<boolean>

// Piping an effect's result into a Deferred
Deferred.into(effect, deferred): Effect<boolean>

// Unsafe
Deferred.doneUnsafe(self, effect): boolean
```

### Example

```ts
import { Deferred, Effect, Fiber } from "effect"

const program = Effect.gen(function*() {
  const deferred = yield* Deferred.make<string, never>()

  const waiter = yield* Effect.forkChild(
    Effect.gen(function*() {
      const value = yield* Deferred.await(deferred)
      console.log("Received:", value)
      return value
    })
  )

  yield* Effect.sleep("1 second")
  yield* Deferred.succeed(deferred, "Hello from setter!")
  yield* Fiber.join(waiter)
})
```

---

## Queue -- Bounded/Unbounded Queues

**Module:** `effect/Queue`

A `Queue<A, E>` is an asynchronous queue supporting offer/take operations with backpressure strategies. Queues can also be signaled as done or failed.

> **v4 Note:** Queue replaces the v3 `Mailbox` module and has separate `Enqueue` (write) and `Dequeue` (read) interfaces.

```ts
interface Queue<in out A, in out E = never> extends Enqueue<A, E>, Dequeue<A, E> {}
```

### Constructors

```ts
Queue.bounded<A, E = never>(capacity: number): Effect<Queue<A, E>>   // Backpressure
Queue.unbounded<A, E = never>(): Effect<Queue<A, E>>                  // No limit
Queue.sliding<A, E = never>(capacity: number): Effect<Queue<A, E>>    // Drops oldest
Queue.dropping<A, E = never>(capacity: number): Effect<Queue<A, E>>   // Drops newest
Queue.make<A, E>(options?): Effect<Queue<A, E>>                       // General constructor
```

### Key Functions

```ts
// Offering (write side)
Queue.offer(queue, message): Effect<boolean>
Queue.offerAll(queue, messages): Effect<Array<A>>  // Returns remaining items that didn't fit
Queue.offerUnsafe(queue, message): boolean         // Synchronous, no Effect wrapping

// Taking (read side)
Queue.take(queue): Effect<A, E>              // Blocks until an item is available
Queue.takeAll(queue): Effect<NonEmptyArray<A>, E>  // Blocks until at least one item, then takes all
Queue.takeN(queue, n): Effect<Array<A>, E>
Queue.takeBetween(queue, min, max): Effect<Array<A>, E>
Queue.poll(queue): Effect<Option<A>>         // Non-blocking single item
Queue.peek(queue): Effect<A, E>              // View front item without removing
Queue.collect(queue): Effect<Array<A>, Pull.ExcludeDone<E>>  // Drain until Done (strips Done from error)
Queue.clear(queue): Effect<Array<A>, Pull.ExcludeDone<E>>   // Remove all current items without blocking

// Completion
Queue.end(queue): Effect<boolean>            // Signal normal completion (requires E includes Cause.Done)
Queue.fail(queue, error): Effect<boolean>    // Signal error completion
Queue.interrupt(queue): Effect<boolean>      // Stop new offers; allow draining existing messages
Queue.shutdown(queue): Effect<boolean>       // Immediately cancel all pending operations

// Inspection
Queue.size(queue): Effect<number>
Queue.isFull(queue): Effect<boolean>
```

### Example: Basic Queue

```ts
import { Effect, Queue } from "effect"

const program = Effect.gen(function*() {
  const queue = yield* Queue.bounded<string>(10)

  yield* Queue.offer(queue, "hello")
  yield* Queue.offerAll(queue, ["world", "!"])

  const item1 = yield* Queue.take(queue)
  const item2 = yield* Queue.take(queue)
  const item3 = yield* Queue.take(queue)

  console.log([item1, item2, item3]) // ["hello", "world", "!"]
})
```

### Example: Producer-Consumer with Completion

```ts
import { Cause, Effect, Fiber, Queue } from "effect"

const program = Effect.gen(function*() {
  const queue = yield* Queue.bounded<number, Cause.Done>(100)

  // Producer: generates items then signals completion
  const producer = yield* Effect.forkChild(
    Effect.gen(function*() {
      for (let i = 0; i < 10; i++) {
        yield* Queue.offer(queue, i)
        yield* Effect.sleep("100 millis")
      }
      yield* Queue.end(queue) // Signal no more items
    })
  )

  // Consumer: takes items until the queue is done
  const consumer = yield* Effect.forkChild(
    Effect.gen(function*() {
      const all = yield* Queue.collect(queue)
      console.log(all) // [0, 1, 2, ..., 9]
      return all
    })
  )

  yield* Fiber.join(producer)
  const results = yield* Fiber.join(consumer)
  return results
})
```

### Queue Completion Model

Queues use `Done` (from `Cause`) as a completion signal in the error channel:

- **`Queue.bounded<A, Cause.Done>(n)`** -- E type includes `Done` to enable `Queue.end`
- **`Queue.end(queue)`** -- signals normal completion; subsequent `take` will fail with `Done` cause
- **`Queue.fail(queue, error)`** -- signals error completion
- **`Queue.collect(queue)`** -- drains all items until `Done`, returns `Array<A>` (strips `Done` from error type via `Pull.ExcludeDone<E>`)
- **`Queue.interrupt(queue)`** -- stops accepting new offers but allows draining existing messages; once drained, queue fails with an interrupt cause
- **`Queue.shutdown(queue)`** -- immediately cancels all pending operations, clears the queue, and transitions to done state

> **Note:** There is no `Queue.await`. To wait for a queue to be fully processed, use `Queue.collect` (for `Cause.Done` queues) or drive the consume loop yourself with `Queue.take` in a loop.

**Key insight:** `Done` is encoded as an error in the `Exit`, but `Queue.collect` uses `Pull.catchDone` internally to treat it as successful completion, not a failure.

### Manual Take Loop with Done Detection

If you need item-by-item processing instead of `Queue.collect`, detect `Done` using `Effect.catchCause` and `Pull.isDoneCause`:

```ts
import { Cause, Effect, Pull, Queue } from "effect"

const processItems = (queue: Queue.Queue<string, Cause.Done>) => {
  const loop: Effect.Effect<void> = Effect.gen(function*() {
    const item = yield* Queue.take(queue)
    yield* Effect.log(`Processing: ${item}`)
    yield* loop
  }).pipe(
    Effect.catchCause((cause) =>
      Pull.isDoneCause(cause)
        ? Effect.void   // Queue ended normally -- exit loop
        : Effect.failCause(cause)  // Real error -- re-raise
    )
  )
  return loop
}
```

### Backpressure Strategies

| Strategy | Constructor | Full Queue Behavior |
|---|---|---|
| **Suspend** (backpressure) | `Queue.bounded(n)` | Producer blocks until space available |
| **Sliding** | `Queue.sliding(n)` | Oldest element dropped, producer succeeds |
| **Dropping** | `Queue.dropping(n)` | New element dropped, producer returns `false` |
| **Unbounded** | `Queue.unbounded()` | Queue grows without limit |

---

## PubSub -- Publish-Subscribe

**Module:** `effect/PubSub`

A `PubSub<A>` is an asynchronous message hub where publishers send messages and subscribers receive them. Supports backpressure strategies and message replay.

```ts
interface PubSub<in out A> extends Pipeable {
  readonly pubsub: PubSub.Atomic<A>
  readonly subscribers: PubSub.Subscribers<A>
  readonly scope: Scope.Closeable
  readonly shutdownHook: Latch
  readonly shutdownFlag: MutableRef<boolean>
  readonly strategy: PubSub.Strategy<A>
}
```

### Constructors

```ts
// All constructors accept either a plain capacity number or an options object with optional replay buffer
PubSub.bounded<A>(capacity: number | { capacity: number; replay?: number }): Effect<PubSub<A>>
PubSub.unbounded<A>(options?: { replay?: number }): Effect<PubSub<A>>
PubSub.dropping<A>(capacity: number | { capacity: number; replay?: number }): Effect<PubSub<A>>
PubSub.sliding<A>(capacity: number | { capacity: number; replay?: number }): Effect<PubSub<A>>
```

### Key Functions

```ts
PubSub.publish(pubsub, value): Effect<boolean>
PubSub.publishAll(pubsub, values): Effect<Array<A>>
PubSub.subscribe(pubsub): Effect<Subscription<A>, never, Scope>

// Subscription operations (read side)
PubSub.take(subscription): Effect<A>
PubSub.takeAll(subscription): Effect<NonEmptyArray<A>>
PubSub.takeUpTo(subscription, n): Effect<Array<A>>
PubSub.takeBetween(subscription, min, max): Effect<Array<A>>
```

### Example

```ts
import { Effect, PubSub } from "effect"

const program = Effect.gen(function*() {
  const pubsub = yield* PubSub.bounded<string>(10)

  // Subscribers must be created BEFORE publishing if no replay buffer is configured.
  // Messages published before a subscription is created are not delivered to that subscriber.
  yield* Effect.scoped(Effect.gen(function*() {
    const subscription = yield* PubSub.subscribe(pubsub)

    yield* PubSub.publish(pubsub, "Hello")

    const message = yield* PubSub.take(subscription)
    console.log(message) // "Hello"
  }))
})
```

---

## Pool -- Resource Pooling

**Module:** `effect/Pool`

A `Pool<A, E>` manages a pool of resources with automatic lifecycle management, concurrency control, and optional TTL-based invalidation.

```ts
interface Pool<in out A, in out E = never> extends Pipeable {
  readonly config: Config<A, E>
  readonly state: State<A, E>
}
```

### Constructors

```ts
// Fixed-size pool
Pool.make<A, E, R>(options: {
  acquire: Effect<A, E, R>
  size: number
  concurrency?: number        // Concurrent access per item (default 1)
  targetUtilization?: number  // 0..1, when to create new items
}): Effect<Pool<A, E>, never, R | Scope>

// Dynamic pool with TTL
Pool.makeWithTTL<A, E, R>(options: {
  acquire: Effect<A, E, R>
  min: number
  max: number
  timeToLive: Duration.Input
  timeToLiveStrategy?: "creation" | "usage"
  concurrency?: number
  targetUtilization?: number
}): Effect<Pool<A, E>, never, R | Scope>
```

---

## Cache -- Caching with TTL

**Module:** `effect/Cache`

A `Cache<Key, A, E, R>` provides a mutable key-value store with automatic TTL management, capacity limits, and lookup functions for cache misses. Multiple concurrent gets for the same key only invoke the lookup once.

> **New in v4:** Cache is now a standalone module (previously `Cache` from `effect`).

```ts
interface Cache<in out Key, in out A, in out E = never, out R = never> extends Pipeable {
  readonly capacity: number
  readonly lookup: (key: Key) => Effect<A, E, R>
  readonly timeToLive: (exit: Exit<A, E>, key: Key) => Duration
}
```

### Constructors

```ts
Cache.make<Key, A, E, R>(options: {
  lookup: (key: Key) => Effect<A, E, R>
  capacity: number
  timeToLive?: Duration.Input
}): Effect<Cache<Key, A, E>>

// Dynamic TTL based on result and key
Cache.makeWith<Key, A, E, R>(options: {
  lookup: (key: Key) => Effect<A, E, R>
  capacity: number
  timeToLive?: (exit: Exit<A, E>, key: Key) => Duration.Input
}): Effect<Cache<Key, A, E>>
```

### Key Functions

```ts
Cache.get(cache, key): Effect<A, E, R>
Cache.set(cache, key, value): Effect<void>
Cache.has(cache, key): Effect<boolean>
Cache.invalidate(cache, key): Effect<void>
Cache.invalidateAll(cache): Effect<void>
Cache.invalidateWhen(cache, predicate): Effect<void>
Cache.refresh(cache, key): Effect<void>
Cache.size(cache): Effect<number>
Cache.keys(cache): Effect<Iterable<Key>>
Cache.values(cache): Effect<Iterable<A>>
Cache.entries(cache): Effect<Iterable<[Key, A]>>
```

### Example

```ts
import { Cache, Effect } from "effect"

const program = Effect.gen(function*() {
  let lookupCount = 0
  const cache = yield* Cache.make({
    capacity: 100,
    lookup: (key: string) =>
      Effect.sync(() => {
        lookupCount++
        return key.length
      }),
    timeToLive: "5 minutes"
  })

  const v1 = yield* Cache.get(cache, "hello") // Calls lookup
  const v2 = yield* Cache.get(cache, "hello") // Returns cached
  console.log(v1, v2, lookupCount)            // 5, 5, 1
})
```

---

## ScopedCache -- Scoped Caching

**Module:** `effect/ScopedCache`

`ScopedCache` is like `Cache` but each cached value has its own `Scope` for resource management. When entries are evicted or invalidated, their scope is closed, cleaning up resources.

```ts
interface ScopedCache<in out Key, in out A, in out E = never, out R = never> extends Pipeable {
  readonly capacity: number
  readonly lookup: (key: Key) => Effect<A, E, R | Scope>
  readonly timeToLive: (exit: Exit<A, E>, key: Key) => Duration
}
```

### Constructors

```ts
ScopedCache.make<Key, A, E, R>(options: {
  lookup: (key: Key) => Effect<A, E, R | Scope>
  capacity: number
  timeToLive?: Duration.Input
}): Effect<ScopedCache<Key, A, E>, never, R | Scope>

ScopedCache.makeWith<Key, A, E, R>(options: {
  lookup: (key: Key) => Effect<A, E, R | Scope>
  capacity: number
  timeToLive?: (exit: Exit<A, E>, key: Key) => Duration.Input
}): Effect<ScopedCache<Key, A, E>, never, R | Scope>
```

### Key Functions

```ts
ScopedCache.get(cache, key): Effect<A, E, R | Scope>
ScopedCache.has(cache, key): Effect<boolean>
ScopedCache.invalidate(cache, key): Effect<void>
ScopedCache.invalidateAll(cache): Effect<void>
ScopedCache.refresh(cache, key): Effect<void, E, R>
ScopedCache.size(cache): Effect<number>
```

---

## Semaphore -- Concurrency Limiting

**Module:** `effect/Semaphore`

A `Semaphore` controls concurrent access by managing a pool of permits. Fibers acquire permits before executing and release them afterward.

```ts
interface Semaphore {
  withPermits(permits: number): <A, E, R>(self: Effect<A, E, R>) => Effect<A, E, R>
  withPermit<A, E, R>(self: Effect<A, E, R>): Effect<A, E, R>
  withPermitsIfAvailable(permits: number):
    <A, E, R>(self: Effect<A, E, R>) => Effect<Option<A>, E, R>
  take(permits: number): Effect<number>
  release(permits: number): Effect<number>
  releaseAll: Effect<number>
  resize(permits: number): Effect<void>
}
```

### Constructors

```ts
Semaphore.make(permits: number): Effect<Semaphore>
Semaphore.makeUnsafe(permits: number): Semaphore

// Partitioned semaphore -- distributes permits across keyed partitions
Semaphore.makePartitioned<K>(options: { permits: number }): Effect<Semaphore.Partitioned<K>>
Semaphore.makePartitionedUnsafe<K>(options: { permits: number }): Semaphore.Partitioned<K>
```

### Example

```ts
import { Effect, Semaphore } from "effect"

const program = Effect.gen(function*() {
  const semaphore = yield* Semaphore.make(2)

  const task = (id: number) =>
    semaphore.withPermits(1)(
      Effect.gen(function*() {
        yield* Effect.log(`Task ${id} acquired permit`)
        yield* Effect.sleep("1 second")
        yield* Effect.log(`Task ${id} releasing permit`)
      })
    )

  // Run 4 tasks, but only 2 can run concurrently
  yield* Effect.all([task(1), task(2), task(3), task(4)], {
    concurrency: "unbounded"
  })
})
```

---

## Latch -- Binary Coordination

**Module:** `effect/Latch`

A `Latch` is a binary coordination primitive -- a gate that can be opened or closed. Fibers can wait for the latch to open before proceeding.

```ts
interface Latch {
  readonly open: Effect<boolean>
  readonly openUnsafe: () => boolean
  readonly close: Effect<boolean>
  readonly closeUnsafe: () => boolean
  readonly release: Effect<boolean>  // Release waiters without opening
  readonly await: Effect<void>       // Wait for latch to be opened
  readonly whenOpen: <A, E, R>(self: Effect<A, E, R>) => Effect<A, E, R>
}
```

### Constructors

```ts
Latch.make(open?: boolean): Effect<Latch>
Latch.makeUnsafe(open?: boolean): Latch
```

### Example

```ts
import { Effect, Latch } from "effect"

const program = Effect.gen(function*() {
  const latch = yield* Latch.make(false)

  const waiter = Effect.gen(function*() {
    yield* Effect.log("Waiting for latch to open...")
    yield* latch.await
    yield* Effect.log("Latch opened! Continuing...")
  })

  const opener = Effect.gen(function*() {
    yield* Effect.sleep("2 seconds")
    yield* Effect.log("Opening latch...")
    yield* latch.open
  })

  yield* Effect.all([waiter, opener], { concurrency: "unbounded" })
})
```

---

## Schedule -- Retry/Repeat Scheduling

**Module:** `effect/Schedule`

A `Schedule<Output, Input, Error, Env>` defines a strategy for repeating or retrying effects. Schedules are composable and can incorporate timing, jitter, and custom logic.

```ts
interface Schedule<out Output, in Input = unknown, out Error = never, out Env = never>
  extends Pipeable {}
```

### v4 Changes

Schedule now uses `InputMetadata<Input>` and `Metadata<Output, Input>` for rich context in custom schedules:

```ts
interface InputMetadata<Input> {
  readonly input: Input
  readonly attempt: number
  readonly start: number
  readonly now: number
  readonly elapsed: number
  readonly elapsedSincePrevious: number
}
```

### Common Constructors

```ts
// Fixed interval between executions
Schedule.spaced(duration: Duration.Input): Schedule<number>

// Exponential backoff
Schedule.exponential(base: Duration.Input, factor?: number): Schedule<Duration>

// Fibonacci-based delays
Schedule.fibonacci(one: Duration.Input): Schedule<Duration>

// Fixed-window schedule
Schedule.fixed(interval: Duration.Input): Schedule<number>
Schedule.windowed(interval: Duration.Input): Schedule<number>

// Count-limited
Schedule.recurs(times: number): Schedule<number>
Schedule.forever: Schedule<number>

// Time-based
Schedule.elapsed: Schedule<Duration>

// State-based -- emits the current state on each step; next computes the next state
Schedule.unfold<State>(initial: State, next: (state: State) => Effect<State>): Schedule<State>

// Cron expression
Schedule.cron(expression: string): Schedule<...>
```

### Combinators

```ts
// Limit iterations
Schedule.take(schedule, n): Schedule<...>

// Add additional delay (effectful; f receives current output and returns extra Duration)
Schedule.addDelay(schedule, f: (output) => Effect<Duration.Input>): Schedule<...>

// Add random jitter to delays
Schedule.jittered(schedule): Schedule<...>

// Compose schedules sequentially (left completes first, then right runs)
Schedule.compose(left, right): Schedule<...>

// Transform output (f may be synchronous or return an Effect)
Schedule.map(schedule, f: (output) => B | Effect<B>): Schedule<B, ...>

// Side effects on each output (f must return an Effect)
Schedule.tapOutput(schedule, f: (output) => Effect<X>): Schedule<...>
```

### Example

```ts
import { Effect, Schedule } from "effect"

// Retry with exponential backoff, max 3 attempts
const retryPolicy = Schedule.exponential("100 millis", 2.0).pipe(
  Schedule.compose(Schedule.recurs(3))
)

const program = Effect.retry(
  Effect.fail("Network error"),
  retryPolicy
)

// Repeat on a fixed schedule
const heartbeat = Effect.log("heartbeat").pipe(
  Effect.repeat(Schedule.spaced("30 seconds"))
)
```

---

## Migration Summary: v3 to v4

### No Longer Effect Subtypes

In v3, you could write `yield* ref` or `yield* fiber`. In v4, these types do not extend Effect:

| Type | v3 | v4 |
|---|---|---|
| `Ref<A>` | `yield* ref` reads the value | `yield* Ref.get(ref)` |
| `Deferred<A, E>` | `yield* deferred` awaits completion | `yield* Deferred.await(deferred)` |
| `Fiber<A, E>` | `yield* fiber` joins the fiber | `yield* Fiber.join(fiber)` |

### Fork Renames

| v3 | v4 |
|---|---|
| `Effect.fork(effect)` | `Effect.forkChild(effect)` |
| `Effect.forkDaemon(effect)` | `Effect.forkDetach(effect)` |
| `Effect.forkScoped(effect)` | `Effect.forkScoped(effect)` (unchanged) |
| `Effect.forkIn(scope)(effect)` | `Effect.forkIn(scope)(effect)` (unchanged) |
| `Effect.forkAll(effects)` | Removed -- use `Effect.all` with `concurrency` or fork individually |
| `Effect.forkWithErrorHandler` | Removed -- use `Fiber.join` or `Fiber.await` |

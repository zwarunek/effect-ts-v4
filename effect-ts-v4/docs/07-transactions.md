# Transactions (Software Transactional Memory)

Effect 4.0 replaces the v3 `STM` module with a simpler, unified transaction system built directly into `Effect`. Transactions use `Effect.atomic` (or `Effect.transaction` for isolation) together with a family of `Tx*` data structures that participate in optimistic concurrency control.

## Migration from v3 STM

In Effect v3, STM was a separate effect type with its own combinators (`STM.map`, `STM.flatMap`, etc.) and required explicit `STM.commit` to run. In v4:

| v3 (STM)                        | v4 (Transactions)                         |
|----------------------------------|-------------------------------------------|
| `STM<A, E, R>`                   | `Effect<A, E, R>` (same type)             |
| `STM.commit(stm)`               | `Effect.atomic(effect)`                   |
| `TRef.make(0)`                   | `TxRef.make(0)`                           |
| `TRef.get(ref)` (returns STM)    | `TxRef.get(ref)` (returns Effect)         |
| `TRef.set(ref, v)` (returns STM) | `TxRef.set(ref, v)` (returns Effect)      |
| `STM.retry`                      | `Effect.retryTransaction`                 |
| Separate `TMap`, `TSet`, etc.    | `TxHashMap`, `TxHashSet`, `TxQueue`, etc. |

The key insight: there is no separate `STM` type. All `Tx*` operations return plain `Effect` values. Wrapping them in `Effect.atomic` makes multi-step operations atomic.

---

## Effect.atomic

```ts
export const atomic: <A, E, R>(
  effect: Effect<A, E, R>
) => Effect<A, E, Exclude<R, Transaction>>
```

`Effect.atomic` defines an optimistic transaction boundary. All reads and writes to `Tx*` data structures inside the block are journaled. On commit, the runtime checks that no other transaction has modified the same references. If a conflict is detected, the entire transaction is retried automatically.

The `Transaction` in `Exclude<R, Transaction>` refers to `Effect.Transaction`, an internal service that carries the transaction journal. `Effect.atomic` and `Effect.transaction` both satisfy and remove this requirement from the effect's environment. User code never needs to provide it manually.

Transactions are **composable**: nesting `Effect.atomic` inside another `Effect.atomic` joins the inner transaction into the outer one (single journal). If no outer transaction exists, `Effect.atomic` creates a new isolated one automatically. Use `Effect.transaction` instead when you always need a fresh isolated transaction even when nested.

```ts
import { Effect, TxRef } from "effect"

const program = Effect.gen(function*() {
  const ref1 = yield* TxRef.make(0)
  const ref2 = yield* TxRef.make(0)

  // All operations within atomic block succeed or fail together
  yield* Effect.atomic(Effect.gen(function*() {
    yield* TxRef.set(ref1, 10)
    yield* TxRef.set(ref2, 20)
    const sum = (yield* TxRef.get(ref1)) + (yield* TxRef.get(ref2))
    console.log(`Transaction sum: ${sum}`) // 30
  }))

  console.log(`Final ref1: ${yield* TxRef.get(ref1)}`) // 10
  console.log(`Final ref2: ${yield* TxRef.get(ref2)}`) // 20
})
```

### Effect.transaction (isolated)

```ts
export const transaction: <A, E, R>(
  effect: Effect<A, E, R>
) => Effect<A, E, Exclude<R, Transaction>>
```

Unlike `Effect.atomic`, `Effect.transaction` always creates a **new isolated** transaction boundary, even when nested inside another transaction. Parent and child transactions have separate journals and retry independently.

```ts
// Nested atomic composes with parent:
yield* Effect.atomic(Effect.gen(function*() {
  yield* TxRef.set(ref1, 10)
  yield* Effect.atomic(Effect.gen(function*() {
    yield* TxRef.set(ref1, 20) // Same journal as parent
  }))
}))

// Effect.transaction is always isolated:
yield* Effect.transaction(Effect.gen(function*() {
  yield* TxRef.set(ref2, 200) // Independent journal
}))
```

### Effect.atomicWith / Effect.transactionWith

These lower-level variants provide access to the raw transaction state:

```ts
export const atomicWith: <A, E, R>(
  f: (state: Transaction["Service"]) => Effect<A, E, R>
) => Effect<A, E, Exclude<R, Transaction>>

export const transactionWith: <A, E, R>(
  f: (state: Transaction["Service"]) => Effect<A, E, R>
) => Effect<A, E, Exclude<R, Transaction>>
```

`atomicWith` composes with an existing parent transaction (same journal), while `transactionWith` always creates an isolated transaction. The `Transaction["Service"]` type exposes `{ journal: Map<TxRef, ...>, retry: boolean }`. These are advanced APIs most users won't need directly.

### Effect.retryTransaction

```ts
export const retryTransaction: Effect<never, never, Transaction>
```

Signals that the current transaction should be retried. The runtime suspends the fiber and re-executes the transaction body when **any** `TxRef` that was read during the transaction changes.

```ts
import { Effect, TxRef } from "effect"

const program = Effect.gen(function*() {
  const ref = yield* TxRef.make(0)

  // Fork a fiber that increments ref every 100ms
  yield* Effect.forkChild(Effect.forever(
    TxRef.update(ref, (n) => n + 1).pipe(Effect.delay("100 millis"))
  ))

  // Retry until ref reaches 10
  yield* Effect.atomic(Effect.gen(function*() {
    const value = yield* TxRef.get(ref)
    if (value < 10) {
      return yield* Effect.retryTransaction
    }
    yield* Effect.log(`done with value: ${value}`)
  }))
})
```

### Retry Semantics

Transactions retry in three scenarios:

1. **Explicit retry** -- The body calls `Effect.retryTransaction`. The runtime suspends and wakes the fiber when any accessed `TxRef` changes.
2. **Conflict detection** -- Another transaction commits a change to a `TxRef` that this transaction read. The runtime detects a version mismatch and re-executes.
3. **Parent retry** -- If a child `Effect.atomic` is composed into a parent transaction, a retry on the parent causes the child to re-execute as well.

Internally, each `TxRef` carries a `version` counter. On commit, the runtime verifies that every ref in the journal still has the same version as when it was first read. If not, the journal is cleared and the body runs again.

**Implementation note:** `Effect.retryTransaction` works by setting an internal `retry` flag and then calling `Effect.interrupt` on the current fiber. The transaction runtime intercepts this interrupt, suspends the fiber until any accessed `TxRef` changes (via a callback registered in each ref's `pending` map), then re-executes the transaction body with a fresh journal.

---

## TxRef -- Transactional Reference

**Module:** `effect/TxRef`

The fundamental building block. A `TxRef<A>` holds a single mutable value that participates in transactions.

```ts
export interface TxRef<in out A> extends Pipeable {
  readonly [TypeId]: typeof TypeId
  version: number
  value: A
  pending: Map<unknown, () => void>
}
```

### Constructors

| Function | Signature |
|----------|-----------|
| `make` | `<A>(initial: A) => Effect<TxRef<A>>` |
| `makeUnsafe` | `<A>(initial: A) => TxRef<A>` |

### Operations

| Function | Signature | Description |
|----------|-----------|-------------|
| `get` | `(self: TxRef<A>) => Effect<A>` | Read the current value |
| `set` | `(self: TxRef<A>, value: A) => Effect<void>` | Write a new value |
| `update` | `(self: TxRef<A>, f: (a: A) => A) => Effect<void>` | Transform the value |
| `modify` | `(self: TxRef<A>, f: (a: A) => [R, A]) => Effect<R>` | Transform and return a derived value |

All operations are dual (support both data-first and data-last styles).

```ts
import { Effect, TxRef } from "effect"

const program = Effect.gen(function*() {
  const counter = yield* TxRef.make(0)

  yield* Effect.atomic(Effect.gen(function*() {
    const current = yield* TxRef.get(counter)
    yield* TxRef.set(counter, current + 1)
  }))

  console.log(yield* TxRef.get(counter)) // 1
})
```

Single-operation "transactions" do not require an explicit `Effect.atomic` wrapper because a single read or write is inherently atomic. Use `Effect.atomic` when you need multiple operations to commit together.

---

## TxHashMap -- Transactional Hash Map

**Module:** `effect/TxHashMap`

A transactional key-value map backed by an immutable `HashMap` inside a `TxRef`. Operations that mutate (like `set`, `remove`, `clear`) update the internal ref in place. Operations that transform (like `map`, `filter`) return a **new** `TxHashMap`.

```ts
export interface TxHashMap<in out K, in out V> extends Inspectable, Pipeable {
  readonly [TypeId]: typeof TypeId
  readonly ref: TxRef<HashMap<K, V>>
}
```

### Constructors

| Function | Signature |
|----------|-----------|
| `empty` | `<K, V>() => Effect<TxHashMap<K, V>>` |
| `make` | `<K, V>(...entries: Array<readonly [K, V]>) => Effect<TxHashMap<K, V>>` |
| `fromIterable` | `<K, V>(entries: Iterable<readonly [K, V]>) => Effect<TxHashMap<K, V>>` |

### Key Operations

| Function | Signature | Behavior |
|----------|-----------|----------|
| `get` | `(self, key) => Effect<Option<V>>` | Safe lookup |
| `set` | `(self, key, value) => Effect<void>` | Mutates in place |
| `has` | `(self, key) => Effect<boolean>` | Key existence |
| `remove` | `(self, key) => Effect<boolean>` | Mutates, returns whether key existed |
| `removeMany` | `(self, keys: Iterable<K>) => Effect<void>` | Remove multiple keys (mutates) |
| `clear` | `(self) => Effect<void>` | Remove all entries (mutates) |
| `isEmpty` | `(self) => Effect<boolean>` | Empty check |
| `isNonEmpty` | `(self) => Effect<boolean>` | Non-empty check |
| `size` | `(self) => Effect<number>` | Entry count |
| `modify` | `(self, key, f) => Effect<Option<V>>` | Update value at key if present; returns the **old** value wrapped in `Option` (or `Option.none()` if key absent) |
| `modifyAt` | `(self, key, f) => Effect<void>` | Upsert/delete via `Option`-returning function |
| `keys` | `(self) => Effect<Array<K>>` | All keys |
| `values` | `(self) => Effect<Array<V>>` | All values (alias: `toValues`) |
| `entries` | `(self) => Effect<Array<readonly [K, V]>>` | All key-value pairs (alias: `toEntries`) |
| `setMany` | `(self, entries: Iterable<readonly [K, V]>) => Effect<void>` | Bulk set (mutates) |
| `snapshot` | `(self) => Effect<HashMap<K, V>>` | Immutable snapshot |
| `union` | `(self, other: HashMap<K, V>) => Effect<void>` | Merge from plain HashMap (mutates) |
| `map` | `(self, f) => Effect<TxHashMap<K, B>>` | Returns new map |
| `filter` | `(self, pred) => Effect<TxHashMap<K, V>>` | Returns new map |
| `filterMap` | `(self, f) => Effect<TxHashMap<K, B>>` | Filter + transform, returns new map |
| `flatMap` | `(self, f) => Effect<TxHashMap<K, B>>` | Returns new merged map |
| `compact` | `(self: TxHashMap<K, Option<A>>) => Effect<TxHashMap<K, A>>` | Remove `None` values, returns new map |
| `reduce` | `(self, zero, f) => Effect<A>` | Fold |
| `forEach` | `(self, f: (V, K) => Effect<void, E, R>) => Effect<void, E, R>` | Side-effectful iteration |
| `some` / `every` / `hasBy` | `(self, pred) => Effect<boolean>` | Predicate checks |
| `findFirst` | `(self, pred) => Effect<[K, V] \| undefined>` | First matching entry |
| `getHash` | `(self, key, hash: number) => Effect<Option<V>>` | Lookup with precomputed hash |
| `hasHash` | `(self, key, hash: number) => Effect<boolean>` | Existence check with precomputed hash |
| `isTxHashMap` | `(value: unknown) => value is TxHashMap<K, V>` | Runtime type guard (not effectful) |

```ts
import { Effect, Option, TxHashMap } from "effect"

const program = Effect.gen(function*() {
  const txMap = yield* TxHashMap.make(["user1", "Alice"], ["user2", "Bob"])

  // Multi-step atomic operation
  yield* Effect.atomic(Effect.gen(function*() {
    const user = yield* TxHashMap.get(txMap, "user1")
    if (Option.isSome(user)) {
      yield* TxHashMap.set(txMap, "user1", user.value + "_updated")
      yield* TxHashMap.remove(txMap, "user2")
    }
  }))

  // After the atomic block: "user1" updated, "user2" removed
  const size = yield* TxHashMap.size(txMap)
  console.log(size) // 1
})
```

### Type-Level Utilities

```ts
type K = TxHashMap.TxHashMap.Key<typeof myMap>     // extracts key type
type V = TxHashMap.TxHashMap.Value<typeof myMap>   // extracts value type
type E = TxHashMap.TxHashMap.Entry<typeof myMap>   // readonly [K, V]
```

---

## TxHashSet -- Transactional Hash Set

**Module:** `effect/TxHashSet`

A transactional set of unique values backed by an immutable `HashSet` inside a `TxRef`.

**Mutation operations** (`add`, `remove`, `clear`) mutate the set in place. **Transform operations** (`union`, `intersection`, `difference`, `map`, `filter`) return a new `TxHashSet`.

```ts
export interface TxHashSet<in out V> extends Inspectable, Pipeable {
  readonly [TypeId]: typeof TypeId
  readonly ref: TxRef<HashSet<V>>
}
```

### Constructors

| Function | Signature |
|----------|-----------|
| `empty` | `<V>() => Effect<TxHashSet<V>>` |
| `make` | `<V>(...values: V[]) => Effect<TxHashSet<V>>` |
| `fromIterable` | `<V>(values: Iterable<V>) => Effect<TxHashSet<V>>` |
| `fromHashSet` | `<V>(hashSet: HashSet<V>) => Effect<TxHashSet<V>>` |

### Key Operations

| Function | Returns | Description |
|----------|---------|-------------|
| `add(self, value)` | `Effect<void>` | Add value (mutates) |
| `remove(self, value)` | `Effect<boolean>` | Remove value (mutates) |
| `has(self, value)` | `Effect<boolean>` | Membership check |
| `size(self)` | `Effect<number>` | Element count |
| `isEmpty(self)` | `Effect<boolean>` | Empty check |
| `clear(self)` | `Effect<void>` | Remove all (mutates) |
| `union(self, that)` | `Effect<TxHashSet<V>>` | New set with both |
| `intersection(self, that)` | `Effect<TxHashSet<V>>` | New set with common |
| `difference(self, that)` | `Effect<TxHashSet<V>>` | New set without that's elements |
| `isSubset(self, that)` | `Effect<boolean>` | Subset check |
| `map(self, f)` | `Effect<TxHashSet<U>>` | Transform values |
| `filter(self, pred)` | `Effect<TxHashSet<V>>` | Filter values |
| `some(self, pred)` | `Effect<boolean>` | True if any value satisfies predicate |
| `every(self, pred)` | `Effect<boolean>` | True if all values satisfy predicate |
| `reduce(self, zero, f)` | `Effect<U>` | Fold |
| `toHashSet(self)` | `Effect<HashSet<V>>` | Immutable snapshot |
| `isTxHashSet(value)` | `value is TxHashSet<unknown>` | Runtime type guard (not effectful) |

```ts
import { Effect, TxHashSet } from "effect"

const program = Effect.gen(function*() {
  const set1 = yield* TxHashSet.make("a", "b", "c")
  const set2 = yield* TxHashSet.make("b", "c", "d")

  const common = yield* TxHashSet.intersection(set1, set2)
  console.log(yield* TxHashSet.size(common)) // 2 ("b" and "c")
})
```

### Type-Level Utilities

```ts
type V = TxHashSet.TxHashSet.Value<typeof mySet>   // extracts value type
```

---

## TxQueue -- Transactional Queue

**Module:** `effect/TxQueue`

A transactional queue with four strategies and lifecycle management. Queues progress through states: **Open** -> **Closing** -> **Done**.

```ts
export interface TxQueue<in out A, in out E = never>
  extends TxEnqueue<A, E>, TxDequeue<A, E> {}
```

The error type parameter `E` allows queues to carry a typed error channel for failure signaling.

### Constructors (Strategies)

| Function | Description |
|----------|-------------|
| `bounded<A, E>(capacity)` | Blocks offer when full, blocks take when empty |
| `unbounded<A, E>()` | No capacity limit |
| `dropping<A, E>(capacity)` | Drops new items when full (returns `false`) |
| `sliding<A, E>(capacity)` | Evicts oldest items when full |

### Offer/Take Operations

| Function | Signature | Description |
|----------|-----------|-------------|
| `offer` | `(self, value) => Effect<boolean>` | Add item; returns false if rejected |
| `offerAll` | `(self, values) => Effect<Array<A>>` | Bulk add; returns rejected items |
| `take` | `(self) => Effect<A, E>` | Remove first item (blocks if empty) |
| `takeAll` | `(self) => Effect<NonEmptyArray<A>, E>` | Take all (blocks until non-empty) |
| `takeN` | `(self, n) => Effect<Array<A>, E>` | Take exactly N items |
| `takeBetween` | `(self, min, max) => Effect<Array<A>, E>` | Take between min and max items |
| `poll` | `(self) => Effect<Option<A>>` | Non-blocking take |
| `peek` | `(self) => Effect<A, E>` | View next without removing |

### Queue State

| Function | Signature |
|----------|-----------|
| `size` | `(self) => Effect<number>` |
| `isEmpty` | `(self) => Effect<boolean>` |
| `isFull` | `(self) => Effect<boolean>` |
| `isOpen` | `(self) => Effect<boolean>` |
| `isClosing` | `(self) => Effect<boolean>` |
| `isDone` | `(self) => Effect<boolean>` |

### Lifecycle

| Function | Signature | Description |
|----------|-----------|-------------|
| `fail(self, error)` | `Effect<boolean>` | Fail queue with typed error; clears all items and moves directly to Done state |
| `failCause(self, cause)` | `Effect<boolean>` | Fail with a `Cause`; if items remain, transitions through Closing first |
| `end(self)` | `Effect<boolean>` | Signal clean completion with `Cause.Done` (queue must accept `Cause.Done` in its error channel) |
| `interrupt(self)` | `Effect<boolean>` | Graceful close; transitions to Closing (existing items can still be drained) then Done |
| `shutdown(self)` | `Effect<boolean>` | Immediate clear + interrupt (clears items then interrupts) |
| `clear(self)` | `Effect<Array<A>, ExcludeDone<E>>` | Remove and return all items without changing queue state |
| `awaitCompletion(self)` | `Effect<void>` | Block until queue reaches Done state |
| `isShutdown(self)` | `Effect<boolean>` | Alias for `isDone` (legacy compatibility) |

Blocking operations (`take`, `takeAll`, `peek`, bounded `offer`) use `Effect.retryTransaction` internally, so the transaction automatically retries when items become available or space opens up.

```ts
import { Effect, TxQueue } from "effect"

const program = Effect.gen(function*() {
  const queue = yield* TxQueue.bounded<number>(10)
  yield* TxQueue.offerAll(queue, [1, 2, 3, 4, 5])

  // Take all items atomically
  const items = yield* TxQueue.takeAll(queue)
  console.log(items) // [1, 2, 3, 4, 5]

  // Sliding queue evicts oldest on overflow
  const sliding = yield* TxQueue.sliding<number>(2)
  yield* TxQueue.offer(sliding, 1)
  yield* TxQueue.offer(sliding, 2)
  yield* TxQueue.offer(sliding, 3) // evicts 1

  const first = yield* TxQueue.take(sliding)
  console.log(first) // 2
})
```

---

## TxSemaphore -- Transactional Semaphore

**Module:** `effect/TxSemaphore`

Manages a fixed number of permits using transactional semantics. When permits run out, `acquire` retries the transaction until permits become available.

```ts
export interface TxSemaphore extends Inspectable, Pipeable {
  readonly [TypeId]: typeof TypeId
  readonly permitsRef: TxRef<number>
  readonly capacity: number
}
```

### API

| Function | Signature | Description |
|----------|-----------|-------------|
| `make(permits)` | `Effect<TxSemaphore>` | Create with N permits |
| `acquire(self)` | `Effect<void>` | Take 1 permit (blocks if none) |
| `acquireN(self, n)` | `Effect<void>` | Take N permits (blocks if not enough) |
| `tryAcquire(self)` | `Effect<boolean>` | Non-blocking single acquire |
| `tryAcquireN(self, n)` | `Effect<boolean>` | Non-blocking N-permit acquire |
| `release(self)` | `Effect<void>` | Return 1 permit |
| `releaseN(self, n)` | `Effect<void>` | Return N permits |
| `available(self)` | `Effect<number>` | Current available permits |
| `capacity(self)` | `Effect<number>` | Total capacity |
| `withPermit(self, effect)` | `Effect<A, E, R>` | Bracket: acquire, run, release |
| `withPermits(self, n, effect)` | `Effect<A, E, R>` | Bracket with N permits |
| `withPermitScoped(self)` | `Effect<void, never, Scope>` | Scoped permit |
| `isTxSemaphore(value)` | `value is TxSemaphore` | Runtime type guard (not effectful) |

**Note:** `release` and `releaseN` are capped at the semaphore's `capacity` — releasing more permits than originally created will not exceed the initial capacity.

**Note:** `acquireN` and `releaseN` are **not dual** — they only support data-first style (`acquireN(self, n)`, `releaseN(self, n)`).

```ts
import { Effect, TxSemaphore } from "effect"

const program = Effect.gen(function*() {
  const sem = yield* TxSemaphore.make(3)

  // Safe bracket pattern
  const result = yield* TxSemaphore.withPermit(sem, Effect.gen(function*() {
    yield* Effect.log("working with permit")
    return 42
  }))

  // Non-blocking attempt
  const acquired = yield* TxSemaphore.tryAcquire(sem)
  console.log(acquired) // true (2 permits still available)
})
```

---

## TxChunk -- Transactional Chunk

**Module:** `effect/TxChunk`

A transactional ordered sequence backed by `TxRef<Chunk<A>>`. Used internally by `TxQueue` and available for direct use. All operations mutate in place.

```ts
export interface TxChunk<in out A> extends Inspectable, Pipeable {
  readonly [TypeId]: typeof TypeId
  readonly ref: TxRef<Chunk<A>>
}
```

### Constructors

| Function | Signature |
|----------|-----------|
| `empty<A>()` | `Effect<TxChunk<A>>` |
| `make(chunk: Chunk<A>)` | `Effect<TxChunk<A>>` |
| `fromIterable(iter: Iterable<A>)` | `Effect<TxChunk<A>>` |
| `makeUnsafe(ref: TxRef<Chunk<A>>)` | `TxChunk<A>` |

### Operations

| Function | Signature | Description |
|----------|-----------|-------------|
| `get(self)` | `Effect<Chunk<A>>` | Read current chunk |
| `set(self, chunk)` | `Effect<void>` | Replace entire chunk |
| `modify(self, f)` | `Effect<R>` | Transform and return derived value |
| `update(self, f)` | `Effect<void>` | Transform in place |
| `append(self, elem)` | `Effect<void>` | Add to end |
| `prepend(self, elem)` | `Effect<void>` | Add to beginning |
| `appendAll(self, chunk)` | `Effect<void>` | Concatenate Chunk at end |
| `prependAll(self, chunk)` | `Effect<void>` | Concatenate Chunk at beginning |
| `concat(self, other)` | `Effect<void>` | Concatenate another TxChunk |
| `take(self, n)` | `Effect<void>` | Keep first N elements |
| `drop(self, n)` | `Effect<void>` | Remove first N elements |
| `slice(self, start, end)` | `Effect<void>` | Keep elements in range |
| `map(self, f: (a: A) => A)` | `Effect<void>` | Transform elements in-place; `f` must return the same type `A` |
| `filter(self, pred)` | `Effect<void>` | Keep matching elements |
| `size(self)` | `Effect<number>` | Element count |
| `isEmpty(self)` | `Effect<boolean>` | Empty check |
| `isNonEmpty(self)` | `Effect<boolean>` | Non-empty check |

```ts
import { Chunk, Effect, TxChunk } from "effect"

const program = Effect.gen(function*() {
  const txChunk = yield* TxChunk.fromIterable([1, 2, 3])

  // Multi-step atomic modification
  yield* Effect.atomic(Effect.gen(function*() {
    yield* TxChunk.prepend(txChunk, 0)
    yield* TxChunk.append(txChunk, 4)
  }))

  const result = yield* TxChunk.get(txChunk)
  console.log(Chunk.toReadonlyArray(result)) // [0, 1, 2, 3, 4]
})
```

---

## When to Use Effect.atomic

Use `Effect.atomic` when you need **multiple operations on Tx\* data structures to be all-or-nothing**:

- **Bank transfer**: Debit one `TxRef`, credit another -- both or neither.
- **Read-modify-write**: Read a `TxHashMap`, compute a derived value, write it back -- no other transaction can interleave.
- **Conditional retry**: Read a value, check a condition, call `Effect.retryTransaction` if not met.

Single operations on `Tx*` structures do not need `Effect.atomic`. Each individual call (e.g., `TxRef.set`, `TxHashMap.get`) is already atomic on its own. The wrapper is only needed for composing multiple operations into a single atomic unit.

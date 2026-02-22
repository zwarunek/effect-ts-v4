# Data Types

Effect 4.0 provides a rich set of immutable data types for modeling values, collections, and domain concepts. This document covers the core data type modules.

---

## Option

**Module:** `effect/Option` | **Since:** 2.0.0

`Option<A>` is a discriminated union representing a value that may or may not exist: `None | Some<A>`. It replaces nullable types with a type-safe wrapper.

### Key Types

```ts
type Option<A> = None<A> | Some<A>

// None has _tag: "None"
// Some has _tag: "Some" and a .value property
```

### Constructors

```ts
import { Option } from "effect"

Option.some(42)              // Some<number>
Option.none<number>()        // None (singleton)

// From nullable values
Option.fromNullishOr(value)  // null | undefined -> None
Option.fromNullOr(value)     // null -> None
Option.fromUndefinedOr(value) // undefined -> None
```

### Transformers

```ts
// map: transform the inner value
Option.map(Option.some(1), (n) => n + 1)  // Some(2)

// flatMap: chain computations that may fail
Option.flatMap(Option.some(1), (n) =>
  n > 0 ? Option.some(n) : Option.none()
)

// andThen: like flatMap but also accepts raw values
Option.andThen(Option.some(1), 42)  // Some(42)
```

### Unwrapping

```ts
Option.getOrElse(opt, () => "default")  // unwrap with fallback
Option.getOrNull(opt)                    // A | null
Option.getOrUndefined(opt)              // A | undefined
Option.getOrThrow(opt)                  // throws if None
```

### Pattern Matching

```ts
Option.match(Option.some(1), {
  onNone: () => "empty",
  onSome: (v) => `value: ${v}`
})
// Output: "value: 1"
```

### Combining

```ts
// Combine multiple Options (all must be Some)
Option.all({ name: Option.some("Alice"), age: Option.some(30) })
// Some({ name: "Alice", age: 30 })

Option.all([Option.some(1), Option.none()])
// None

// Generator syntax
const result = Option.gen(function*() {
  const a = yield* Option.some(1)
  const b = yield* Option.some(2)
  return a + b
})
// Some(3)
```

### Guards

```ts
Option.isOption(value)  // type guard for Option<unknown>
Option.isNone(opt)      // narrows to None<A>
Option.isSome(opt)      // narrows to Some<A>
```

### Gotchas

- `Option.some(null)` is a valid `Some`. Use `fromNullishOr` to treat null/undefined as `None`.
- When yielded in `Effect.gen`, a `None` becomes a `NoSuchElementError`.
- `None` is a singleton; compare with `isNone`, not `===`.

---

## NullOr (NEW in v4)

**Module:** `effect/NullOr` | **Since:** 4.0.0

Allocation-free utilities for working with `A | null`, where `null` means "no value." Use instead of `Option` when `null` already models absence in your domain.

### When to Use

- `null` represents absence and `A` itself never includes `null`.
- You want zero-allocation overhead.
- You do not need to distinguish `None` from `Some(null)`.

### Core API

```ts
import { NullOr } from "effect"

// Transform
NullOr.map("hello", (s) => s.toUpperCase())  // "HELLO"
NullOr.map(null, (s) => s.toUpperCase())     // null

// Pattern match
NullOr.match(value, {
  onNull: () => "absent",
  onNotNull: (a) => `present: ${a}`
})

// Unwrap
NullOr.getOrThrow(value)                // throws if null
NullOr.getOrThrowWith(() => new MyError())(value)

// Lift throwable functions
const safeParse = NullOr.liftThrowable(JSON.parse)
safeParse("invalid")  // null (instead of throwing)
```

### Combiner / Reducer Integration

```ts
import { NullOr, Combiner, Reducer } from "effect"

const Sum = Combiner.make<number>((a, b) => a + b)

// Reducer: null + b = b, a + null = a, a + b = combine(a, b)
// initialValue is null
const reducer = NullOr.makeReducer(Sum)

// Fail-fast combiner: if either is null, result is null
const failFast = NullOr.makeCombinerFailFast(Sum)

// Fail-fast reducer: wraps an existing Reducer; any null short-circuits to null
const SumReducer = Reducer.make<number>((a, b) => a + b, 0)
const failFastReducer = NullOr.makeReducerFailFast(SumReducer)
```

---

## UndefinedOr (NEW in v4)

**Module:** `effect/UndefinedOr` | **Since:** 4.0.0

Mirror of `NullOr` but for `A | undefined`. Same API surface with `undefined` as the absence sentinel.

```ts
import { UndefinedOr } from "effect"

UndefinedOr.map(42, (n) => n * 2)          // 84
UndefinedOr.map(undefined, (n) => n * 2)   // undefined

UndefinedOr.match(value, {
  onUndefined: () => "absent",
  onDefined: (a) => `present: ${a}`
})

UndefinedOr.getOrThrow(value)
UndefinedOr.liftThrowable(riskyFn)

// Combiner/Reducer integration (same as NullOr)
UndefinedOr.makeReducer(combiner)
UndefinedOr.makeCombinerFailFast(combiner)
UndefinedOr.makeReducerFailFast(reducer)
```

---

## Combiner (NEW in v4)

**Module:** `effect/Combiner` | **Since:** 4.0.0

Replaces `Semigroup` from v3. A `Combiner<A>` wraps a single binary function `(self: A, that: A) => A` describing how two values merge. It carries no identity element (for that, use `Reducer`).

### Interface

```ts
interface Combiner<A> {
  readonly combine: (self: A, that: A) => A
}
```

### Constructors

```ts
import { Combiner, Number as Num } from "effect"

// From a custom function
const Sum = Combiner.make<number>((a, b) => a + b)
Sum.combine(3, 4)  // 7

// Built-in constructors
Combiner.min(Num.Order)    // picks the smaller value
Combiner.max(Num.Order)    // picks the larger value
Combiner.first<number>()   // always returns self (left)
Combiner.last<number>()    // always returns that (right)
Combiner.constant(0)       // ignores both, returns 0

// Flip argument order
Combiner.flip(stringConcat)  // combine("a","b") -> "ba"

// Intercalate (insert separator)
const csv = Combiner.intercalate(",")(stringCombiner)
csv.combine("a", "b")  // "a,b"
```

---

## Reducer (NEW in v4)

**Module:** `effect/Reducer` | **Since:** 4.0.0

Replaces `Monoid` from v3. A `Reducer<A>` extends `Combiner<A>` by adding an `initialValue` (identity element) and a `combineAll` method that folds an entire collection.

### Interface

```ts
interface Reducer<A> extends Combiner<A> {
  readonly initialValue: A
  readonly combineAll: (collection: Iterable<A>) => A
}
```

### Constructors

```ts
import { Reducer } from "effect"

const Sum = Reducer.make<number>((a, b) => a + b, 0)
Sum.combine(3, 4)         // 7
Sum.combineAll([1,2,3,4]) // 10
Sum.combineAll([])         // 0

// With custom short-circuiting combineAll
const Product = Reducer.make<number>(
  (a, b) => a * b,
  1,
  (coll) => {
    let acc = 1
    for (const n of coll) {
      if (n === 0) return 0
      acc *= n
    }
    return acc
  }
)

// Flip argument order (preserves initialValue)
Reducer.flip(Sum)
```

### Pre-built Reducers

Many modules export ready-made reducers:
- `Number.ReducerSum`, `Number.ReducerMultiply`
- `String.ReducerConcat`
- `Boolean.ReducerAnd`, `Boolean.ReducerOr`

---

## Array

**Module:** `effect/Array` | **Since:** 2.0.0

Immutable functional utilities for `Array<A>` and `NonEmptyReadonlyArray<A>`. All functions return new arrays; the input is never mutated. Most functions are dual (data-first and data-last).

### Key Types

```ts
type NonEmptyReadonlyArray<A> = readonly [A, ...Array<A>]
type NonEmptyArray<A> = [A, ...Array<A>]
```

### Constructors

```ts
import { Array } from "effect"

Array.make(1, 2, 3)         // [1, 2, 3] (NonEmptyArray)
Array.of(42)                 // [42]
Array.empty<number>()        // []
Array.range(1, 5)            // [1, 2, 3, 4, 5] (inclusive)
Array.fromIterable(set)      // from any iterable
Array.makeBy(5, (i) => i*2)  // [0, 2, 4, 6, 8]
Array.replicate(3, "x")      // ["x", "x", "x"]
```

### Access

```ts
Array.head(arr)    // Option<A> (safe)
Array.last(arr)    // Option<A>
Array.get(arr, 2)  // Option<A>
Array.tail(arr)    // Array<A> | undefined (not Option)
```

### Transform & Filter

```ts
Array.map(arr, (n) => n * 2)
Array.flatMap(arr, (n) => [n, n])
Array.filter(arr, (n) => n > 0)
Array.partition(arr, (n) => n > 0)  // [fails, passes]
Array.dedupe(arr)
Array.sort(arr, Order.Number)
```

### Fold & Search

```ts
Array.reduce(arr, 0, (acc, n) => acc + n)
Array.findFirst(arr, (n) => n > 10)  // Option<A>
Array.contains(arr, 42)
Array.join(arr, ", ")
```

### Set Operations

```ts
Array.union(a, b)
Array.intersection(a, b)
Array.difference(a, b)
```

### Gotchas

- `fromIterable` returns the original array reference if given an array; use `Array.copy` if you need a copy.
- `range(start, end)` is inclusive on both ends.
- `sort`, `reverse`, etc. always allocate a new array.

---

## Chunk

**Module:** `effect/Chunk` | **Since:** 2.0.0

Persistent indexed sequence optimized for append, prepend, and concatenation via a tree-like structure with structural sharing.

### Performance

| Operation       | Complexity           |
|----------------|----------------------|
| Append/Prepend | O(1) amortized       |
| Random Access  | O(log n)             |
| Concatenation  | O(log min(m, n))     |
| Iteration      | O(n)                 |

### Core API

```ts
import { Chunk } from "effect"

// Constructors
Chunk.make(1, 2, 3)
Chunk.empty<number>()
Chunk.fromIterable([1, 2, 3])
Chunk.range(1, 5)

// Operations
Chunk.append(chunk, 4)
Chunk.prepend(chunk, 0)
Chunk.appendAll(chunk1, chunk2)
Chunk.map(chunk, (n) => n * 2)
Chunk.filter(chunk, (n) => n > 2)
Chunk.reduce(chunk, 0, (acc, n) => acc + n)

// Conversion
Chunk.toArray(chunk)
Chunk.toReadonlyArray(chunk)
```

---

## Record

**Module:** `effect/Record` | **Since:** 2.0.0

Utilities for working with `Record<string, A>` (plain JS objects as key-value maps). All operations are immutable.

### Core API

```ts
import { Record } from "effect"

Record.empty<string, number>()
Record.fromEntries([["a", 1], ["b", 2]])

Record.get(record, "key")     // Option<A>
Record.has(record, "key")     // boolean
Record.set(record, "key", 42) // new record
Record.remove(record, "key")  // new record

Record.map(record, (v, k) => v * 2)
Record.filter(record, (v) => v > 0)
Record.reduce(record, init, fn)

Record.keys(record)    // Array<string>
Record.values(record)  // Array<A>
Record.toEntries(record)
Record.size(record)

// Set operations
Record.union(r1, r2, combiner)
Record.intersection(r1, r2, combiner)
```

---

## HashMap

**Module:** `effect/HashMap` | **Since:** 2.0.0

Immutable hash map using a Hash Array Mapped Trie (HAMT). Keys are compared by structural equality (`Equal.equals`).

```ts
import { HashMap, Option } from "effect"

const map = HashMap.make(["a", 1], ["b", 2], ["c", 3])

HashMap.get(map, "a")     // Option.some(1)
HashMap.get(map, "z")     // Option.none()
HashMap.has(map, "b")     // true
HashMap.size(map)          // 3

// Immutable updates
const updated = HashMap.set(map, "d", 4)
const removed = HashMap.remove(map, "a")

// Modify an existing value (only updates if key is present; callback receives the value directly, not Option)
HashMap.modify(map, "a", (v) => v + 1)

// modifyAt: upsert using Option — called with Some(current) or None if absent
HashMap.modifyAt(map, "a", (opt) =>
  Option.isSome(opt) ? Option.some(opt.value + 1) : Option.some(1)
)

// Iteration
HashMap.keys(map)     // IterableIterator<string>
HashMap.values(map)   // IterableIterator<number>
HashMap.entries(map)  // IterableIterator<[string, number]>

// Transform
HashMap.map(map, (v) => v * 2)
HashMap.filter(map, (v) => v > 1)
HashMap.reduce(map, 0, (acc, v) => acc + v)
HashMap.forEach(map, (v, k) => console.log(k, v))
```

### Type Utilities

```ts
type K = HashMap.HashMap.Key<typeof map>     // string
type V = HashMap.HashMap.Value<typeof map>   // number
type E = HashMap.HashMap.Entry<typeof map>   // [string, number]
```

---

## HashSet

**Module:** `effect/HashSet` | **Since:** 2.0.0

Immutable set backed by a `HashMap`. Elements are compared by structural equality.

```ts
import { HashSet } from "effect"

const set = HashSet.make("apple", "banana", "cherry")

HashSet.has(set, "apple")   // true
HashSet.size(set)            // 3

const added = HashSet.add(set, "grape")    // 4 elements
const removed = HashSet.remove(set, "banana") // 2 elements

// From iterable (deduplicates)
HashSet.fromIterable([1, 2, 2, 3])  // {1, 2, 3}

// Set operations
HashSet.union(set1, set2)
HashSet.intersection(set1, set2)
HashSet.difference(set1, set2)
```

---

## Data

**Module:** `effect/Data` | **Since:** 2.0.0

Value types with structural equality. `Data` classes provide immutable objects that compare by their contents rather than by reference.

### Data.Class

```ts
import { Data, Equal } from "effect"

class Person extends Data.Class<{ readonly name: string }> {}

const mike1 = new Person({ name: "Mike" })
const mike2 = new Person({ name: "Mike" })

Equal.equals(mike1, mike2)  // true (structural equality)
```

### Data.TaggedClass

Adds a `_tag` discriminant field automatically:

```ts
class Person extends Data.TaggedClass("Person")<{
  readonly name: string
}> {}

const p = new Person({ name: "Mike" })
p._tag  // "Person"
```

### TaggedEnum

Create discriminated unions with constructors:

```ts
import { Data } from "effect"

type HttpError = Data.TaggedEnum<{
  BadRequest: { readonly status: 400; readonly message: string }
  NotFound: { readonly status: 404; readonly message: string }
}>

const { BadRequest, NotFound, $is, $match } = Data.taggedEnum<HttpError>()

const err = BadRequest({ status: 400, message: "Invalid" })
err._tag  // "BadRequest"

// Type predicate
$is("BadRequest")(err)  // true

// Pattern matching
$match(err, {
  BadRequest: ({ message }) => `Bad: ${message}`,
  NotFound: ({ message }) => `Not found: ${message}`
})
```

---

## Brand

**Module:** `effect/Brand` | **Since:** 2.0.0

Branded/nominal types add a phantom type tag to primitive types, preventing accidental misuse.

### Type Alias

```ts
import { Brand } from "effect"

type UserId = number & Brand.Brand<"UserId">
type OrderId = number & Brand.Brand<"OrderId">
// UserId and OrderId are incompatible even though both are numbers
```

### Constructors

```ts
// No runtime validation (nominal only)
const UserId = Brand.nominal<UserId>()
const id = UserId(123)

// With validation
const PositiveInt = Brand.make<PositiveInt>((n) =>
  n > 0 ? undefined : "must be positive"
)

PositiveInt(5)         // 5 (as PositiveInt)
PositiveInt(-1)        // throws BrandError

PositiveInt.option(-1) // None
PositiveInt.result(-1) // Failure<BrandError>
PositiveInt.is(5)      // true (type guard)
```

### Combining Brands

```ts
const PositiveId = Brand.all(Positive, Integer)
// Validates both constraints
```

### BrandError (NEW in v4)

In v4, invalid brand construction returns a `BrandError` class (with `_tag: "BrandError"` and an `issue` property) instead of an array of errors.

---

## Match

**Module:** `effect/Match` | **Since:** 4.0.0

Type-safe pattern matching with exhaustiveness checking. Replaces verbose if/else or switch statements.

### Creating Matchers

```ts
import { Match } from "effect"

// From a type (returns a reusable function)
const matcher = Match.type<string | number>().pipe(
  Match.when(Match.number, (n) => `number: ${n}`),
  Match.when(Match.string, (s) => `string: ${s}`),
  Match.exhaustive
)
matcher(42)       // "number: 42"
matcher("hello")  // "string: hello"

// From a specific value (returns the result directly)
const result = Match.value(someInput).pipe(
  Match.when({ type: "user" }, (u) => u.name),
  Match.orElse(() => "unknown")
)
```

### Pattern Combinators

- `Match.when(pattern, handler)` -- positive match
- `Match.not(pattern, handler)` -- negative match (exclude)
- `Match.tag(tagName, handler)` -- match on `_tag` discriminant

### Built-in Predicates

```ts
Match.string    // matches string
Match.number    // matches number
Match.boolean   // matches boolean
Match.bigint    // matches bigint
Match.defined   // matches non-null, non-undefined
```

### Finalizers

- `Match.exhaustive` -- compile error if cases are not exhaustive
- `Match.orElse(fn)` -- catch-all fallback
- `Match.option` -- returns `Option<A>` (None if no match)
- `Match.result` -- returns `Result<A, Remaining>`

### Tag-based Matching (Discriminated Unions)

```ts
type Result =
  | { _tag: "Success"; data: string }
  | { _tag: "Error"; message: string }
  | { _tag: "Loading" }

// Match.typeTags -- creates a reusable function
const format = Match.typeTags<Result>()({
  Success: (r) => `Data: ${r.data}`,
  Error: (r) => `Error: ${r.message}`,
  Loading: () => "Loading..."
})

// Match.valueTags -- matches a specific value immediately
const msg = Match.valueTags(myResult, {
  Success: (r) => r.data,
  Error: (r) => r.message,
  Loading: () => "..."
})
```

---

## Equal

**Module:** `effect/Equal` | **Since:** 2.0.0

Deep structural equality for all value types.

### Interface

```ts
interface Equal extends Hash {
  [Equal.symbol](that: Equal): boolean
}
```

### Usage

```ts
import { Equal } from "effect"

Equal.equals(1, 1)                        // true
Equal.equals(NaN, NaN)                    // true (unlike ===)
Equal.equals({ a: 1 }, { a: 1 })         // true (structural)
Equal.equals([1, [2]], [1, [2]])          // true (deep)
Equal.equals(new Date("2023"), new Date("2023"))  // true

// Maps and Sets (order-independent)
Equal.equals(
  new Map([["a", 1], ["b", 2]]),
  new Map([["b", 2], ["a", 1]])
)  // true
```

### Custom Equality

```ts
class Coordinate implements Equal.Equal {
  constructor(readonly x: number, readonly y: number) {}

  [Equal.symbol](that: Equal.Equal): boolean {
    return that instanceof Coordinate &&
      this.x === that.x && this.y === that.y
  }

  [Hash.symbol](): number {
    return Hash.string(`${this.x},${this.y}`)
  }
}
```

### Critical: Immutability Requirement

Equality results are cached. Objects must be treated as immutable after first comparison. Mutating objects after `Equal.equals()` leads to stale cached results.

---

## Hash

**Module:** `effect/Hash` | **Since:** 2.0.0

Compute hash values for any type. Required companion to `Equal` for consistent equality semantics.

```ts
import { Hash } from "effect"

Hash.hash(42)                  // numeric hash
Hash.hash("hello")             // string hash
Hash.hash({ a: 1 })            // structural hash
Hash.hash([1, 2, 3])           // array hash

// Custom hashable type
class MyType implements Hash.Hash {
  [Hash.symbol](): number {
    return Hash.hash(this.value)
  }
}
```

Same immutability requirement as `Equal`: hash results are cached via `WeakMap`.

---

## Order

**Module:** `effect/Order` | **Since:** 2.0.0

Total ordering for comparing values. An `Order<A>` is a function `(a: A, b: A) => -1 | 0 | 1`.

### Built-in Orders

```ts
import { Order } from "effect"

Order.Number(5, 10)   // -1
Order.String("a","b") // -1
Order.Boolean          // false < true
Order.BigInt
Order.Date
```

### Custom Orders

```ts
const byAge = Order.make<Person>((a, b) =>
  a.age < b.age ? -1 : a.age > b.age ? 1 : 0
)

// Or derive from an existing order
const byAge = Order.mapInput(Order.Number, (p: Person) => p.age)
```

### Comparison Helpers

```ts
Order.isLessThan(Order.Number)(5, 10)          // true
Order.isGreaterThanOrEqualTo(Order.Number)(5, 5) // true
Order.min(Order.Number)(3, 7)                   // 3
Order.max(Order.Number)(3, 7)                   // 7
Order.clamp(Order.Number)({ minimum: 0, maximum: 100 })(150) // 100
Order.isBetween(Order.Number)({ minimum: 0, maximum: 10 })(5) // true
```

### Combining Orders

```ts
// Multi-criteria: sort by age then name
const byAgeThenName = Order.combine(
  Order.mapInput(Order.Number, (p: Person) => p.age),
  Order.mapInput(Order.String, (p: Person) => p.name)
)
```

### Gotchas

- `Order.Number` treats `NaN` as equal and less than any other number.
- `Order.make` uses `===` shortcut: if `self === that`, returns `0` without calling the compare function.

---

## Duration

**Module:** `effect/Duration` | **Since:** 2.0.0

Immutable time spans with nanosecond precision (via BigInt).

### Constructors

```ts
import { Duration } from "effect"

Duration.millis(500)
Duration.seconds(30)
Duration.minutes(5)
Duration.hours(2)
Duration.days(1)
Duration.weeks(1)
Duration.nanos(1000000n)
Duration.micros(1000n)

Duration.zero       // 0ms
Duration.infinity   // infinite duration

// From string (v4 Input type)
Duration.fromInputUnsafe("5 seconds")
Duration.fromInputUnsafe("100 millis")

// Safe parsing (returns undefined on failure, NEW in v4)
Duration.fromInput("5 seconds")    // Duration
Duration.fromInput("invalid" as any) // undefined
```

### Input Type

`Duration.Input` accepts multiple formats:

```ts
type Input =
  | Duration
  | number                        // milliseconds
  | bigint                        // nanoseconds
  | readonly [seconds: number, nanos: number]  // tuple (both are numbers, not bigint)
  | `${number} ${Unit}`           // string like "5 seconds"
```

### Conversions

```ts
Duration.toMillis(dur)    // number
Duration.toSeconds(dur)   // number
Duration.toNanos(dur)     // bigint | undefined
```

### Arithmetic & Comparison

```ts
Duration.add(a, b)
Duration.subtract(a, b)
Duration.times(dur, factor)       // multiply by a number (returns Duration)
Duration.divide(dur, divisor)     // returns Duration | undefined (undefined if divisor is 0 or non-finite)

Duration.Order       // Order<Duration>
Duration.min(a, b)
Duration.max(a, b)
Duration.clamp({ minimum, maximum })(dur)
Duration.between({ minimum, maximum })(dur)
```

---

## DateTime

**Module:** `effect/DateTime` | **Since:** 3.6.0

Points in time with optional time zone support.

### Types

```ts
type DateTime = Utc | Zoned

interface Utc {
  readonly _tag: "Utc"
  readonly epochMillis: number
}

interface Zoned {
  readonly _tag: "Zoned"
  readonly epochMillis: number
  readonly zone: TimeZone
}
```

### Constructors

```ts
import { DateTime } from "effect"

// Current time
DateTime.nowUnsafe()  // Utc

// From various inputs
DateTime.makeUnsafe(new Date())
DateTime.makeUnsafe("2024-01-01")
DateTime.makeUnsafe(1704067200000)
DateTime.makeUnsafe({ year: 2024, month: 1, day: 1 })

// Zoned
DateTime.makeZonedUnsafe(new Date(), {
  timeZone: "Europe/London"
})
```

### Input Type

`DateTime.Input` accepts: `DateTime | Partial<Parts> | Date | number | string`

### Parts

DateTime provides access to components via `PartsWithWeekday`:

```ts
interface PartsWithWeekday {
  millis: number
  seconds: number
  minutes: number
  hours: number
  day: number
  weekDay: number
  month: number
  year: number
}
```

### Time Units

```ts
type Unit =
  | "milli" | "millis"
  | "second" | "seconds"
  | "minute" | "minutes"
  | "hour" | "hours"
  | "day" | "days"
  | "week" | "weeks"
  | "month" | "months"
  | "year" | "years"
```

---

## Summary of v4 Changes

| What Changed | v3 | v4 |
|---|---|---|
| Semigroup | `Semigroup<A>` | `Combiner<A>` |
| Monoid | `Monoid<A>` | `Reducer<A>` |
| Nullable utilities | Use `Option.fromNullable` | `NullOr.map`, `NullOr.match`, etc. |
| Undefined utilities | Use `Option.fromNullable` | `UndefinedOr.map`, `UndefinedOr.match`, etc. |
| Brand errors | Array of `BrandErrors` | `BrandError` class with `.issue` |
| Match module | `@effect/match` package | Built into `effect/Match` (since 4.0) |
| Duration.fromInput | N/A | Returns `Duration \| undefined` (safe) |

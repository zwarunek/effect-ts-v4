# Schema: Validation, Parsing, and Serialization

> **Migration note**: `@effect/schema` was a separate package in Effect v3. In
> Effect 4.0 it is built into the core `effect` package. Import everything from
> `"effect"` -- there is no longer a `@effect/schema` import.

```ts
import { Schema, SchemaAST, SchemaIssue, SchemaParser, SchemaGetter, SchemaTransformation } from "effect"
```

---

## Core Type: `Schema<Type, Encoded, Context>`

Every schema carries three type parameters (though in everyday code only `Type`
is visible):

| Parameter | Meaning |
|-----------|---------|
| `Type` | The decoded (runtime) TypeScript type |
| `Encoded` | The wire / serialized representation |
| `Context` | Effect services required for parsing (default `never`) |

The public surface exposes several "views" of a schema:

```ts
// Only cares about the decoded type
interface Schema<out T> extends Top { Type: T }

// Tracks both decoded and encoded
interface Codec<out T, out E = T, out RD = never, out RE = never> extends Schema<T> {
  Encoded: E
  DecodingServices: RD
  EncodingServices: RE
}

// Decode-only view
interface Decoder<out T, out RD = never> extends Codec<T, unknown, RD, unknown> {}

// Encode-only view
interface Encoder<out E, out RE = never> extends Codec<unknown, E, unknown, RE> {}
```

Use `Schema.Type<S>` and `Codec.Encoded<S>` to extract the type-level
parameters from a schema value.

---

## Primitive Schemas

All primitives have `Type === Encoded` (no transformation needed):

```ts
Schema.String    // Schema<string>
Schema.Number    // Schema<number>
Schema.Boolean   // Schema<boolean>
Schema.BigInt    // Schema<bigint>
Schema.Symbol    // Schema<symbol>
Schema.Null      // Schema<null>
Schema.Undefined // Schema<undefined>
Schema.Void      // Schema<void>
Schema.Unknown   // Schema<unknown>
Schema.Any       // Schema<any>
Schema.Never     // Schema<never>
```

---

## Literals and Enums

```ts
// Single literal
const Admin = Schema.Literal("admin")          // Schema<"admin">

// Union of literals
const Role = Schema.Literals(["admin", "user", "guest"])
// Schema<"admin" | "user" | "guest">

// Transform one literal to another during decoding
const One = Schema.Literal(1).transform(true)
// Codec<true, 1> -- decodes 1 to true, encodes true to 1

// TypeScript enums
enum Fruit { Apple, Banana, Cherry }
const FruitSchema = Schema.Enum(Fruit)         // Schema<Fruit>
```

---

## Structs

`Schema.Struct` creates a schema for an object with known property names:

```ts
const Person = Schema.Struct({
  name: Schema.String,
  age: Schema.Number,
})
// Schema<{ readonly name: string; readonly age: number }>
```

### Optional and Default Fields

```ts
const Config = Schema.Struct({
  host: Schema.String,

  // Optional key -- the key may be absent in the encoded form
  port: Schema.optionalKey(Schema.Number),
  // { readonly port?: number | undefined }

  // Optional value -- the key is present but value may be undefined
  nickname: Schema.optional(Schema.String),
  // { readonly nickname: string | undefined }

  // Constructor default -- auto-filled when using makeUnsafe
  // defaultValue receives Option<undefined>; return Option.some(value) to supply a default
  role: Schema.String.pipe(
    Schema.withConstructorDefault((_input) => Option.some("user"))
  ),

  // Decoding default (key variant) -- the encoded key may be absent; fills in a default
  // withDecodingDefaultKey: encoded form has the key as optional (key may be missing)
  retries: Schema.Number.pipe(
    Schema.withDecodingDefaultKey(() => 3)
  ),

  // Decoding default (value variant) -- the encoded key may be absent OR undefined
  // withDecodingDefault: encoded form allows key absent OR undefined
  timeout: Schema.Number.pipe(
    Schema.withDecodingDefault(() => 30)
  ),
})
```

### Tagged Structs

```ts
const Circle = Schema.TaggedStruct("Circle", {
  radius: Schema.Number,
})
// Schema<{ readonly _tag: "Circle"; readonly radius: number }>

// Equivalent to:
Schema.Struct({
  _tag: Schema.tag("Circle"),  // auto-default in constructor
  radius: Schema.Number,
})
```

### Struct Manipulation

```ts
// Add fields to an existing struct
const PersonWithEmail = Person.mapFields((fields) => ({
  ...fields,
  email: Schema.String,
}))

// Or use the shorthand
const PersonWithEmail2 = Person.pipe(
  Schema.fieldsAssign({ email: Schema.String })
)

// Rename keys during encoding
const ApiPerson = Person.pipe(
  Schema.encodeKeys({ name: "full_name", age: "years_old" })
)
// Encoded: { full_name: string; years_old: number }
// Type:    { name: string; age: number }
```

---

## Records

```ts
// string keys, number values
const Scores = Schema.Record(Schema.String, Schema.Number)
// Schema<{ readonly [x: string]: number }>
```

---

## Tuples and Arrays

```ts
// Fixed-length tuple
const Point = Schema.Tuple([Schema.Number, Schema.Number])
// Schema<readonly [number, number]>

// Tuple with rest element
const AtLeastTwo = Schema.TupleWithRest(
  Schema.Tuple([Schema.String, Schema.String]),
  [Schema.String]
)
// Schema<readonly [string, string, ...string[]]>

// Homogeneous array
const Names = Schema.Array(Schema.String)
// Schema<readonly string[]>

// Non-empty array
const NonEmpty = Schema.NonEmptyArray(Schema.String)
// Schema<readonly [string, ...string[]]>

// Unique array (all elements distinct)
const Tags = Schema.UniqueArray(Schema.String)

// Mutable array (removes readonly)
const MutableNames = Schema.mutable(Schema.Array(Schema.String))
// Schema<string[]>
```

---

## Unions

```ts
const StringOrNumber = Schema.Union([Schema.String, Schema.Number])
// Schema<string | number>

// Nullable / optional shortcuts
const NullableString   = Schema.NullOr(Schema.String)    // string | null
const OptionalString   = Schema.UndefinedOr(Schema.String) // string | undefined
const NullishString    = Schema.NullishOr(Schema.String)  // string | null | undefined
```

### Discriminated (Tagged) Unions

```ts
const Shape = Schema.Union([
  Schema.TaggedStruct("Circle",    { radius: Schema.Number }),
  Schema.TaggedStruct("Rectangle", { width: Schema.Number, height: Schema.Number }),
])

// Shorthand: build a tagged union directly from a map of tag -> extra fields.
// Each entry becomes a TaggedStruct internally; the result has .match, .guards, .cases.
const Shape2 = Schema.TaggedUnion({
  Circle:    { radius: Schema.Number },
  Rectangle: { width: Schema.Number, height: Schema.Number },
})

// Or, enhance an existing Union with pattern-matching helpers:
const ShapeTagged = Shape.pipe(Schema.toTaggedUnion("_tag"))

// Curried pattern matching: returns a function that takes the value
const getArea = ShapeTagged.match({
  Circle:    (c) => Math.PI * c.radius ** 2,
  Rectangle: (r) => r.width * r.height,
})
// getArea(shape) => number

// Or pass value first, cases second for immediate result
const area = ShapeTagged.match(myShape, {
  Circle:    (c) => Math.PI * c.radius ** 2,
  Rectangle: (r) => r.width * r.height,
})
```

---

## Template Literals

```ts
const SemVer = Schema.TemplateLiteral([Schema.Number, ".", Schema.Number, ".", Schema.Number])
// Schema<`${number}.${number}.${number}`>
```

---

## Recursive Schemas

Use `Schema.suspend` to break circular references:

```ts
interface Category {
  readonly name: string
  readonly children: ReadonlyArray<Category>
}

const Category: Schema.Schema<Category> = Schema.Struct({
  name: Schema.String,
  children: Schema.Array(Schema.suspend(() => Category)),
})
```

---

## Checks and Filters

Check functions like `Schema.isNonEmpty()`, `Schema.isInt()`, etc. return `AST.Filter` objects.
They are used in two equivalent ways:

- `.check(...filters)` — method on any schema, accepts one or more filters directly
- `Schema.check(...filters)` — curried pipeable helper, for use inside `.pipe()`

```ts
// Method form (preferred for simple cases)
const Username = Schema.String.check(
  Schema.isNonEmpty(),
  Schema.isMinLength(3),
  Schema.isMaxLength(20),
  Schema.isPattern(/^[a-zA-Z0-9_]+$/)
)

const Age = Schema.Number.check(
  Schema.isInt(),
  Schema.isBetween({ minimum: 0, maximum: 150 })
)

// Pipe form (useful when chaining with other pipe operations)
const Age2 = Schema.Number.pipe(
  Schema.check(Schema.isInt(), Schema.isBetween({ minimum: 0, maximum: 150 })),
  Schema.brand("Age")
)
```

### Built-in String Checks

| Check | Description |
|-------|-------------|
| `isNonEmpty()` | At least 1 character |
| `isMinLength(n)` | Minimum length |
| `isMaxLength(n)` | Maximum length |
| `isLengthBetween(min, max)` | Length in range |
| `isPattern(regex)` | Matches regex |
| `isTrimmed()` | No leading/trailing whitespace |
| `isLowercased()` | All lowercase |
| `isUppercased()` | All uppercase |
| `isCapitalized()` | First char uppercase |
| `isUUID(version)` | UUID format; version is `1`–`8` or `undefined` (all versions) — you must explicitly pass `undefined` to allow any version |
| `isULID()` | ULID format |
| `isBase64()` | Base64 encoded |
| `isStartsWith(s)` | Starts with prefix |
| `isEndsWith(s)` | Ends with suffix |
| `isIncludes(s)` | Contains substring |

### Built-in Number Checks

| Check | Description |
|-------|-------------|
| `isInt()` | Safe integer (uses `Number.isSafeInteger`) |
| `isFinite()` | Not NaN / Infinity |
| `isGreaterThan(n)` | > n (exclusive) |
| `isGreaterThanOrEqualTo(n)` | >= n (inclusive) |
| `isLessThan(n)` | < n (exclusive) |
| `isLessThanOrEqualTo(n)` | <= n (inclusive) |
| `isBetween({ minimum, maximum })` | In range (inclusive by default; pass `exclusiveMinimum`/`exclusiveMaximum` to adjust) |
| `isMultipleOf(n)` | Divisible by n |

### Built-in Array / String Length Checks

The following checks work on any value with a `.length` property (strings and arrays):

| Check | Description |
|-------|-------------|
| `isNonEmpty()` | At least 1 element / character (alias for `isMinLength(1)`) |
| `isMinLength(n)` | Minimum length |
| `isMaxLength(n)` | Maximum length |
| `isLengthBetween(min, max)` | Length in range |

Array-only:

| Check | Description |
|-------|-------------|
| `isUnique()` | All elements unique (uses `Equal.equals` by default) |

### Custom Filters via `refine`

```ts
const Even = Schema.Number.pipe(
  Schema.refine(
    (n): n is number => n % 2 === 0,
    { expected: "an even number" }
  )
)
```

---

## Branded Types

Brands add nominal typing at the type level without runtime overhead:

```ts
const UserId = Schema.String.pipe(
  Schema.check(Schema.isUUID(4)),
  Schema.brand("UserId")
)
// Schema<string & Brand<"UserId">>

// Creating branded values (validates at runtime)
const id = UserId.makeUnsafe("550e8400-e29b-41d4-a716-446655440000")
// typeof id: string & Brand<"UserId">
```

You can integrate existing `Brand.Constructor` instances:

```ts
import { Brand } from "effect"

type PositiveInt = number & Brand.Brand<"PositiveInt">
const PositiveInt = Brand.nominal<PositiveInt>()

const PositiveIntSchema = Schema.Number.pipe(
  Schema.fromBrand("PositiveInt", PositiveInt)
)
```

---

## Built-in Composed Schemas

These schemas combine a primitive with checks or a transformation:

```ts
// String-based
Schema.NonEmptyString      // string with isNonEmpty
Schema.Trimmed             // string with isTrimmed
Schema.Char                // string of exactly length 1

// Number-based
Schema.Int                 // number with isInt
Schema.Finite              // number with isFinite

// Transformation schemas (Encoded !== Type)
Schema.NumberFromString    // Codec<number, string>  -- parses "42" to 42; rejects NaN/Infinity on decode
Schema.FiniteFromString    // Codec<number, string>  -- same but also rejects non-finite on encode
Schema.Trim                // Codec<string, string>  -- trims whitespace on decode; encode is passthrough
Schema.BooleanFromBit      // Codec<boolean, 0 | 1>  -- 0/1 to boolean
```

---

## Transformations: decodeTo, decode, encodeTo

### `decodeTo` -- full transformation between two schemas

This is the primary API for defining transformations. It is curried and used
with `.pipe()`:

```ts
import { Schema, SchemaGetter } from "effect"

// String -> Number transformation
const NumberFromString = Schema.String.pipe(
  Schema.decodeTo(
    Schema.Number,
    {
      decode: SchemaGetter.transform((s) => Number(s)),
      encode: SchemaGetter.transform((n) => String(n)),
    }
  )
)
// Codec<number, string>
// - decodes "123" to 123
// - encodes 123 to "123"
```

### `decode` -- same-schema transformation

When the encoded and decoded types are the same schema but you want to
transform the value:

```ts
const TrimmedString = Schema.String.pipe(
  Schema.decode({
    decode: SchemaGetter.transform((s) => s.trim()),
    encode: SchemaGetter.transform((s) => s.trim()),
  })
)
```

### Schema Composition (passthrough)

When `decodeTo` is called without a transformation, it performs schema
composition -- the "from" type side must match the "to" encoded side:

```ts
const IntFromString = Schema.NumberFromString.pipe(
  Schema.decodeTo(Schema.Int)
)
// Codec<number, string> -- decodes string to int
```

### SchemaTransformation Helpers

The `SchemaTransformation` module provides pre-built bidirectional
transformations. `transform` and `transformOrFail` take plain functions (not
`SchemaGetter.Getter` instances). The resulting `Transformation` object can
then be passed directly as the second argument to `Schema.decodeTo`:

```ts
import { Schema, SchemaTransformation } from "effect"

// transform: both decode and encode are pure (sync, infallible) functions
SchemaTransformation.transform({ decode: (e) => ..., encode: (t) => ... })

// transformOrFail: both sides return Effect<T|E, Issue>
SchemaTransformation.transformOrFail({ decode: (e, opts) => Effect..., encode: (t, opts) => Effect... })

SchemaTransformation.passthrough()      // Identity (no-op)
SchemaTransformation.trim()             // Trim strings on decode; passthrough on encode
SchemaTransformation.toLowerCase()      // Lowercase on decode; passthrough on encode
SchemaTransformation.toUpperCase()      // Uppercase on decode; passthrough on encode
SchemaTransformation.numberFromString   // string <-> number (same as Schema.NumberFromString)
SchemaTransformation.bigintFromString   // string <-> bigint
```

Transformations compose:

```ts
const trimAndLower = SchemaTransformation.trim().compose(
  SchemaTransformation.toLowerCase()
)
// decode: trim then lowercase; encode: passthrough
```

### SchemaGetter Primitives

`SchemaGetter.Getter` is the single-direction building block:

```ts
import { SchemaGetter } from "effect"

SchemaGetter.transform((s) => s.trim())       // Pure transform
SchemaGetter.transformOrFail((s) => ...)       // May fail with Issue
SchemaGetter.passthrough()                     // Identity
SchemaGetter.succeed(42)                       // Always returns 42
SchemaGetter.withDefault(() => "fallback")     // Fills in missing keys
SchemaGetter.omit()                            // Removes key from output
```

---

## Parsing and Validation API

All parsing functions exist in multiple variants:

| Suffix | Return type | Async? | Throws? |
|--------|-------------|--------|---------|
| `Sync` | `T` | No | Yes (throws on error) |
| `Effect` | `Effect<T, SchemaError, R>` | Yes | No |
| `Exit` | `Exit<T, SchemaError>` | No | No |
| `Option` | `Option<T>` | No | No |
| `Promise` | `Promise<T>` | Yes | Yes |

And two input signature variants:

| Prefix | Input type | Notes |
|--------|------------|-------|
| `decodeUnknown*` | `unknown` | Input type is `unknown`; full runtime type check |
| `decode*` | `S["Encoded"]` | TypeScript trusts the input is already the encoded type; runtime behavior is the same |

### Decoding (Encoded -> Type)

```ts
const Person = Schema.Struct({ name: Schema.String, age: Schema.Number })

// Synchronous -- throws on failure
const person = Schema.decodeUnknownSync(Person)({ name: "Alice", age: 30 })

// Effect-based -- returns Effect<Person, SchemaError>
const personEffect = Schema.decodeUnknownEffect(Person)({ name: "Alice", age: 30 })

// Exit-based -- returns Exit<Person, SchemaError>
const personExit = Schema.decodeUnknownExit(Person)({ name: "Alice", age: 30 })

// Option-based -- returns Option<Person>
const personOption = Schema.decodeUnknownOption(Person)({ name: "Alice", age: 30 })
```

### Encoding (Type -> Encoded)

```ts
// Schema.encodeSync, encodeEffect, encodeExit, encodeOption, encodePromise
const encoded = Schema.encodeSync(NumberFromString)(42)
// encoded: "42"
```

### Type Guards and Assertions

```ts
// Type guard -- narrows unknown to T
const isString = Schema.is(Schema.String)
if (isString(value)) {
  value.toUpperCase() // value is string
}

// Assertion -- throws if invalid
const assertString = Schema.asserts(Schema.String)
assertString(value) // value is string after this line
```

### makeUnsafe -- Construct Validated Values

```ts
const Person = Schema.Struct({
  name: Schema.NonEmptyString,
  age: Schema.Number.check(Schema.isBetween({ minimum: 0, maximum: 150 })),
  _tag: Schema.tag("Person"), // auto-filled
})

// _tag is auto-filled from the tag default
const alice = Person.makeUnsafe({ name: "Alice", age: 30 })
// { _tag: "Person", name: "Alice", age: 30 }
```

---

## Error Handling: SchemaError and SchemaIssue

Failed parsing operations produce a `SchemaError` wrapping an `Issue` tree:

```ts
try {
  Schema.decodeUnknownSync(Person)({ name: 42 })
} catch (e) {
  if (Schema.isSchemaError(e)) {
    console.log(e.issue.toString())
    // Human-readable validation error
  }
}
```

### Issue Types

`SchemaIssue.Issue` is a discriminated union (`_tag`):

| Tag | Description |
|-----|-------------|
| `InvalidType` | Wrong type (expected string, got number) |
| `InvalidValue` | Value fails a check/filter |
| `MissingKey` | Required property missing |
| `UnexpectedKey` | Unknown property present |
| `Forbidden` | Operation not allowed |
| `OneOf` | Ambiguous union match |
| `Filter` | Check/refinement failure (wraps inner issue) |
| `Encoding` | Transformation step failed (wraps inner issue) |
| `Pointer` | Adds property path context (wraps inner issue) |
| `Composite` | Multiple issues from a struct/array |
| `AnyOf` | Multiple union candidates failed |

---

## Standard Schema V1 Interop

Effect schemas conform to the [Standard Schema V1](https://standardschema.dev/)
specification for library interoperability:

```ts
const standardSchema = Schema.toStandardSchemaV1(Person)

// Use with any Standard Schema-compatible library
const result = standardSchema["~standard"].validate({ name: "Alice", age: 30 })
// { value: { name: "Alice", age: 30 } }
```

---

## JSON Schema Generation

Generate JSON Schema documents from Effect schemas:

```ts
const jsonSchemaDoc = Schema.toJsonSchemaDocument(Person)
// { dialect: "draft-2020-12", schema: {...}, definitions: {...} }
```

Options control output behavior:

```ts
Schema.toJsonSchemaDocument(Person, {
  additionalProperties: false,    // Disallow extra properties (default)
  generateDescriptions: true,     // Auto-generate descriptions from checks
})
```

The `JsonSchema` module supports converting between dialects:

```ts
import { JsonSchema } from "effect"

// Parse from any dialect into canonical draft-2020-12
const doc = JsonSchema.fromSchemaDraft07(rawJsonSchema)

// Convert to other dialects
const draft07  = JsonSchema.toDocumentDraft07(doc)
const oapi31   = JsonSchema.toMultiDocumentOpenApi3_1(multiDoc)
```

Supported dialects: `"draft-07"`, `"draft-2020-12"`, `"openapi-3.1"`,
`"openapi-3.0"`.

---

## Schema Representation (IR)

`SchemaRepresentation` is a serializable intermediate representation sitting
between the internal AST and external formats:

```ts
import { SchemaRepresentation } from "effect"

// Schema AST -> Document (serializable IR)
const doc = SchemaRepresentation.fromAST(Person.ast)

// Document -> JSON Schema
const jsonSchema = SchemaRepresentation.toJsonSchemaDocument(doc)

// Document -> runtime Schema (reconstruct)
const reconstructed = SchemaRepresentation.toSchema(doc)

// Document -> TypeScript code
const code = SchemaRepresentation.toCodeDocument(multiDoc)
```

---

## Optics: Immutable Data Updates (New in v4)

The `Optic` module provides composable optics for immutable data updates. Optics
integrate with Schema through `Schema.toIso`.

### Optic Types

| Optic | get | set | Description |
|-------|-----|-----|-------------|
| `Iso<S, A>` | `S -> A` | `A -> S` | Lossless bidirectional conversion |
| `Lens<S, A>` | `S -> A` | `(A, S) -> S` | Focus on a part of a whole |
| `Prism<S, A>` | `S -> Result<A>` | `A -> S` | Focus that may not match |
| `Optional<S, A>` | `S -> Result<A>` | `(A, S) -> Result<S>` | May fail both ways |
| `Traversal<S, A>` | Focus on many elements | | Array traversal |

All optics compose via `.compose()` and the result type is automatically
inferred (Iso + Iso = Iso, Lens + Lens = Lens, etc.).

### Constructors

```ts
import { Optic } from "effect"

// Iso -- lossless bidirectional
const celsiusToFahrenheit = Optic.makeIso(
  (c: number) => c * 9/5 + 32,  // get
  (f: number) => (f - 32) * 5/9  // set (reverse)
)

// Lens -- focus on a field
const nameLens = Optic.makeLens(
  (person: { name: string; age: number }) => person.name,
  (name, person) => ({ ...person, name })
)

// Prism -- partial focus
const somePrism = Optic.makePrism(
  (opt: Option<string>) =>
    Option.isSome(opt) ? Result.succeed(opt.value) : Result.fail("none"),
  (s) => Option.some(s)
)

// Identity
const id = Optic.id<string>()  // Iso<string, string>
```

### Optic Operations

```ts
// Get the focused value
nameLens.get(person)                    // "Alice"
nameLens.getResult(person)              // Result.succeed("Alice")

// Replace the focused value
nameLens.replace("Bob", person)         // { name: "Bob", age: 30 }

// Modify the focused value
nameLens.modify((name) => name.toUpperCase())(person)
// { name: "ALICE", age: 30 }
```

### Composing Optics

```ts
const addressLens = Optic.makeLens(
  (p: Person) => p.address,
  (addr, p) => ({ ...p, address: addr })
)

const cityLens = Optic.makeLens(
  (a: Address) => a.city,
  (city, a) => ({ ...a, city })
)

// Compose to focus on nested fields
const personCityLens = addressLens.compose(cityLens)
personCityLens.get(person)                    // "NYC"
personCityLens.replace("LA", person)          // deep immutable update
```

### Path-Based Navigation

Optics support convenient path-based access into structs:

```ts
// .key() for required keys (Lens)
const ageLens = Optic.id<Person>().key("age")

// .optionalKey() for optional keys
const nicknameLens = Optic.id<Person>().optionalKey("nickname")

// .tag() for discriminated union narrowing
const circlePrism = Optic.id<Shape>().tag("Circle")

// .pick() / .omit() for struct subsets
const nameAndAge = Optic.id<Person>().pick(["name", "age"])

// .forEach() for array traversal
const allNames = Optic.id<ReadonlyArray<Person>>()
  .forEach((iso) => iso.key("name"))
```

### Built-in Optics

```ts
Optic.id<S>()                        // Iso<S, S> -- identity
Optic.some<A>()                      // Prism<Option<A>, A>
Optic.none<A>()                      // Prism<Option<A>, undefined>
Optic.success<A, E>()                // Prism<Result<A, E>, A>
Optic.failure<A, E>()                // Prism<Result<A, E>, E>
Optic.entries<A>()                   // Iso<Record<string, A>, ReadonlyArray<[string, A]>>
Optic.fromChecks(...checks)          // Prism<T, T> from Schema checks
```

### Traversals

```ts
// A Traversal focuses on multiple elements (extends Optional<S, ReadonlyArray<A>>)
const allNames = Optic.id<ReadonlyArray<Person>>()
  .forEach((iso) => iso.key("name"))

// Modify all focused elements
allNames.modifyAll((name) => name.toUpperCase())(people)

// Get all focused elements as an array
Optic.getAll(allNames)(people) // ["Alice", "Bob", ...]
```

### Schema to Optic

```ts
// Convert a schema to an Iso between Type and its serializable (Iso) form
const personIso = Schema.toIso(Person)
// Iso<PersonType, PersonIsoForm>

// Identity iso on the source side
const sourceIso = Schema.toIsoSource(Person)
// Iso<PersonType, PersonType>
```

---

## SchemaAST: Inspecting Schema Structure

The `SchemaAST` module exposes the internal abstract syntax tree. Every schema
has an `.ast` property:

```ts
import { Schema, SchemaAST } from "effect"

const Person = Schema.Struct({ name: Schema.String, age: Schema.Number })
const ast = Person.ast

if (SchemaAST.isObjects(ast)) {
  console.log(ast.propertySignatures.map(ps => ps.name))
  // ["name", "age"]
}

// Get the type-side AST (strips encoding)
const typeAst = SchemaAST.toType(ast)

// Get the encoded-side AST
const encodedAst = SchemaAST.toEncoded(ast)

// Swap decode/encode directions
const flipped = SchemaAST.flip(ast)
```

AST node types: `Declaration`, `Null`, `Undefined`, `Void`, `Never`, `Unknown`,
`Any`, `String`, `Number`, `Boolean`, `BigInt`, `Symbol`, `Literal`,
`UniqueSymbol`, `ObjectKeyword`, `Enum`, `TemplateLiteral`, `Arrays`,
`Objects`, `Union`, `Suspend`.

---

## Declaring Custom Types

### Non-Parametric Types

```ts
const MyDate = Schema.declare(
  (u: unknown): u is Date => u instanceof Date,
  { expected: "a Date instance" }
)
```

### Parametric Types

```ts
const MySet = Schema.declareConstructor<ReadonlySet<unknown>, ReadonlySet<unknown>>()(
  [itemSchema],  // type parameters
  ([itemCodec]) => (input, ast, options) => {
    // parse logic returning Effect<ReadonlySet<T>, Issue>
  },
  { expected: "a Set" }
)
```

---

## Class-Based Schemas

Effect 4.0 provides `TaggedStruct` as the primary way to define data classes
with schemas. The `_tag` field serves as a discriminator:

```ts
const User = Schema.TaggedStruct("User", {
  id: Schema.String.pipe(Schema.check(Schema.isUUID(4)), Schema.brand("UserId")),
  name: Schema.NonEmptyString,
  email: Schema.String.check(Schema.isPattern(/@/)),
  createdAt: Schema.Date,
})

type User = Schema.Schema.Type<typeof User>
```

---

## Complete Example: API Validation

```ts
import { Schema } from "effect"

// Define domain schemas
const Email = Schema.String.pipe(
  Schema.check(Schema.isPattern(/^[^@]+@[^@]+\.[^@]+$/)),
  Schema.brand("Email")
)

const Age = Schema.Number.check(
  Schema.isInt(),
  Schema.isBetween({ minimum: 0, maximum: 150 })
)

const CreateUserRequest = Schema.Struct({
  email: Email,
  name: Schema.NonEmptyString,
  age: Age,
  role: Schema.String.pipe(
    Schema.withDecodingDefaultKey(() => "member")
  ),
})

// Decode from unknown input (e.g., request body)
const parseUser = Schema.decodeUnknownSync(CreateUserRequest)

try {
  const user = parseUser({ email: "alice@example.com", name: "Alice", age: 30 })
  // user.role === "member" (from default)
} catch (e) {
  if (Schema.isSchemaError(e)) {
    console.error(e.message)
  }
}

// Generate JSON Schema for documentation
const jsonSchema = Schema.toJsonSchemaDocument(CreateUserRequest)
```

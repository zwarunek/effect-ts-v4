---
name: effect-ts-v4
description: "Comprehensive Effect 4.0 (TypeScript) documentation for agents. Use when writing, reviewing, or refactoring code that imports from 'effect'. Covers Effect type, services/layers, Schema, Stream, concurrency, transactions, and the unstable ecosystem (HTTP, RPC, SQL, CLI, AI, Cluster). Triggers on: effect typescript, effect 4, effect-ts, Effect.gen, ServiceMap, Layer, Schema."
---

# Effect 4.0 Reference Skill

Comprehensive reference for building applications with Effect 4.0 (TypeScript). Use this skill when writing, reviewing, or refactoring code that uses the `effect` package.

## When to Load Docs

Use the decision tree below to determine which doc files to read. Always start with the most relevant file for the user's task. Read additional files only as needed.

## Doc Files

| File | Topics | Read When |
|------|--------|-----------|
| `docs/01-core-effect.md` | Effect type, pipe/flow, Effect.gen, Result, Exit, Cause, Scope, error handling | Writing or understanding core Effect code, error handling, resource management |
| `docs/02-services-layers.md` | ServiceMap.Service, Layer, Config, ConfigProvider, ManagedRuntime | Defining services, dependency injection, layer composition, configuration |
| `docs/03-data-types.md` | Option, Result, Array, Chunk, HashMap, HashSet, Duration, DateTime, Data, Brand, Order, Equivalence | Working with Effect data types, collections, domain modeling |
| `docs/04-concurrency.md` | Fiber, FiberHandle, FiberMap, FiberSet, Ref, SynchronizedRef, SubscriptionRef, Deferred, Queue, PubSub, Pool, Cache, ScopedCache, Semaphore, Latch, Schedule | Concurrency, parallelism, synchronization, queues, caching, scheduling |
| `docs/05-streaming.md` | Stream, Sink, Channel, Pull | Streaming data, pull-based processing, stream composition |
| `docs/06-schema.md` | Schema, SchemaAST, SchemaParser, validation, parsing, serialization | Data validation, parsing, encoding/decoding, API schemas |
| `docs/07-transactions.md` | TxRef, TxHashMap, TxHashSet, TxQueue, TxSemaphore, Effect.atomic | Software transactional memory, atomic operations, STM migration |
| `docs/08-patterns.md` | RcRef, RcMap, ScopedRef, LayerMap, ManagedRuntime, Match, composition patterns | Resource lifecycle, pattern matching, best practices, "which module should I use" |
| `docs/09-migration.md` | v3 to v4 migration, all breaking changes, rename tables | Migrating from Effect v3, understanding what changed |
| `docs/10-ecosystem.md` | @effect/platform-*, @effect/vitest, @effect/opentelemetry, testing, TestClock, **unstable modules**: HTTP, HTTP API, RPC, SQL, CLI, AI, Cluster, Workflow | Platform setup, testing, observability, HTTP servers/clients, database access, CLI tools, AI integration, distributed systems |

## Decision Tree

```
What is the user doing?
│
├─ Writing/reading core Effect code (Effect.gen, pipe, map, flatMap)
│  └─ Read: docs/01-core-effect.md
│
├─ Defining services, layers, or dependency injection
│  └─ Read: docs/02-services-layers.md
│
├─ Working with Option, Result, collections, or domain types
│  └─ Read: docs/03-data-types.md
│
├─ Concurrency (fibers, refs, queues, semaphores, scheduling)
│  └─ Read: docs/04-concurrency.md
│
├─ Streaming data (Stream, Sink, Channel)
│  └─ Read: docs/05-streaming.md
│
├─ Validation, parsing, or Schema
│  └─ Read: docs/06-schema.md
│
├─ Transactions or STM
│  └─ Read: docs/07-transactions.md
│
├─ Resource management patterns, pattern matching, "which module?"
│  └─ Read: docs/08-patterns.md
│
├─ Migrating from v3, understanding renames
│  └─ Read: docs/09-migration.md
│
├─ Testing, platform setup, ecosystem packages
│  └─ Read: docs/10-ecosystem.md
│
├─ HTTP server/client, API definitions, RPC, SQL, CLI, AI, Cluster, Workflow
│  └─ Read: docs/10-ecosystem.md (unstable modules section)
│
└─ Not sure / general question
   └─ Read: docs/01-core-effect.md, then others as needed
```

## Critical v4 Rules

When generating Effect 4.0 code, always follow these rules:

1. **Services use `ServiceMap.Service`**, not `Context.Tag` or `Effect.Tag`
2. **`Ref`, `Deferred`, `Fiber` are NOT Effect subtypes** -- use `Ref.get()`, `Deferred.await()`, `Fiber.join()`
3. **`Effect.fork` is now `Effect.forkChild`**, `Effect.forkDaemon` is now `Effect.forkDetach`
4. **`Effect.catchAll` is now `Effect.catch`**, `Effect.catchAllCause` is now `Effect.catchCause`
5. **`Either` is now `Result`** -- constructors: `Result.succeed(value)`, `Result.fail(error)`. Fields: `.success`, `.failure`. Tags: `"Success"`, `"Failure"`
6. **`FiberRef` is removed** -- use `ServiceMap.Reference` / `References.*`
7. **Schema is in core `effect` package** -- no `@effect/schema` import
8. **Layer naming convention**: use `.layer` not `.Default` or `.Live`
9. **Generator `this`**: use `Effect.gen({ self: this }, function*() { ... })` not `Effect.gen(this, function*() { ... })`
10. **Cause is flat**: `cause.reasons` array, not a recursive tree

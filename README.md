# Effect 4.0 Agent Skill

An agent skill for [Effect 4.0](https://github.com/Effect-TS/effect-smol) (TypeScript). Enables AI coding agents to understand and write Effect v4 code accurately.

## Install

```bash
npx skills add zwarunek/effect-skill
```

## Source Reference

These docs were written against **Effect 4.0 beta 9** (`effect@4.0.0-beta.9`), commit `7cb4f16653cc1394fa4329f8ffdf65f895f98f52` from [Effect-TS/effect-smol](https://github.com/Effect-TS/effect-smol).

## What's Covered

| File | Topics |
|------|--------|
| `docs/01-core-effect.md` | Effect type, pipe/flow, Effect.gen, Result, Exit, Cause, Scope |
| `docs/02-services-layers.md` | ServiceMap.Service, Layer, Config, ConfigProvider, ManagedRuntime |
| `docs/03-data-types.md` | Option, Array, Chunk, HashMap, HashSet, Duration, DateTime, Data, Brand |
| `docs/04-concurrency.md` | Fiber, Ref, Queue, PubSub, Deferred, Pool, Cache, Semaphore, Schedule |
| `docs/05-streaming.md` | Stream, Sink, Channel, Pull |
| `docs/06-schema.md` | Schema, SchemaAST, Optic, validation, parsing |
| `docs/07-transactions.md` | TxRef, TxHashMap, TxHashSet, TxQueue, Effect.atomic |
| `docs/08-patterns.md` | RcRef, RcMap, ScopedRef, LayerMap, Match, composition patterns |
| `docs/09-migration.md` | v3 to v4 migration, all breaking changes |
| `docs/10-ecosystem.md` | HTTP, RPC, SQL, CLI, AI, Cluster, testing, platform |

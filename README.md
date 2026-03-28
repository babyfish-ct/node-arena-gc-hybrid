# Hybrid Arena + GC Memory Model for JavaScript Runtimes

**Core idea**: Let GC learn business context via SPI. Rather than guessing which objects are short-lived, expose a minimal SPI that allows the hosting runtime to communicate request-scoped lifetime boundaries to the engine.

---

## The Problem

V8 has no idea where your request boundaries are. The 300 objects allocated for request #4821 will all be dead in 12ms — but V8 doesn't know that. It traces them repeatedly and collects them on its own schedule, causing p99 latency spikes that no amount of tuning truly eliminates.

---

## The Model

When an HTTP request arrives, Node creates an **Arena** bound to its async context. All objects land there via pointer-bump allocation. When the request ends, the entire Arena is freed in a single `free()` call — no tracing, no marking, no sweeping.

The GC exposes three SPI calls:

```c
Arena *createArena();
void releaseArena(Arena *arena);
void setContextArena(Arena *arena);
```

> The initial capacity of Arena is small (e.g., 1KB), grown dynamically via a linked chunk list as needed — bounded only by the total Arena Heap capacity.

Node's HTTP layer calls these. Application code calls nothing. `express`, `fastify`, `koa` — all existing code benefits without modification.

Object allocation rules:
- **Arena context active** → allocate in Arena
- **No Arena context** (non-Node, long-lived connections) or large objects → standard GC heap

---

## The Invariant

> Arena objects may reference GC objects. GC objects must never reference Arena objects.

Enforced by extending V8's existing write barrier with a single bitmask check:

```
on write: target[field] = value
  if address_of(value) in Arena range
  AND address_of(target) NOT in Arena range:
    promote(value)
```
---

## Promotion: The Real Cost

1. When a GC object holds a reference to an Arena object, `promote()` is triggered. This performs a DFS traversal of the escaping subgraph, moving all reachable Arena objects to the GC heap and updating their pointers.

2. Promoting 100 objects out of 10,000 still requires scanning the entire Arena to fix up pointers in the remaining 9,900 objects that may reference the promoted ones.

**Why it's not as bad as it sounds**:

Each request's Arena is a complete island. No cross-Arena references exist by design — request A's objects never reference request B's objects. The fixup scope is strictly bounded to a single request's Arena, not the entire heap or young generation.

In high-concurrency scenarios with tens of thousands of concurrent Arenas, each Arena's promotion is fully independent. There is no global stop-the-world pause — fixup work is distributed across requests, not aggregated into a single blocking event.

The net question is empirical: **how often do objects escape, and how large are the escaping subgraphs in typical HTTP workloads?** If escape is rare — and in well-structured applications it typically is — the Arena releases the vast majority of objects at zero tracing cost, and promotion handles the exceptions.

---

## This Is Generational GC, Reimagined

Classical generational GC applies the generational hypothesis without business context: collect a small nursery frequently, promote survivors. The young generation is small because the GC must collect eagerly to reclaim space.

This proposal gives the GC what it never had: **explicit lifetime boundaries from the business layer**. The result is a young generation sized exactly to the request, collected exactly once, at exactly the right moment, with zero tracing overhead for the common case.

---

## Bonus: Async JSON Serialization

Arena-rooted response objects are request-scoped and not shared across concurrent requests. This makes it safe to serialize them on a **libuv thread pool thread** instead of blocking the event loop.

If the I/O thread encounters a GC object reference during traversal, it writes a placeholder. The main thread patches only those fragments afterward. In the common case — a fully Arena-rooted response — `JSON.stringify` completes entirely off the main thread.

*(Speculative — thread safety details require careful analysis.)*

---

## What This Is Not

- Not a replacement for GC. GC handles everything that escapes.
- Not a developer API. Zero application code changes required.
- Not applicable to WebSockets, SSE, or long-lived contexts — those use the standard GC heap.

---

## The Hard Parts

- Write barrier must cover all reference write paths: JIT code, interpreter, native extensions, WeakMaps.
- Promotion DFS must be atomic with respect to JS execution.
- Dual heap requires careful virtual memory layout coordination with V8's existing allocator.
- DevTools memory profiler and heap snapshots need updating.

None are unsolvable. All require V8 team involvement.

---

**Original Documentation**: https://github.com/babyfish-ct/node-arena-gc-hybrid
**Copy at Node Discussion**: https://github.com/orgs/nodejs/discussions/5143

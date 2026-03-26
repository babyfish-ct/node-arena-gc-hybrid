# What If Node.js Could Have Zero-GC Request Handling — Without You Changing a Single Line of Code?

## A Proposal for a Hybrid Arena + GC Memory Model Baked into the JavaScript Runtime

---

## The Problem Nobody Talks About Enough

Node.js handles millions of requests per day in production systems worldwide. And yet, the GC pause problem never truly goes away.

You know the symptom: **p99 latency spikes**. Not p50, not p95 — but that brutal p99 tail that your SLA demands you fix, and that you can never quite eliminate. The culprit is almost always the same: V8's garbage collector kicking in at the worst possible moment, pausing your event loop, freezing your in-flight requests.

The standard advice? Tune `--max-old-space-size`. Reduce allocations. Use object pooling. Profile with `--trace-gc`.

These are band-aids. They don't address the root cause.

**The root cause is this: V8 has no idea where your request boundaries are.**

It doesn't know that the 300 objects you just allocated for request #4821 will all be dead in 12 milliseconds when the response is sent. It just sees a heap full of objects and runs its mark-and-sweep algorithm on its own schedule, indiscriminately.

What if the runtime *did* know? What if it could use that knowledge to make almost all of your request objects essentially free to allocate and free?

---

## The Idea: Request-Scoped Arena Allocation, Invisible to Developers

Here's the core proposal, stated simply:

When a short-lived request (e.g., HTTP request/response cycle) enters the Node.js runtime, the runtime automatically creates an Arena — a contiguous memory region — bound to that request's async context. All JavaScript objects allocated during that request's async scope are placed in this Arena. If the request is **lucky enough**, when it ends, all JavaScript objects are no longer useful, and the entire arena is released in a single operation. No GC needed.

What's discussed here is the lucky scenario; the unlucky scenario will be discussed later.

---

## The Architecture: Dual Heap with Address-Masked Write Barriers

**The Core Solution**: Rather than forcing the GC to guess which small fraction of objects are short-lived, let the GC expose an SPI — and let the Node layer use that SPI to tell the GC what is actually true in real-world business workloads: the majority of objects are short-lived. 

**The end goal**: business code remains completely unaware that any of this is happening.

### Two Heaps, Not One

The fundamental architectural choice is a **dual heap model**:

- **GC Heap**: The standard V8 managed heap. All long-lived objects live here. GC traces, marks, and sweeps this heap as normal.
- **Arena Heap**: A separate memory region, outside the GC heap's address range, used exclusively for Arena-allocated objects.

**heap membership is determined entirely by address range**. An object is an Arena object if and only if its address falls within the Arena Heap's address range. No object header modification. No layout disruption to V8's densely packed object format.

The GC exposes the following SPI:

```c
Arena *createArena();
void releaseArena(Arena *arena);
void setContextArena(Arena *arena);
```

- **createArena**: Allocates an Arena from the Arena heap. Initially small (e.g., 1KB), but supports dynamic growth via a linked chunk chain.
- **releaseArena**: Releases the Arena, all objects residing within it are automatically destroyed.
- **setContextArena**: Sets the Arena for the current main thread context.

The mechanism works as follows:

-   When Node begins processing a request, it creates an Arena. Once all async flows complete — regardless of success or failure — the Arena is released. Throughout this process, the Arena context is tracked by the associated `AsyncContext`.

-   When the main thread gains control, it sets the Arena context. Before yielding control, it sets it back to NULL.

-   Whenever the main thread creates a JavaScript object, the following rules apply:

    - Large objects ignore the Arena context entirely and are allocated directly on the GC heap — born as GC objects from the start.
    - If the current Arena context is NULL, the object is allocated directly on the GC heap — born as a GC object. This covers all scenarios where no one has called the SPI, especially non-Node environments.
    - If the current Arena context is non-NULL, memory is allocated from the current Arena, yielding an Arena object rather than a GC object.

### The Invariant

The scenario discussed earlier — where all JavaScript objects in the arena are no longer useful upon request completion — is a very lucky case. However, reality is not always so ideal.

One invariant governs the entire model:

> **Arena objects may reference GC objects. GC objects must never reference Arena objects.**

Arena objects holding references into the GC heap is perfectly safe — GC objects outlive the Arena, so those references are always valid. The dangerous direction is the reverse: a GC object referencing an Arena object would become a dangling pointer the moment the Arena is freed.

### Write Barrier via Address Masking

V8 already maintains write barriers for generational GC. This proposal extends the write barrier with a single, extremely cheap check:

```
on every reference write: target[field] = value

  if address_of(value) is in Arena Heap range:
    if address_of(target) is NOT in Arena Heap range:
      // GC object attempting to hold Arena reference → promote
      promote(value)
```

The address range check is implemented as a **bitmask operation** — one AND, one compare. This is effectively free in the context of a write barrier that already runs on every reference write. No hash lookups, no metadata tables, no object header reads.

When `promote(value)` is triggered, a depth-first traversal from `value` promotes all reachable Arena objects to the GC heap, ensuring the invariant is restored before the write completes.

The Write barrier is important. This `promote` mechanism is critical to prevent dangling pointers caused by premature Arena release.

### Arena-Tagged Allocation

When a short-lived async context is created (e.g., HTTP request), Node creates an Arena in the Arena Heap and associates it with that context via `AsyncContext`. All allocations within this context use pointer-bump allocation within the Arena. When no Arena context is active — including all non-Node environments and long-lived connection scenarios — allocation falls through to the standard GC heap.

### Request Completion = Arena Release

When the async context ends, the Arena is freed in a single operation — one `free()` call, potentially releasing thousands of objects simultaneously. No tracing, no marking, no sweeping.

---

## This Is Generational GC — Reimagined for Business Workloads

It is worth being explicit about what this model *is* at a deeper level.

**This is a variant of generational garbage collection.**

Classical generational GC is built on the *generational hypothesis*: most objects die young. The young generation is a small, fast-collected region; objects that survive enough GC cycles are promoted to the old generation.

This proposal takes the same hypothesis and applies it with **business domain knowledge** that the GC runtime has never had before:

- Classical young generation: small (a few MB), collected every few milliseconds, objects survive by outlasting GC cycles
- Arena generation: **as large as a request demands**, collected exactly once (at request end), objects survive by being explicitly promoted

The key insight is that for high-throughput HTTP workloads, the *request boundary* is a far more precise and meaningful lifetime boundary than anything a generic GC cycle can infer. Rather than letting the GC guess which objects are short-lived by watching allocation pressure, we tell the runtime exactly where the lifetime boundary is.

**The Arena is a young generation that perfectly matches the business workload's actual object lifetime distribution.** It is not a small, eagerly-collected nursery — it is a request-sized generation that is collected exactly once, with zero tracing overhead, at exactly the right moment.

---

## Why This Works: Most Objects Are Boring

The insight that makes this viable is embarrassingly simple:

**The vast majority of objects in a typical web request have a lifespan of exactly one request.**

Your ORM query result? Dead at response time. Your DTO? Dead at response time. Your middleware context object? Dead at response time. Your parsed JSON body? Dead at response time.

In a well-structured Node.js application, perhaps 95%+ of allocated objects never escape the request that created them. They're born, used, and should die together. But today's GC doesn't know this — so it treats them the same as long-lived objects, tracing them repeatedly before eventually collecting them.

With request-scoped Arenas, these objects cost almost nothing:

- **Allocation**: pointer bump. Near zero cost.
- **Deallocation**: the entire Arena is freed at once when the request ends. Near zero cost per object.
- **GC involvement**: zero. The GC never sees these objects at all.

The minority of objects that genuinely escape — items placed in a Redis-like in-memory cache, singleton services, WebSocket state — are promoted by the runtime when a cross-boundary write is detected, and the GC handles them normally.

---

## Invariants

The model maintains two critical invariants:

1. **No cross-heap references from GC to Arena**: After promotion, no GC-managed object may reference an Arena object. The write barrier enforces this by promoting entire reachable subgraphs when escape is detected.

2. **Arena objects are request-scoped**: All objects allocated within an Arena share exactly one lifetime — the request that created them. An Arena object cannot outlive its request without being explicitly promoted to the GC heap.

These invariants guarantee that when a request completes, the entire Arena can be freed in a single operation without risk of dangling pointers.

---

## Zero Developer Awareness Required

This is **not** a new allocator API for application developers. This is **not** a pragma or annotation. This is **not** a new language feature.

You write:

```javascript
app.get('/users/:id', async (req, res) => {
  const user = await db.findUser(req.params.id);  // Arena allocated
  const dto = transformUser(user);                 // Arena allocated
  const response = buildResponse(dto);             // Arena allocated
  res.json(response);
  // Request ends → Arena freed → all of the above gone instantly
});
```

Nothing changes. No `arena.alloc()`. No `defer arena.free()`. No lifetime annotations. Your existing code, your existing third-party libraries, your existing ORM — all of it silently benefits. `express`, `fastify`, `koa` — they all just work. Because this lives entirely inside the runtime.

---

## The `promote()` Case: When You Actually Need Global State

```javascript
const cache = new Map(); // GC-managed (module scope, no Arena)

app.get('/config', async (req, res) => {
  const config = await loadConfig(); // Arena allocated initially
  cache.set('config', config);       // Write barrier fires!
                                     // address_of(config) → Arena Heap range
                                     // address_of(cache entry) → GC Heap range
                                     // → promote(config) triggered
                                     // config and its entire object graph move to GC heap
  res.json(config);
});
```

The developer wrote nothing special. The write barrier detected the address range violation and promoted the object graph automatically. `config` is now GC-managed and lives until removed from `cache`. All other objects from this request still die with the Arena.

---

## Bonus: Async JSON Serialization via libuv Thread Pool

One additional optimization becomes possible under this model — and it addresses a long-standing Node.js pain point: **`JSON.stringify` blocking the main thread**.

Under this proposal, the runtime knows at serialization time whether the root object being serialized is a GC object or an Arena object. This distinction opens a new optimization path:

**Case A: GC root** — The object graph may contain long-lived references and shared state. Serialization proceeds on the main thread as today. No change.

**Case B: Arena root** — The object graph is request-scoped. It is, by construction, not shared with any other concurrent request. This makes it safe to serialize off the main thread.

When the serialization root is an Arena object, the runtime dispatches a serialization task to **libuv's thread pool**. The I/O thread walks the object graph and serializes it. If it encounters a reference to a GC object (which is valid — Arena objects may reference GC objects), it writes a **placeholder** in the output buffer and records the reference.

When the I/O thread completes, control returns to the main thread. If placeholders exist, the main thread serializes only those GC-rooted fragments and splices them into the output. If no placeholders exist, the result is ready immediately.

In the common case — a fully Arena-rooted response object with no GC references — `JSON.stringify` completes entirely off the main thread, with **zero main thread blocking time**.

This is particularly significant for large API responses: the serialization of a 500KB response object currently occupies the event loop for the entire duration. Under this model, that work moves to a background thread, keeping the event loop free to accept and begin processing the next request.

> **Note**: This optimization is proposed here as a direction worth exploring, not a fully specified design. The interaction between Arena lifetime, thread safety, and GC object references during concurrent serialization requires careful analysis. It is presented to illustrate the broader potential of the dual-heap model beyond GC pressure reduction.

---

## Edge Cases: When Arena Is Not Used

Not all asynchronous contexts are suitable for Arena allocation:

- **WebSockets and SSE**: Long-lived connections spanning minutes or hours bypass Arena allocation. The runtime detects these persistent contexts and allocates directly to the GC heap.
- **Streaming responses**: The runtime may create a fresh Arena per chunk rather than per request.
- **Nested request contexts**: When one request spawns another (e.g., internal `fetch`), the child inherits the parent's Arena by default.
- **Non-Node environments**: CLI tools, build scripts, long-running background workers — none of these set `Arena*`, so they see exactly the same behavior as today.

The decision is made at context creation time based on the async resource type. HTTP request/response is the primary beneficiary. Everything else is unchanged.

---

## Comparison With Existing Approaches

| Approach | Pros | Cons |
|----------|------|------|
| **Object pooling** | Reuses objects, reduces allocation | Requires developer effort; only works for specific types |
| **Reducing allocations** | Less GC pressure | Fights JavaScript's grain; makes code ugly |
| **Bun / JavaScriptCore** | Faster I/O, native code | Same GC model; doesn't solve the root problem |
| **Rust/Go rewrite** | Zero GC, predictable latency | Discards ecosystem and team knowledge |
| **This proposal** | Zero developer effort; all existing code benefits; solves the root cause | Requires V8 implementation work |

---

## The Hard Parts (Being Honest)

This is not a trivial change to V8. The challenges are real:

- **Dual heap address space management**: Reserving and managing a separate Arena Heap address range requires coordination with the OS allocator and V8's existing memory management. The address masking approach avoids object header changes but requires careful virtual memory layout.

- **Write barrier completeness**: The address-range write barrier must cover all reference write paths — compiled JIT code, interpreter, native extensions, WeakMaps, and internal V8 structures. Missing one path is a memory safety bug.

- **Promotion atomicity**: The DFS traversal that promotes an object graph must be atomic with respect to JavaScript execution. No interleaving GC or other operations should observe a partially promoted graph.

- **Tooling**: Chrome DevTools memory profiler, heap snapshots, and allocation trackers would need to understand the dual heap model.

None of these are unsolvable. But they require V8 team involvement — this is an RFC-level proposal, not a weekend project.

---

## The Philosophical Point

The systems programming world has known about Arena allocation for decades. Game engines use it. Compilers use it. High-performance C++ servers use it. Apache's `apr_pool_t` has been doing request-scoped Arena allocation since the early 2000s.

The conventional wisdom has been that managed languages can't have this because "you don't control allocation."

But that's only true if you think of the runtime as a black box that the language sits on top of. If the runtime *is* the implementation — if Node.js chooses to implement this — then the managed language gets all the ergonomics of GC *and* the performance characteristics of Arena allocation, for the common case.

**GC is not the enemy. GC doing work it doesn't need to do is the enemy.**

Most of your objects don't need GC. They need a bump allocator and a `free()` at request end. Let GC focus on the objects that actually need it.

---

## Conclusion

This proposal is not about replacing GC. It is about giving the runtime **semantic awareness of request boundaries** — awareness it could always have had, but never used — and using that awareness to implement a form of generational GC that is perfectly tuned to the actual object lifetime distribution of high-throughput HTTP workloads.

The result:

- **Arena objects**: ~95% of request allocations. Near-zero allocation cost. Near-zero deallocation cost. GC never touches them.
- **GC objects**: ~5% that genuinely escape. Handled normally.
- **Developer experience**: Unchanged. Zero new APIs to learn. Zero migration cost. All existing code benefits.
- **Async JSON serialization**: A path toward moving large response serialization off the main thread entirely.

**The best performance optimization is the one your users don't have to think about.**

---

[ChenTao](https://github.com/babyfish-ct)

March 26, 2026

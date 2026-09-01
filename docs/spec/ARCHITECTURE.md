<overview>

# Architecture

The internal architecture of Convoy, top down. This document is the overview; each mechanism has a
dedicated file under [`architecture/`](architecture/) that carries the full detail. Read this first,
then descend to the topic you need.

This specification is governed by the working law in [`../PHILOSOPHY.md`](../PHILOSOPHY.md) and
[`../CONVENTIONS.md`](../CONVENTIONS.md); it references that law rather than duplicating it.

</overview>

<problem-shape>

## The problem shape

One host. A producer generates large buffers (images, tensors) at high rate. Several consumer
processes must each read every buffer to keep the Jetson's GPU saturated. On the Jetson the CPU
and the integrated GPU share one physical memory, so a buffer that lives in shared memory is
directly readable by both without transfer. The binding constraint is the copy budget: at the
target rate and buffer size, copying each buffer even once per consumer would exhaust memory
bandwidth and starve the GPU. So the architecture is organized around one goal (**fill a buffer once, read it many
times with no copy**) and around failing loudly the instant that guarantee cannot be met.

</problem-shape>

<layers>

## The layers

Convoy is two cooperating pieces plus an optional store.

### The daemon (compiled core)

A single long-lived compiled process. It owns everything with a lifecycle:

- **Ring lifecycle**: allocation of ring segments and their control regions, and unlink on owner
  release. Centralizing this in the daemon means no binding-side process can leak or prematurely
  unlink a segment another process is still using.
- **The persistence workers**: the pool that services `save_slice` by reading buffer regions and
  writing them to the object store, plus the connection to that store.

The daemon holds almost no state. The control region it lays out is ephemeral (law 4), so the
daemon's own restart is cheap: re-initialize the control region from zero, reconnect the store, done.

Critically, the daemon is **out of the per-operation hot path.** It sets rings up and tears them
down, but the moment-to-moment producer/consumer rendezvous does not route through it.

### The bindings (thin client, loaded per process)

A thin library loaded into each producer and consumer process. It performs the per-operation work
directly against shared memory, with no round trip to the daemon:

- `attach` to a ring by name and hold the receipt.
- The `put` / `get` view surface into sectors.
- The counter operations (`signal` / `await` / `peek`) against the control region.
- The `save` / `retrieve` calls (which do reach the daemon's persistence path, off the hot loop).

The first binding is expected to be a scripting-language layer over the compiled core, because that
is where consumer code is most convenient to write; the surface is defined so that additional
language bindings can be added without changing the core.

### The object store (optional, pluggable, to the side)

A local object store, running as its own process, reached over the local network by the daemon's
persistence workers. It is **hidden behind `save` and `retrieve`**: no consumer knows its name,
holds its endpoint, or addresses it directly. It is swappable for any store that satisfies the
persistence contract (durable blobs, metadata, declarative size-based retention). It is entirely
optional: a pipeline that only moves buffers and never persists them does not run one.

</layers>

<through-line>

## The through-line: how a buffer travels

1. **Create.** The owner calls `create_ring(n, config)`. The daemon allocates `n` sectors and a
   control region and returns a handle. `config` fixes the buffer's static nature: sector size, and
   the retention policy for anything saved from it.
2. **Attach.** Each consumer calls `attach(name)` and receives a receipt granting view access.
3. **Produce.** The producer fills the current sector via `put` (or by writing through a `get` view)
   and calls `signal`: increment the counter, wake waiters.
4. **Consume.** Each consumer `await`s the counter, computes its sector as `counter mod n`, and takes
   a zero-copy `get` view. It checks staleness itself (`current - mine >= n` means it was lapped) and
   fails loud if it has fallen behind.
5. **Persist (optional).** A consumer that needs a durable record calls `save_slice(region, meta)`.
   The daemon's workers read that region directly and write it to the store. This is explicit in the
   loop, never fired implicitly by the counter.
6. **Retrieve (later, elsewhere).** Any process calls `retrieve(filter, projection, return_object)`
   to read persisted records back out of the store.
7. **Drop.** On shutdown each process `drop`s its handle; the owner's drop unlinks the segment.

Steps 3 and 4 (the hot path) touch only shared memory and one wait syscall. The daemon participates
in 1, 5, 6, and 7 only.

</through-line>

<cross-process-substrate>

## Cross-process substrate

The producer, the consumers, and the daemon must share one shared-memory region. When these run as
isolated containers, they must join a **shared IPC namespace** so they see the same `/dev/shm`; the
cross-process counter wait keys on the physical page, so a genuinely shared segment is required, not
two segments with the same name. Size the shared-memory mount for the worst case: all live rings and
all their sectors resident at once. Detail in
[`architecture/runtime-and-bindings.md`](architecture/runtime-and-bindings.md).

</cross-process-substrate>

<mechanism-files>

## The mechanism files

| File | Mechanism |
|------|-----------|
| [`architecture/rings-and-buffers.md`](architecture/rings-and-buffers.md) | The ring, sectors, the single buffer as a degenerate ring, and the `put`/`get` view surface. |
| [`architecture/ownership-and-receipts.md`](architecture/ownership-and-receipts.md) | Single-owner lifecycle, receipts, create/attach/drop semantics. |
| [`architecture/signaling.md`](architecture/signaling.md) | The waitable counter: its operations, the low-level wait primitive, and how consumers detect staleness. |
| [`architecture/persistence.md`](architecture/persistence.md) | `save_slice` / `retrieve`, the pluggable store, declarative retention, and why there is no delete. |
| [`architecture/runtime-and-bindings.md`](architecture/runtime-and-bindings.md) | The daemon/binding split, the hot-path boundary, and the cross-process IPC substrate. |
| [`architecture/failure-model.md`](architecture/failure-model.md) | Fail-loud/restart-fast in practice: named conditions, the ephemeral/durable split, restart-to-known-good. |

</mechanism-files>

<overview>

# Convoy

A high-performance shared-memory **data plane** for zero-copy movement of large image and
tensor buffers between processes on a single host.

Convoy targets NVIDIA Jetson (Orin-class): high-throughput edge pipelines where sensor bandwidth
outruns the copy budget and where the CPU and the integrated GPU share the same physical memory.
A small compiled core owns and
vends buffers by receipt; thin language bindings expose zero-copy views across the process
boundary. Persistence to a pluggable local object store is optional and sits off the hot path.

> **Status: design / pre-implementation.** This repository currently holds the project charter,
> philosophy, architecture, and decision records. No implementation code has been written yet.
> The documents here are the contract the implementation will be built against.

</overview>

<idea>

## The idea

A producer fills a buffer once. Any number of consumers read that same buffer as a view, with no
copy and no broker in the path. A monotonic counter in a small shared control region is the only
coordination: the producer increments it, consumers wait on it, and each consumer decides for
itself what the value means. That is the whole mechanism. Everything else is either sugar over it
or an optional durable record written to the side.

</idea>

<closed-primitives>

## The closed primitive set

Convoy exposes a **fixed, closed primitive set**. New capability comes from *composing*
them, never from adding to them (see [`docs/PHILOSOPHY.md`](docs/PHILOSOPHY.md)).

| Primitive | Role |
|-----------|------|
| `create_ring(n, config)` | Allocate a ring of `n` sectors plus its control region. The caller becomes the ring's single owner. |
| `attach(name)` | Map an existing ring as a consumer. |
| `drop` | Release a handle; unlink the ring if the caller is the owner. |
| waitable counter (`signal` / `await` / `peek`) | The per-ring monotonic counter in the control region. One primitive, three operations: wake waiters after an increment, block until it changes, read it without blocking. |
| `save_slice(region, metadata)` | Persist a region of a buffer to the object store, reading the buffer directly. |
| `retrieve(filter, projection, return_object)` | The single read path out of the store: select records by filter, choose which descriptor fields return, and gate the payload blob. |

**Buffer access** is mediated by two view operations, not a raw memory handle:
`get(region) -> view` returns a zero-copy view into a sector, and `put(region, data)` writes
through into it. A view is never a copy.

**Sugar** (built entirely on the closed set): `create_buffer()` is `create_ring(1)`; `save(buffer, meta)`
is `save_slice` over the whole buffer; `query` / `get` / `list` are `retrieve` with different
projection and blob choices.

</closed-primitives>

<architecture>

## Architecture in one paragraph

A compiled daemon owns ring lifecycle (allocation, the control region layout, unlink) and runs the
persistence workers. A thin binding library, loaded into each producer and consumer process,
performs the per-operation work: attach, the `put`/`get` view surface, the counter operations, and
the `save`/`retrieve` calls. The daemon stays **out of the per-operation hot path**: producer and
consumers rendezvous directly through the shared control region, a handful of instructions and one
wait syscall. Coordination state (the control region) is ephemeral and rebuilt from zero on every
start; the only durable state is whatever is written to the object store. See
[`docs/spec/ARCHITECTURE.md`](docs/spec/ARCHITECTURE.md).

</architecture>

<design-law>

## Design law

Five rules govern every decision in this project. They are argued in
[`docs/PHILOSOPHY.md`](docs/PHILOSOPHY.md) and enforced as contribution gates in
[`CONTRIBUTING.md`](CONTRIBUTING.md).

1. **Closed primitives.** A small, fixed, fully testable verb set. Compose, never extend.
2. **Single-owner, receipt-based ownership.** Every buffer has exactly one owner; access is granted
   by an explicit receipt. No orphaned or non-deterministic ownership.
3. **Zero-copy by default.** Data is viewed across the process boundary, not copied.
4. **Fail loud, restart fast.** Any deviation from a known-good state is a crash, not a degraded
   limp. Coordination state is ephemeral and rebuildable; only data is durable.
5. **Determinism boundary.** Mechanical operations are deterministic and independently testable.
   No hidden global state.

</design-law>

<repository-layout>

## Repository layout

| Path | What |
|------|------|
| `README.md` | This file. |
| `docs/PHILOSOPHY.md` | The design law, argued. The project's constitution. |
| `docs/CONVENTIONS.md` | Repository, code, and documentation conventions. |
| `docs/spec/ARCHITECTURE.md` | The internal architecture overview. |
| `docs/spec/architecture/` | Per-mechanism detail: rings, ownership, signaling, persistence, runtime, failure model. |
| `docs/spec/decisions/` | Architecture Decision Records. |
| `CONTRIBUTING.md` | How to contribute, with the closed-primitives rule as a hard gate. |
| `LICENSE` | Apache-2.0. |

</repository-layout>

<license>

## License

Apache License 2.0. See [`LICENSE`](LICENSE).

</license>

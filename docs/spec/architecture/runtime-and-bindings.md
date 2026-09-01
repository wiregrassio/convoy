<overview>

# Runtime and bindings

How Convoy is split across processes at runtime: a compiled daemon that owns lifecycle and
persistence, thin bindings loaded into each producer and consumer, and the shared-memory substrate
they rendezvous through. The organizing rule is that the daemon stays out of the per-operation hot
path.

</overview>

<daemon-binding-split>

## The daemon / binding split

| Layer | Owns |
|-------|------|
| **Compiled daemon** | Ring lifecycle (allocate, lay out the control region, unlink), and the persistence workers plus the object-store connection. |
| **Thin binding** (loaded per process) | `attach`; the `put`/`get` view surface; the counter operations (`signal`/`await`/`peek`); the `save`/`retrieve` calls. |

The daemon is a single long-lived process. The binding is a library linked into every producer and
consumer. The division is deliberate: the daemon holds the things that have a lifecycle and must be
owned by exactly one long-lived process (segments, the store connection), and the binding holds the
per-operation mechanics that must run with no extra process hop.

### The hot-path boundary

The per-operation path (a producer filling a sector and signaling, a consumer waiting and taking a
view) touches **only shared memory and one wait syscall**. It does not call into the daemon. Producer
and consumers rendezvous directly through the shared control region (see [`signaling.md`](signaling.md)).

This is the single most important runtime property. If the daemon mediated every operation, it would
insert an inter-process round trip into the hottest path in the system, exactly where the copy budget
is already tight. Confining the daemon to setup, teardown, and off-loop persistence lets the hot path
be a handful of instructions and a wait. The daemon participates only in `create_ring`, `drop`
(unlink), and the `save`/`retrieve` persistence path, none of which sit in the per-unit loop.

### Why a compiled core with thin bindings

The core is compiled for predictable timing: no garbage-collector pauses, no interpreter-wide lock, no
managed-runtime jitter on the path that must keep pace with a high-rate producer. The bindings are thin
and live in whatever language the consumers are most naturally written in, because consumer code is
where ergonomics matter most and hard real-time timing matters least. Keeping the timing-critical core
compiled and the ergonomic surface thin gives both properties without compromise. Concentrating segment
lifecycle in the compiled daemon also removes a whole class of teardown bug: because a binding process
no longer owns the segment, it cannot unlink a segment that is still in use when it exits (see
[`ownership-and-receipts.md`](ownership-and-receipts.md)).

</daemon-binding-split>

<shared-memory-substrate>

## The shared-memory substrate

All three roles (producer, consumers, daemon) must map the **same** shared-memory region. The
cross-process counter wait keys on the physical memory page, so they need a genuinely shared segment,
not two identically named segments in separate namespaces.

On the Jetson the CPU and the integrated GPU share one physical memory, so a buffer in `/dev/shm` is
directly readable by the GPU with no transfer. That unified memory is what makes the zero-copy read
surface possible at all.

When the roles run as isolated containers, they must join a **shared IPC namespace** so that they see a
common shared-memory mount (`/dev/shm`). The typical arrangement is: the daemon's container exposes a
shareable IPC namespace, and each consumer container joins the daemon's namespace rather than the
host's. Scoping a private IPC namespace to the Convoy processes shares the segment with exactly the
processes that need it, instead of exposing it on (and risking a name collision with) the whole host's
shared memory.

**Sizing.** Size the shared-memory mount for the worst case: every live ring and all of its sectors
resident simultaneously. The segments are large (they hold full image or tensor buffers), so this is a
real budget to compute, not a default to accept. Undersizing it turns a normal allocation into a
failure.

</shared-memory-substrate>

<restart>

## What restart looks like

The daemon holds almost no state, so its restart is cheap and total. On start it lays out fresh control
regions (initialized from zero, nothing is reconciled) and reconnects to the object store. There is no
coordination state to recover because coordination state is ephemeral by design; the only durable state
lives in the store and is reached the same way after a restart as before. This is the ephemeral/durable
split (see [`failure-model.md`](failure-model.md)) expressed in the runtime: the process you restart is
nearly stateless, which is what makes restart-to-known-good a real recovery strategy rather than a hope.

</restart>

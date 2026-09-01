<overview>

# Ownership and receipts

Every ring has exactly one owner and a set of consumers holding receipts. This is law 2 of the design
(see [`../../PHILOSOPHY.md`](../../PHILOSOPHY.md)) and it is what makes lifecycle deterministic across
processes.

</overview>

<owner-and-consumers>

## Owner and consumers

- **The owner** is the process that called `create_ring`. It is the sole process responsible for the
  segment's existence. There is exactly one owner per ring, fixed at creation, never transferred.
- **A consumer** is a process that called `attach`. It holds a **receipt**, a handle that grants
  access to the ring (a view surface, the counter operations) but confers no ownership. A ring may
  have any number of consumers.

The asymmetry is the point. Ownership is a single, unambiguous responsibility; access is freely
shared. Nothing about having a receipt makes a consumer responsible for the segment's lifecycle.

</owner-and-consumers>

<lifecycle-primitives>

## The three lifecycle primitives

### `create_ring(n, config)`: become the owner

Allocates the segment (sectors plus control region) and returns the owner's handle. The `config`
carries the static properties of the ring: sector size and the retention policy applied to anything
saved from it (see [`persistence.md`](persistence.md)). Static properties belong at creation because
they describe what the ring *is* and do not change over its life; putting them here keeps them out of
the recurring per-operation calls, where they would be noise.

### `attach(name)`: obtain a receipt

Maps an existing ring by name into the calling process and returns a consumer receipt. The consumer
can now take views and operate the counter. It cannot unlink the segment.

### `drop`: release a handle

Releases the caller's handle. For a **consumer**, `drop` detaches its mapping and nothing more. For
the **owner**, `drop` additionally **unlinks** the segment, removing it. Because only the owner's drop
unlinks, the segment's existence is controlled by exactly one process throughout its life.

</lifecycle-primitives>

<centralize-unlink>

## Why centralize unlink in the owner (and the daemon)

The lifecycle-owning role is held by the long-lived daemon in practice: it is the process that calls
`create_ring` on behalf of the system and outlives any individual consumer. This closes a class of
bug found in designs where consumers can unlink shared segments. If a consumer process is allowed to
unlink on its own exit, then a consumer crashing or exiting can pull a live segment out from under
producers and other consumers that are still using it, and, worse, whether that happens depends on
process exit ordering, which is non-deterministic. Concentrating unlink in the single owner removes
the ordering dependency entirely: consumers come and go, attaching and dropping receipts, and the
segment persists exactly as long as its owner holds it.

</centralize-unlink>

<receipts-not-locks>

## Why receipts instead of shared locks

The receipt grants access without granting a lock that crosses the process boundary. This matters for
the failure model. A cross-process lock is a resource a consumer can hold at the moment it dies, and
recovering a segment from a dead lock-holder is a reconciliation path (detect the dead holder, decide
whether its partial work is valid, release or repair) that the fail-loud/restart-fast law forbids
(see [`failure-model.md`](failure-model.md)). A receipt holds no such lock. A consumer that dies
simply stops calling the counter; there is nothing held, nothing to reconcile, nothing to recover. The
coordination that crosses the boundary (the counter) carries no ownership and no lock by design, see
[`signaling.md`](signaling.md).

</receipts-not-locks>

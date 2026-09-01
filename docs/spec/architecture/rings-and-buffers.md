<overview>

# Rings and buffers

The unit Convoy manages is a **ring**: `n` fixed-size sectors plus a small control region, allocated
as one shared-memory segment. A producer writes into sectors in sequence; consumers read them. The
single buffer is not a separate concept: it is a ring of one sector.

</overview>

<sectors-and-ring>

## Sectors and the ring

`create_ring(n, config)` allocates `n` sectors of the size given in `config`, contiguous in one
segment, alongside a control region (the counter and ring bookkeeping, see
[`signaling.md`](signaling.md)). Sector size is fixed at creation because it is part of what the
buffer *is*: a ring is a homogeneous sequence of equally sized slots, and a consumer computes a
sector's location arithmetically from its index. Variable-size sectors would make that arithmetic
impossible and reintroduce an allocator on the hot path.

The producer advances through sectors in order. After filling sector `k` it signals; the next fill
targets sector `k+1`, wrapping at `n`. A consumer that has processed up to counter value `m` reads
sector `m mod n`. Because indices wrap, sector reuse is inherent: sector `k mod n` will be
overwritten again `n` signals later. That reuse is the entire reason the staleness check exists (see
below and [`signaling.md`](signaling.md)).

</sectors-and-ring>

<degenerate-ring>

## The single buffer is a degenerate ring

`create_buffer()` is sugar for `create_ring(1)`. A one-sector ring is a plain shared buffer: there is
one slot, the index is always zero and is elided from the sugar surface. Crucially, the **failure
rule is identical** at `n = 1` and `n > 1`. The rule is "fail if the producer advances past what a
consumer is still reading," which at `n > 1` means "fail on a full lap" (the producer wrapped all the
way around and caught the consumer) and at `n = 1` means "fail on any advance while a read is live"
(the single slot is being torn). One rule, one set of semantics, expressed once. The single buffer is
not a special case with its own logic; it is the smallest ring.

This is why the ring is the primitive and the buffer is the sugar, not the other way around. Deriving
the buffer from the ring gives both the same overrun semantics for free. Deriving a ring from a
buffer would require bolting indexing and lap detection on afterward as new behavior.

</degenerate-ring>

<access-surface>

## The buffer access surface: `put` and `get`, views only

Data is not exposed as a raw, rebindable memory handle. Access goes through two operations:

- **`get(region) -> view`** returns a **zero-copy view** into the sector's memory. It must be a view,
  never a copy. The view behaves like an ordinary array to the caller; only its acquisition is
  mediated. Staleness is not `get`'s concern: the consumer checks the counter (`current - mine >= n`)
  and fails loud if it has been lapped. Returning a copy "for safety" would both destroy the
  zero-copy read the whole system exists to protect and manufacture the degraded-but-running state the
  failure model forbids (see [`failure-model.md`](failure-model.md)).
- **`put(region, data)`** writes through into the sector, an in-place write (a memory copy, or a
  direct transfer where the platform allows). This is the only write affordance.

For a ring, `get` and `put` take a slot index; the single-buffer sugar hides it.

### Why mediated access instead of a raw array handle

A raw, rebindable array attribute is a footgun: a caller can reassign it and silently swap the shared
segment for process-local heap memory, breaking cross-process sharing with no error at the point of
the mistake. The code keeps running, reading and writing memory nobody else can see. The `put`/`get`
surface makes that gesture inexpressible. There is no handle to rebind; there is only "write through
into the sector" and "take a view of the sector." Containment is partly by convention (a dynamic
binding cannot forcibly prevent a caller from holding onto a view), but the safe default is that the
advertised, ergonomic surface is `put`/`get` and the raw segment is never handed out to be rebound.

</access-surface>

<what-a-ring-is-not>

## What a ring is not

A ring holds bytes. It does not know what those bytes mean: that they are an image, a tensor, a
frame in a sequence, a tile of a larger image. The counter is a monotonic integer; "frame number" is
one consumer's interpretation of it, not a Convoy concept. Anything that assembles a buffer from
smaller pieces, or that interprets sectors as structured records, is a consumer concern living above
Convoy. This boundary is what keeps the primitive set closed (see [`../../PHILOSOPHY.md`](../../PHILOSOPHY.md)).

</what-a-ring-is-not>

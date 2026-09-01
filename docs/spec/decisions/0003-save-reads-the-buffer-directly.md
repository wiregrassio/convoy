<header>

# ADR 0003: Save reads the buffer directly, no snapshot or staging

Status: Accepted
Date: 2026-08-30

</header>

<context>

## Context

An early persistence design had `save` first copy the buffer into a staging buffer, after which an
asynchronous worker encoded the staged copy and wrote it to the store. The stated reason was to protect
the save from the producer overwriting the buffer while the encode was still in flight.

</context>

<decision>

## Decision

Drop the snapshot. `save_slice` reads the live buffer region directly. There is no staging buffer, no
temp-storage primitive, and no per-save allocation. The save is called explicitly in the consumer loop,
and if it cannot keep pace with the production cadence, that is a fail-loud crash.

</decision>

<alternatives>

## Alternatives

- **Snapshot into a staging buffer before saving.** Rejected: the snapshot only guards against the save
  falling behind the production cadence, but in this system, falling behind is *already* the
  catastrophic keep-up failure (if the save of unit N is not done before unit N+1 arrives, the data is
  already lost). The copy does not prevent that failure; it converts one already-lost unit into a
  growing pile of copies, i.e. a memory leak. It defends a position that already means you have lost,
  and defends it badly.
- **A pre-allocated, reused staging buffer (no per-save allocation, but still a copy).** Rejected: same
  defense of an already-lost position, merely without the allocation cost.
- **Keep a temp-storage primitive for the staging buffer.** Rejected: allocating and later dropping a
  staging sector is just `create` followed by `drop`. It was never a distinct primitive, so keeping it
  would have widened the primitive set for nothing (see
  [`0001-closed-primitives.md`](0001-closed-primitives.md)).

</alternatives>

<consequences>

## Consequences

- No temp-storage or snapshot primitive on the interface; the closed primitive set stands.
- The save reads the live buffer safely within the window the surrounding pipeline guarantees the buffer
  is stable; if that timing budget is ever violated, it fails loud rather than buffering around the
  violation.
- This is the fail-loud / restart-fast law applied to the persistence path: do not build machinery to
  defend against an already-lost state: crash into it, visibly (see
  [`../architecture/failure-model.md`](../architecture/failure-model.md)).

</consequences>

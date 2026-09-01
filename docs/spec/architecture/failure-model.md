<overview>

# Failure model

Convoy fails loud and restarts fast. There are no fallbacks, no retries-in-place, no
catch-and-continue paths. When the system leaves a known-good state it emits a specific, named
condition and crashes; recovery is a restart to a freshly initialized, known-good state. This document
explains why that is safe and what makes it cheap.

</overview>

<principle>

## The principle

A degraded system that keeps running is more dangerous than one that stops. A consumer reading a torn
buffer, a save that has silently fallen behind and is dropping data, a coordination state that no longer
matches reality: each of these, if allowed to continue, hides the fault until it corrupts a downstream
result or exhausts a resource. Convoy refuses the degraded middle ground. Any deviation from a
known-good state is treated as fatal at the moment it is detected.

Concretely, deviations surface as **specific named conditions**, not generic errors: a save that
cannot keep the required cadence, a consumer overrun where the producer has lapped a still-in-use
sector, and so on. A named condition says exactly what invariant broke, so the crash is diagnosable
rather than mysterious. The system does not catch these and continue; it reports and exits.

</principle>

<ephemeral-durable-split>

## Why crashing is safe: the ephemeral / durable split

Fail-loud is only a responsible strategy because Convoy separates two kinds of state and treats them
completely differently:

- **Coordination state is ephemeral.** The control region (the counter, the ring bookkeeping) is
  rebuilt from zero on every start. Nothing about it is persisted, and nothing is reconciled across a
  restart. A fresh control region is a correct control region by construction; there is no prior value
  a restart must recover or agree with.
- **Data state is durable, and lives only in the object store.** The one place durable state exists is
  the store, reached through `save`/`retrieve` (see [`persistence.md`](persistence.md)). It is
  externalized precisely so that a restart carries nothing forward that could come back inconsistent
  with reality.

These two never mix. Coordination state is never persisted; durable data never lives in the control
region. That strict separation is the whole reason a crash-and-restart is clean: there is no persisted
coordination state that could survive a crash in a stale or half-updated form and have to be
painstakingly reconciled on the way back up. The recovery path that fail-loud forbids ("figure out what
the persisted coordination state means and repair it") cannot exist here because that state is never
persisted in the first place.

</ephemeral-durable-split>

<cheap-restart>

## Why restart is cheap

Because the daemon holds almost no state (see [`runtime-and-bindings.md`](runtime-and-bindings.md)), the
cost of a restart is small: re-initialize the control regions from zero and reconnect to the store. No
replay, no reconciliation, no warm-up of coordination structures. Restart-to-known-good is therefore a
real, fast recovery strategy, not a euphemism for a lengthy rebuild. The design pays for this by keeping
the daemon deliberately close to stateless: every piece of state that would make restart expensive is
either ephemeral (thrown away and rebuilt) or durable (in the store, untouched by the restart).

</cheap-restart>

<consequences>

## Consequences for callers

- **No defensive copies.** `get` returns a view and never a copy, because a copy would only defend
  against a producer overwriting a live read, which is an overrun, already a fail-loud condition. The
  copy would not prevent the loss, it would just hide it and cost throughput (see
  [`rings-and-buffers.md`](rings-and-buffers.md)).
- **Overrun is the consumer's check.** A consumer is responsible for comparing the counter against the
  value it started on and crashing if it has been lapped (`current - mine >= n`). Convoy gives the
  consumer the signal; the consumer must act on it rather than trust a stale view (see
  [`signaling.md`](signaling.md)).
- **A save that cannot keep up crashes.** If persistence cannot match the production cadence, that is a
  named fatal condition, not something to buffer around. Buffering around it would trade a clean crash
  for an unbounded memory leak (see [`persistence.md`](persistence.md) and
  [ADR&nbsp;0003](../decisions/0003-save-reads-the-buffer-directly.md)).
- **A dead process leaves nothing to clean up.** Because coordination crosses the boundary as a
  lock-free counter and consumers hold receipts rather than locks (see
  [`ownership-and-receipts.md`](ownership-and-receipts.md)), a process that dies holds nothing. It simply
  stops advancing or waiting on the counter. There is no orphaned lock to break and no partial
  transaction to roll back.

</consequences>

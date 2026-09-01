<overview>

# Signaling: the waitable counter

The only coordination that crosses the process boundary is a single monotonic counter living in each
ring's control region. It is one primitive with three operations. It carries no lock and no ownership,
which is what makes the whole fail-loud/restart-fast model safe.

</overview>

<operations>

## One counter, three operations

The waitable counter is **one primitive, not several**. It is a monotonically increasing integer in
shared memory with four underlying actions (read, increment, block-until-changed, wake-blocked-waiters)
exposed as three operations:

- **`signal`** = increment the counter, then wake any waiters. Used by the producer after it finishes
  filling a sector.
- **`await`** = block until the counter changes, then read it. Used by a consumer waiting for the next
  unit of work.
- **`peek`** = read the counter without blocking. Used when a consumer wants the current value but must
  not block.

That is the entire coordination surface. There is no queue, no message, no broker. The producer moves
one integer; consumers watch it.

</operations>

<low-level-wait>

## The low-level wait

Blocking is implemented with a low-level, lock-free wait/wake facility on the shared counter word (a
futex-style operation: a process sleeps until a specific memory word changes, and another process wakes
sleepers on that word). The important property is that this is **not a lock**. No process holds
anything across the boundary. A waiter is either asleep on a memory address or awake; a signaler
increments the word and issues a wake. If a waiter dies, nothing is held and nothing leaks: it simply
never wakes again, because it is gone. There is no owner of the wait, so there is nothing to reconcile
on a death. This is the concrete reason the design uses a raw wait/wake facility rather than a
cross-process mutex or condition variable: a mutex is a lock a dead worker can die holding, and
recovering from that (the dead-owner handling a robust mutex requires) is exactly the recovery path the
design law forbids.

### Wait, don't spin

Consumers **block** on the counter; they do not busy-spin polling it. A spin loop is marginally lower
latency but pins a Jetson core at full utilization, which raises power draw and board temperature; on the
Orin-class boards Convoy targets, thermal headroom is a real constraint. At the operation
cadence Convoy is built for, the wake latency of a blocking wait is negligible relative to the work per
unit, so the thermally quiet option wins on the metric that actually matters. `peek` exists for the
cases that genuinely cannot block; the default is `await`.

</low-level-wait>

<interpretation>

## How consumers interpret the counter

The counter is just an integer; the consumer supplies all meaning:

- **Which sector:** for a ring of `n` sectors, counter value `m` corresponds to sector `m mod n`.
- **Staleness / overrun:** a consumer remembers the value `mine` it last began processing. Before it
  trusts a view, it compares against the current value. If `current - mine >= n`, the producer has
  advanced a full ring around while the consumer was still working on its sector: the sector has been
  (or is being) overwritten, so the consumer's view is torn. That is an overrun, and it is a fail-loud
  crash, not something to paper over (see [`failure-model.md`](failure-model.md)). At `n = 1` this
  reduces to "fail on any advance during a live read", the single-buffer tear.

The counter therefore does triple duty with no extra machinery: it is the work signal (it advanced, so
there is new data), the liveness signal (it is still advancing, so the producer is alive), and the
staleness signal (it advanced too far, so I have fallen behind). No separate heartbeat, no separate
sequence number, no separate liveness channel: one integer, watched.

</interpretation>

<ephemeral-coordination>

## Why coordination is ephemeral

The counter and the ring bookkeeping around it live in the control region, which is **ephemeral**: it
is rebuilt from zero every time the system starts, and nothing about it is persisted or reconciled
across a restart. A counter that starts at zero in a freshly created control region is, by definition,
correct: there is no "previous value" that a restart must recover or agree with. This is what lets the
failure model crash and restart freely: the coordination state has no durable form that could come back
inconsistent. Durable state lives only in the object store, never in the control region, and the two
never mix. See [`failure-model.md`](failure-model.md).

</ephemeral-coordination>

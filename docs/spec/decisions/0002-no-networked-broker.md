<header>

# ADR 0002: No networked broker; coordination collapses to local primitives

Status: Accepted
Date: 2026-08-30

</header>

<context>

## Context

A common pattern for coordinating producers and consumers is a networked broker (an in-memory
key-value server or message queue) carrying the work queue, liveness/heartbeat, shared configuration,
and event forwarding. That pattern is built for a distributed problem: fan-in buffering, work-stealing
across workers, and coordination across many hosts. Convoy's target is the opposite shape. Every
process runs on **one host**, sharing one shared-memory mount, and the consumers are homogeneous with
fixed, statically assigned work. None of the distributed conditions a broker exists to handle are
present.

</context>

<decision>

## Decision

Use **no networked broker.** Each job such a broker would carry collapses to a local primitive:

- **Work distribution:** none needed. Consumers get a static assignment at startup; there is no queue
  to hand out units of work.
- **Ready signal:** the waitable counter in the shared control region, a lock-free wait/wake on one
  integer (see [`../architecture/signaling.md`](../architecture/signaling.md)).
- **Liveness / heartbeat:** the same counter advancing is the liveness signal; a supervising owner can
  additionally observe its child processes directly. No separate heartbeat channel.
- **Configuration:** static at ring creation, carried in `config`. Reconfiguration is a restart, not a
  live-updated shared value.
- **Event / result forwarding:** none; durable results are metadata on saved slices in the object store
  (see [`0004-pluggable-local-object-store.md`](0004-pluggable-local-object-store.md)).

</decision>

<alternatives>

## Alternatives

- **Keep a broker as the coordination layer.** Rejected: a networked broker solving a distributed
  problem this single-host deployment does not have adds a process, a network hop, and a failure surface
  for no benefit. It also invites persisted coordination state, which the failure model forbids.
- **Swap the broker for a different single broker.** Rejected: the goal is not to find one broker to
  carry everything, but to match each primitive to the actual shape of the traffic: a lock-free counter
  for the one-to-many ready signal, direct writes for persistence. A single general broker is the wrong
  tool applied uniformly.

</alternatives>

<consequences>

## Consequences

- No broker process in the stack, and no network hop on the coordination path.
- The only coordination that crosses the process boundary is the lock-free counter in the control
  region; the daemon stays out of the per-operation hot path.
- Static assignment (no work queue) is what makes fail-loud on a dead consumer correct: because a dead
  consumer's work is not redistributed to survivors, a dead consumer is a fatal condition that triggers a
  restart rather than a silent degradation (see
  [`../architecture/failure-model.md`](../architecture/failure-model.md)).
- Coordination state gains no durable form, which is a precondition for the ephemeral/durable split the
  failure model depends on.

</consequences>

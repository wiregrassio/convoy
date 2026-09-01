<header>

# ADR 0004: A pluggable local object store with declarative retention

Status: Accepted
Date: 2026-08-30

</header>

<context>

## Context

Convoy needs an optional durable record for buffers a consumer chooses to persist. An earlier approach
used a general-purpose blob store fronted by upload workers, with retention and lifecycle configured as
a separate policy system. The deciding pain was not throughput, it was **lifecycle management**: the
store's retention and expiry were fought for a long time and stayed unreliable. The store could not be
trusted to age data out on its own, so operators were left manually managing what should have been
automatic.

</context>

<decision>

## Decision

Persist through a **pluggable local object store, purpose-built for time-ordered blob data, whose
retention is a declarative size quota.** Set a size per bucket; when the bucket fills, the store evicts
the oldest data first-in-first-out. The store:

- runs as its own local process, reached by the daemon's persistence workers over the local network;
- is **hidden entirely behind `save` / `retrieve`**: no consumer knows its name, holds its endpoint, or
  addresses it directly, so it is swappable for any store meeting the same contract;
- carries **metadata on each saved slice**, so structured records that annotate data (labels, results)
  live as that metadata rather than in a second parallel store;
- exposes **no delete primitive** through Convoy: retention is the quota, full stop.

</decision>

<alternatives>

## Alternatives

- **Keep the previous store and fix its lifecycle.** Rejected: significant effort was already spent and
  it stayed unreliable. The lifecycle *was* the failure, and it did not become operable with more
  tuning.
- **A general object store plus external lifecycle rules.** Rejected: this reintroduces exactly the
  manual lifecycle-rule management that failed. The goal is to make retention one declarative knob, not
  a policy engine to operate.
- **A columnar / time-series database for the blobs.** Rejected: those are shaped for structured rows
  and metrics, not large binary blobs. What is needed here is an object store; a metrics store is a
  different tool for a different job.
- **Expose a `delete` primitive.** Rejected: a delete reintroduces manual lifecycle management (the
  precise thing the declarative quota exists to remove) and would widen the closed primitive set (see
  [`0001-closed-primitives.md`](0001-closed-primitives.md)).

</alternatives>

<consequences>

## Consequences

- Retention is one declarative setting per bucket: set a size and forget it. No TTLs, no tiers, no
  external policy system.
- There is no delete primitive in the data plane; lifecycle is the store's quota.
- Structured results become store metadata on saved slices, which removes any need for a second store or
  a separate forwarding path for them.
- Because the store is hidden behind `save` / `retrieve`, it is an implementation detail: swappable, and
  never exposed as an endpoint or credential to any consumer (see
  [`../architecture/persistence.md`](../architecture/persistence.md)).

</consequences>

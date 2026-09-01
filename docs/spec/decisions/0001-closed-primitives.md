<header>

# ADR 0001: A closed primitive set, composition over extension

Status: Accepted
Date: 2026-08-30

</header>

<context>

## Context

Convoy is infrastructure that many consumer processes depend on to move data. Infrastructure like this
succeeds or fails on whether it can be trusted, and trust in a low-level data plane comes from being
able to reason about and test the whole of its behavior. An interface that grows a new operation every
time a consumer has a new need has an unbounded surface: it can never be exhaustively tested, its
interactions multiply, and over time it accumulates operations that encode specific consumers'
assumptions into what should be a neutral substrate. The pressure to grow the interface is constant and
always locally reasonable: each new verb "just" serves one more real use case.

</context>

<decision>

## Decision

Fix a small, closed set of primitives and hold it closed. New capability comes from **composing**
existing primitives, never from adding new ones. The primitive set is:

- `create_ring(n, config)`, `attach(name)`, `drop`
- the waitable counter (`signal` / `await` / `peek`), one primitive
- `save_slice(region, metadata)`, `retrieve(filter, projection, return_object)`

plus the mediated `put` / `get` view surface for buffer access. Common cases are served by **sugar**
(`create_buffer`, `save`, `query` / `get` / `list`), which are thin compositions of the set, not new
members of it. A proposed new primitive is treated as a design smell: the first question is which
composition of existing primitives satisfies the need, and the second is whether the need is actually a
consumer concern (meaning, scheduling, interpretation of bytes) that does not belong in the plane at
all.

</decision>

<alternatives>

## Alternatives

- **Let the interface grow to fit consumers.** Rejected: this is the unbounded-surface problem itself.
  Each addition is locally justified and the sum is an untestable, opinionated interface.
- **A large "batteries-included" primitive set chosen up front.** Rejected: a big fixed set is still a
  large surface to prove correct, and pre-guessing every primitive bakes in assumptions as surely as
  growing the set does. Small-and-composable beats large-and-fixed.
- **No sugar, expose only the raw primitives.** Rejected: without sugar the common cases read poorly,
  which creates pressure to add primitives to make them ergonomic. Sugar relieves that pressure without
  widening the surface that must be verified.

</alternatives>

<consequences>

## Consequences

- The interface is enumerable and testable by construction: every primitive, every failure mode, every
  interaction can be covered.
- Capability grows in the consumer layer (compositions of primitives), not in the plane. The plane stays
  the same size as the system's uses multiply.
- There is a hard scope test for any proposed feature: if it requires the plane to understand what the
  bytes mean, it is rejected as a consumer concern. This is the gate every contribution is measured
  against (see [`../../../CONTRIBUTING.md`](../../../CONTRIBUTING.md)).

</consequences>

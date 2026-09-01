<purpose>

# Contributing to Convoy

Convoy is small, strict infrastructure. Contributions are welcome, but the bar is unusual: the hardest
part of contributing is often showing that a change is *not* needed as a new primitive. Read
[`docs/PHILOSOPHY.md`](docs/PHILOSOPHY.md) before proposing anything: it is the constitution every
contribution is measured against.

</purpose>

<hard-gate>

## The hard gate: the closed primitive set

**Convoy's primitive set is closed. A pull request that adds a new primitive starts from a position of
rejection and must overturn it.** The primitives are:

- `create_ring`, `attach`, `drop`
- the waitable counter (`signal` / `await` / `peek`)
- `save_slice`, `retrieve`
- the `put` / `get` view surface

Before proposing a new primitive, you must show, in the PR description:

1. **The composition you tried.** Which arrangement of existing primitives was attempted, and precisely
   why it cannot express the need. "It would be more convenient" is not "it cannot be expressed":
   convenience is what sugar is for.
2. **That the need is not a consumer concern.** If the feature requires Convoy to understand what the
   bytes *mean* (frames, tiles, batches, records, detections) or to make a scheduling decision, it is a
   consumer concern and belongs above the plane, not in it. This is the single most common reason a
   proposed primitive is rejected.
3. **That it cannot be sugar.** If the real goal is an ergonomic shortcut for a common composition, it is
   sugar (a thin, named composition of existing primitives), not a primitive. Sugar is welcome; new
   primitives almost never are.

Only if all three are answered does a new primitive get considered, and even then it must come with the
full failure-mode and interaction analysis that every primitive carries. Widening the set widens the
surface that must be proven correct, forever, so the burden is deliberately high.

</hard-gate>

<good-contributions>

## What good contributions look like

- **New sugar** for a genuinely common composition, named for what it composes.
- **A new language binding**, kept thin (`attach`, the view surface, the counter, `save`/`retrieve`) and
  adding no interpretation of bytes.
- **Hardening the failure modes**: better named conditions, tighter overrun detection, more thorough
  restart-to-known-good coverage.
- **Documentation** that adds the missing "why" to a choice, or corrects one that drifted.

</good-contributions>

<design-law-compliance>

## Design-law compliance (checked in review)

Every change is reviewed against the five laws. A change that violates one is rejected regardless of how
useful it seems, because the laws are what make the plane trustworthy:

1. **Closed primitives**: compose, do not extend (the gate above).
2. **Single-owner, receipt-based ownership**: no change may introduce shared or ambiguous ownership of a
   segment, or a cross-boundary lock.
3. **Zero-copy by default**: no defensive copy on the read path; the read surface hands out views.
4. **Fail loud, restart fast**: no fallbacks, no catch-and-continue; no fallback default on a config
   read (an absent value fails loud at startup; any dev default is an obviously fake, greppable
   sentinel); coordination state stays ephemeral, only data is durable.
5. **Determinism boundary**: no hidden global state; effects are explicit at their call site.

</design-law-compliance>

<testing-bar>

## Testing bar

- Every primitive change lands with tests for the primitive **and its failure modes** (see
  [`docs/CONVENTIONS.md`](docs/CONVENTIONS.md)). The overrun crash and restart-to-known-good are behaviors
  under test, not incidental error paths.
- The mechanical layer is deterministic and tested in isolation. A change that makes a primitive hard to
  test in isolation is a red flag: it has probably acquired a hidden dependency.

</testing-bar>

<pull-request-expectations>

## Pull request expectations

- State which of the five laws your change touches and how it complies.
- If you believe you need a new primitive, structure the PR description around the three questions above.
  A new-primitive PR without that analysis will be asked for it before any review of the code.
- Keep the change scoped to one mechanism. A PR that spans several mechanisms is usually several PRs.
- Match the existing naming and documentation register. New behavior is documented where it lives: a new
  mechanism gets a `docs/spec/architecture/` file; a cross-cutting or reversal decision gets an ADR in
  `docs/spec/decisions/`.

</pull-request-expectations>

<recording-decisions>

## Recording decisions

A change that is cross-cutting, sets project structure, or reverses an earlier direction gets an
Architecture Decision Record in [`docs/spec/decisions/`](docs/spec/decisions/). Follow the existing format
(Context / Decision / Alternatives / Consequences), take the next sequential number, and never edit an
accepted ADR: supersede it with a new one. Per-mechanism rationale that is fully captured by a mechanism
doc does not need an ADR.

</recording-decisions>

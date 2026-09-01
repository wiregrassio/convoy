<overview>

# Philosophy

This is the project's constitution. Convoy is a small piece of infrastructure with an unusually
strict remit, and the discipline below is the reason it can stay small. Every architectural choice
in this repository traces back to one of these five laws. A design that contradicts one of them is
a bug in the design, not a reason to relax the law.

The context that produced these rules: a single host, a producer generating large image or tensor
buffers faster than they can be copied, and several consumer processes that must all read those
buffers to keep the Jetson's GPU saturated. In that setting a copy is not a convenience, it is the
thing that destroys the throughput the whole system exists to provide. The laws fall out of taking
that seriously.

</overview>

<closed-primitives>

## 1. Closed primitives

**A small, fixed, fully testable set of verbs. New capability comes from composing them, never from
extending or adding to them.**

Convoy has exactly 6 primitives (`create_ring`, `attach`, `drop`, the waitable counter,
`save_slice`, `retrieve`) plus a mediated `put`/`get` view surface. That set is closed. Adding a
seventh primitive is treated the way a language designer treats adding a new arithmetic operator:
almost always the real need is a composition of the operators that already exist, and the request
is a signal that a consumer concern is leaking into the plane.

The payoff is testability by construction. An enumerable interface with no growth path can be tested
exhaustively: every primitive, every failure mode, every interaction. An interface that grows a verb
per use case has an unbounded test surface and can never be trusted the way infrastructure must be.
The common cases stay ergonomic through **sugar** (`create_buffer`, `save`, `query`), which are
thin compositions of the closed set, not new members of it. Sugar keeps the surface friendly without
widening the thing that must be proven correct.

The corollary is a scope test. If a proposed feature would require the plane to understand what the
bytes *mean* (frames, tiles, batches, detections), it does not belong here. Convoy moves bytes and
signals a counter. Meaning is the consumer's job. Keeping meaning out is what keeps the primitive set
closable.

</closed-primitives>

<single-owner-ownership>

## 2. Single-owner, receipt-based ownership

**Every buffer has exactly one owner. Access is granted by an explicit receipt. There is no
orphaned, shared, or non-deterministic ownership, ever.**

A ring is created by exactly one process, which owns it. Consumers `attach` to it and hold a receipt
that grants read (and, where configured, write-through) access, but never ownership. `drop` releases
a handle, and only the owner's `drop` unlinks the underlying segment. This makes lifecycle
deterministic: there is always exactly one process responsible for a segment's existence, so a
segment is never leaked by ambiguity about who should free it and never freed out from under a live
reader by a consumer that thought it was in charge.

This is the rule that a shared lock across the process boundary would violate. A lock a consumer can
hold is a lock a dead consumer can die holding, and recovering a segment from a dead lock-holder is
exactly the non-deterministic ownership this law forbids. Ownership lives with one process; the
coordination that crosses the boundary carries no ownership at all (see law 4 and the signaling
mechanism).

</single-owner-ownership>

<zero-copy>

## 3. Zero-copy by default

**Data is viewed across the process boundary, not copied.**

The read surface hands out views into shared memory and never copies "for safety." `get` returns a
view; the save reads the buffer in place. A defensive copy on read would reintroduce a per-operation
allocation on the hottest path in the system and buy nothing, because the only thing a copy could
protect against, the producer overwriting a buffer mid-read, is a condition the counter already
detects and fails loud on (see law 4). Correctness comes from the staleness check, not from copying,
so the copy is pure cost.

Convoy makes **no claim about how a producer filled a buffer.** Producers often assemble a buffer
from smaller pieces, and that assembly may be copy-ful; it is the producer's business and outside
this plane. The zero-copy guarantee is a property of the *read* surface: once a buffer exists, every
consumer that reads it does so without a copy.

</zero-copy>

<fail-loud>

## 4. Fail loud, restart fast

**Any deviation from a known-good state is a crash, not a degraded limp. Coordination state is
ephemeral and rebuildable; only data is durable. Recovery is restart-to-known-good.**

Convoy does not have fallbacks, retries-in-place, or catch-and-continue paths. When something is
wrong (a consumer has fallen far enough behind that its view is stale, a save cannot keep the
required cadence), the system emits a specific, named condition and crashes. It does not attempt to
soldier on in a degraded state, because a degraded state that keeps running is a state that hides the
fault until it corrupts something.

The same rule governs configuration, and the config boundary is where the subtle version of the
failure lives. A configuration value has no plausible default: an absent or unreadable value fails
loud at load, at startup, not later inside a running path. A plausible default is worse than a
missing one, because it lets the system run on a wrong value while looking healthy, turning a broken
config path into a machine-specific latent fault whose symptom is detached in time from its cause.
Where a development or test default is unavoidable, it is an obviously fake, greppable sentinel that
fails loud in production, never a value that could be mistaken for real.

This is only safe because of the ephemeral/durable split. The control region (the counter, the ring
bookkeeping) is coordination state, and it is rebuilt from zero on every start. Nothing is
reconciled across a restart, because there is nothing to reconcile: a fresh control region is a
correct control region. The only durable state is what was written to the object store, and that is
externalized precisely so that a restart carries nothing forward that could be inconsistent with
reality. Cheap, total restart is a feature the design pays for by keeping the daemon almost
stateless.

The alternative (persisting coordination state so a restart can "resume") is the thing this law
exists to forbid. Persisted coordination can survive a crash in a form that no longer matches
reality, and reconciling it is the exact recovery path that turns a clean restart into an
open-ended debugging problem.

</fail-loud>

<determinism-boundary>

## 5. Determinism boundary

**Mechanical operations are deterministic and independently testable. No hidden global state.**

Each primitive does one mechanical thing whose inputs and outputs are fully visible at the call
site. Actions are explicit where they happen: a persist is a call in the loop, not a side effect
fired implicitly when a counter advances. Static properties of a buffer (its size, its retention
policy) are set once at creation, where they describe what the buffer *is*; recurring actions live
at the site where they recur. A reader tracing a consumer loop sees every durable effect at the line
that causes it, with nothing hidden in a declaration elsewhere and nothing depending on invisible
global state. That visibility is what makes the mechanical layer testable in isolation and what keeps
a later refactor from silently severing an effect that was never written down where it happened.

</determinism-boundary>

---

<how-the-laws-relate>

## How the laws relate

They are not five independent preferences; they reinforce each other. Closed primitives (1) keep the
surface small enough to test exhaustively, which is what makes fail-loud (4) tractable: you can
enumerate the deviations to crash on. Single-owner ownership (2) is what lets coordination cross the
boundary carrying no lock, which is what makes the ephemeral control region safe to throw away and
rebuild (4). Zero-copy (3) is the property the whole thing exists to protect, and the determinism
boundary (5) is what keeps every one of the others verifiable rather than aspirational. Weaken any
one and the others start to cost more than they should.

</how-the-laws-relate>

<overview>

# Conventions

Repository, code, and documentation conventions for Convoy. These exist so the project reads as one
coherent thing regardless of who wrote a given piece. They are conventions, not the design law (the
law is in [`PHILOSOPHY.md`](PHILOSOPHY.md)), but they are enforced in review all the same.

</overview>

<repository-layout>

## Repository layout

- `docs/` holds all prose. `docs/spec/architecture/` is one file per mechanism; `docs/spec/decisions/` is the ADR
  register. The compiled core and the language bindings get their own top-level directories when
  implementation begins; the layout is intentionally not created ahead of the code that fills it, so the
  design dictates the structure rather than the reverse.
- One concept per file. If a document needs a second `#`-level title, it is probably two documents.
- Cross-reference by relative link, so the docs remain navigable as a set and nothing points outside the
  repository.

</repository-layout>

<code-conventions>

## Code conventions (for when implementation begins)

- **The compiled core is the timing-critical layer.** It carries no garbage collector, no
  interpreter-wide lock, and no managed-runtime jitter on the hot path. Anything on the per-operation
  path lives here.
- **Bindings stay thin.** A binding performs `attach`, the `put`/`get` view surface, the counter
  operations, and the `save`/`retrieve` calls, nothing more. Logic that interprets bytes does not
  belong in a binding; it belongs in the consumer that uses the binding.
- **Module-size discipline.** A module implements one mechanism. When a module starts spanning two
  mechanisms, split it. Small modules with a single responsibility are the code-level expression of the
  closed-primitive philosophy.
- **No hidden global state.** State is passed explicitly. A mechanical operation's inputs and effects
  are visible at its call site (design law 5). This is what keeps the core independently testable.
- **Names match the primitives.** The vocabulary in code is the vocabulary in these docs: `create_ring`,
  `attach`, `drop`, `signal`/`await`/`peek`, `save_slice`, `retrieve`, `put`, `get`. Sugar is named for
  what it composes (`create_buffer`, `save`, `query`, `list`). Do not invent synonyms.

</code-conventions>

<failure-conventions>

## Failure and error conventions

- **Fail loud.** No fallbacks, no catch-and-continue, no silent degradation. A deviation from a
  known-good state is a crash (design law 4).
- **Name the condition.** A fatal condition is a specific, named thing (an overrun, a save that cannot
  keep cadence), not a generic error. The name states which invariant broke, so a crash is diagnosable.
- **No defensive copies to mask a fault.** A copy that exists only to paper over a staleness or timing
  problem is forbidden: it hides the fault and costs the throughput the plane exists to protect.
- **No plausible config defaults.** A configuration value is validated at load; an absent or invalid
  one crashes at startup, not later. A default is never a plausible value that could run silently. Any
  unavoidable development or test default is an obviously fake, greppable sentinel that fails loud in
  production.

</failure-conventions>

<testing-conventions>

## Testing conventions

- **The closed primitive set is testable by construction; test it that way.** Every primitive, every
  failure mode, and the interactions between them are covered. The whole point of keeping the set closed
  is that this is achievable: do not let it slide.
- **Test the failure modes as first-class behavior.** The overrun crash, the save-cannot-keep-up crash,
  and restart-to-known-good are behaviors with tests, not incidental error paths.
- **Deterministic mechanical layer.** Because the mechanical operations are deterministic and free of
  hidden global state, they are tested in isolation without elaborate harnesses. Preserve that: a
  primitive that becomes hard to test in isolation has probably grown a hidden dependency.

</testing-conventions>

<documentation-register>

## Documentation register

- Prose is dense and technical. State the mechanism and the reason it is shaped that way; omit filler.
- Every non-obvious choice carries its "why." A document that says only *what* without *why* is
  incomplete: the reasoning is the load-bearing part, because it is what lets a future reader tell a
  principled constraint from an incidental one.
- Do not use em dashes. Use commas, colons, parentheses, or periods.
- Keep the docs freestanding. A document refers only to other documents in this repository and to
  generic technical concepts, never to anything outside the project.

</documentation-register>

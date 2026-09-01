<purpose>

# architecture/

One file per mechanism. Each carries the full detail behind a section of the architecture
overview (`../ARCHITECTURE.md`): the design, the reasoning, and the invariants it must hold.
These are flowing argument, not reference tables; read the whole of the one you need.

</purpose>

<contracts>

## Contracts

- Upholds the repository working law (see `../../../CLAUDE.md`), especially fail-loud and
  shortest-complete prose: each mechanism states its failure behavior as a named crash, never a
  silent fallback.
- Each file is scoped to one mechanism. Cross-mechanism concerns live in the overview or an ADR,
  not duplicated here.
- Platform-wide law (closed primitives, fail-loud, ephemeral coordination) lives in
  `../../PHILOSOPHY.md`; these files inherit and cite it rather than restate it.

</contracts>

<files>

## Files

| File | Mechanism |
|------|-----------|
| `rings-and-buffers.md` | The ring and sectors, the single buffer as a degenerate ring, the put/get view surface. |
| `ownership-and-receipts.md` | Single-owner lifecycle, receipts, create/attach/drop semantics. |
| `signaling.md` | The waitable counter, its low-level wait, staleness detection. |
| `persistence.md` | save/retrieve, the pluggable object store, declarative retention, no delete. |
| `runtime-and-bindings.md` | The daemon and binding split, the hot-path boundary, the cross-process substrate. |
| `failure-model.md` | Fail-loud and restart-fast in practice: named conditions, ephemeral versus durable state. |
| `CLAUDE.md` | This file. |

</files>

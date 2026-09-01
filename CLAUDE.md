<purpose>

# convoy/

Convoy is a shared-memory data plane for zero-copy movement of large image and tensor buffers
between processes on a single host. It targets NVIDIA Jetson (Orin-class): high-throughput edge
pipelines where sensor bandwidth outruns the copy budget and where the CPU and the integrated GPU
share physical memory.

Status: design / pre-implementation. This repository holds the charter, philosophy,
architecture, and decision records; no implementation code exists yet. These documents are the
contract the implementation will be built against.

</purpose>

<operating-principles>

## Working law

Every file in this repository, and every future contributor or agent working in it, upholds
these. They are the project's own law, not a style preference.

- **Freestanding.** The containment law. Every file is self-contained. Name, link to, or import
  nothing outside this repository. The project does not assume anything else exists.
- **Fail loud, restart fast.** A deviation from a known-good state is a named crash, never a
  silent fallback, retry-in-place, or degraded limp. Coordination state is ephemeral and rebuilt
  from zero; only data is durable.
- **Shortest-complete prose.** Write the shortest form from which a reader reconstructs the whole.
  No filler, hedging, preamble, or re-summary. Notes state why, never restate what the text says.
- **Single source of truth.** Define a value, name, or contract once and reference it. Never copy
  something that can drift; never hardcode a number that can be derived.
- **Make the correct path the easy path.** Let structure enforce a rule. Do not rely on a reader
  remembering to do the right thing.
- **Grounded figures only.** Every capacity or performance claim names its mechanism or is marked
  not-yet-measured. No invented magnitudes.
- **No em or en dashes.** Use commas, colons, parentheses, or periods.

</operating-principles>

<files>

## Top-level files

| File | Purpose |
|------|---------|
| `README.md` | What Convoy is, the closed primitive set, one-paragraph architecture, status. |
| `CONTRIBUTING.md` | How to contribute, with the closed-primitives rule as a hard gate. |
| `LICENSE` | Apache-2.0. |
| `CLAUDE.md` | This file. Repository map and working law. |

</files>

<subdirectories>

## Subdirectories

| Directory | Purpose |
|-----------|---------|
| `docs/` | The design documents: philosophy (the working law argued), conventions, architecture overview. |
| `docs/spec/architecture/` | Per-mechanism detail: rings, ownership, signaling, persistence, runtime, failure model. |
| `docs/spec/decisions/` | Architecture Decision Records, immutable once accepted. |

</subdirectories>

<notes>

## Notes

The interface is a closed primitive set; new capability comes from composing them, never
from adding to the set. That closure is the spine of the design and the bar every contribution is
measured against (see `docs/PHILOSOPHY.md`, `CONTRIBUTING.md`).

</notes>

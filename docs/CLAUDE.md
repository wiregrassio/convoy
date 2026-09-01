<purpose>

# docs/

The working law and the specification. The intent and the argued philosophy live here at top
level; the specification of what to build lives under `spec/`. The narrative files are flowing
argument; read them in full, not as reference lookups.

</purpose>

<contracts>

## Contracts

- Upholds the repository working law (see `../CLAUDE.md`): freestanding, fail-loud,
  shortest-complete prose, single source of truth, grounded figures, no em dashes.
- `PHILOSOPHY.md` and `CONVENTIONS.md` are the working law: the design constitution argued rather
  than listed, plus the conventions. Every other document defers to them and does not restate them.
- The specification of what to build lives under `spec/` and is governed by the working law here:
  it references `PHILOSOPHY.md` and `CONVENTIONS.md` rather than duplicating them.

</contracts>

<files>

## Files

| File | Purpose |
|------|---------|
| `PHILOSOPHY.md` | The five-law design constitution, argued. The working law. |
| `CONVENTIONS.md` | Repository, code, and documentation conventions. The working law. |
| `CLAUDE.md` | This file. |

</files>

<subdirectories>

## Subdirectories

| Directory | Purpose |
|-----------|---------|
| `spec/` | The specification of what to build: the architecture overview, the per-mechanism specs, and the ADRs. Governed by the working law above. |

</subdirectories>

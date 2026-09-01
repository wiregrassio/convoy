<purpose>

# spec/

The specification of what to build. `ARCHITECTURE.md` is the top-down overview; `architecture/`
carries the per-mechanism specs behind it; `decisions/` holds the Architecture Decision Records.
These are flowing argument, not reference tables; read the whole of the one you need.

</purpose>

<contracts>

## Contracts

- Upholds the repository working law (see `../../CLAUDE.md`): freestanding, fail-loud,
  shortest-complete prose, single source of truth, grounded figures, no em dashes.
- Governed by the working law in `../PHILOSOPHY.md` and `../CONVENTIONS.md`. The spec references
  that law and inherits it; it does not duplicate or restate it.
- Per-mechanism rationale lives in `architecture/`; cross-cutting or reversal decisions live in
  `decisions/`. A document does not duplicate what one of those owns.

</contracts>

<files>

## Files

| File | Purpose |
|------|---------|
| `ARCHITECTURE.md` | The internal architecture overview and the map into `architecture/`. |
| `CLAUDE.md` | This file. |

</files>

<subdirectories>

## Subdirectories

| Directory | Purpose |
|-----------|---------|
| `architecture/` | One file per mechanism: the full depth behind the overview. |
| `decisions/` | The Architecture Decision Records. |

</subdirectories>

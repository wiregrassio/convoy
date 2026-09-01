<purpose>

# decisions/

Architecture Decision Records. One file per significant decision, immutable once accepted. This
register holds decisions that cut across mechanisms or set project structure, together with the
alternatives that were rejected. It is not a change log and not a restatement of the
per-mechanism rationale in `../architecture/`.

</purpose>

<contracts>

## Contracts

- Upholds the repository working law (see `../../../CLAUDE.md`).
- **Immutable once accepted.** Never edit a decided ADR. If it is reversed, write a new ADR that
  supersedes it and set the old one's status to superseded. A superseded decision is history, not
  staleness. This is the single-source-of-truth law applied to decisions: the record is the one
  place a decision and its rejected alternatives live.
- A decision fully captured by one mechanism's rationale does not get an ADR. An ADR is for
  decisions with no single mechanism home, or reversals.
- Format: `NNNN-short-slug.md`, sequential, each with Context, Decision, Alternatives,
  Consequences.

</contracts>

<files>

## Files

| File | Purpose |
|------|---------|
| `README.md` | The ADR index and format contract. |
| `0001-closed-primitives.md` | A closed primitive set, composition over extension. |
| `0002-no-networked-broker.md` | Coordination collapses to local primitives, no networked broker. |
| `0003-save-reads-the-buffer-directly.md` | Save reads the buffer directly, no snapshot or staging. |
| `0004-pluggable-local-object-store.md` | A pluggable local object store with declarative retention. |
| `CLAUDE.md` | This file. |

</files>

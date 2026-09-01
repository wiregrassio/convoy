<overview>

# Architecture Decision Records

One file per significant decision, immutable once accepted. This register records decisions that cut
across mechanisms or set project structure, together with the alternatives that were rejected. It is
**not** a change log and **not** a restatement of the per-mechanism rationale in
[`../architecture/`](../architecture/): a decision fully captured by one mechanism doc does not need an
ADR.

</overview>

<format>

## Format

`NNNN-short-slug.md`, sequential. Each record:

```
# ADR NNNN: Title
Status: Accepted | Superseded by ADR-MMMM
Date: YYYY-MM-DD
## Context       the forcing facts / constraints
## Decision      what was chosen
## Alternatives  what was rejected, and why
## Consequences  what this commits the project to
```

**Immutable once accepted.** Do not edit a decided ADR. If it is reversed, write a new ADR that
supersedes it and set the old one's status to "Superseded by ADR-MMMM." A superseded decision is
history, not staleness.

</format>

<index>

## Index

| ADR | Title | Status |
|-----|-------|--------|
| [0001](0001-closed-primitives.md) | A closed primitive set, composition over extension | Accepted |
| [0002](0002-no-networked-broker.md) | No networked broker; coordination collapses to local primitives | Accepted |
| [0003](0003-save-reads-the-buffer-directly.md) | Save reads the buffer directly, no snapshot or staging | Accepted |
| [0004](0004-pluggable-local-object-store.md) | A pluggable local object store with declarative retention | Accepted |

</index>

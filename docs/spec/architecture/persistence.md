<overview>

# Persistence

Persistence is the optional, off-hot-path side of Convoy: two primitives (`save_slice` and
`retrieve`) over a pluggable local object store that is hidden behind them. A pipeline that only moves
buffers never touches this layer; one that needs a durable record uses it without ever learning what
the store is.

</overview>

<save-slice>

## `save_slice(region, metadata)`

Persists a region of a buffer to the object store, together with caller-supplied metadata. Two
properties define it:

- **It reads the buffer directly.** There is no snapshot, no staging buffer, and no per-save
  allocation. The persistence worker reads the live region and writes it to the store. The reasoning
  is in [ADR&nbsp;0003](../decisions/0003-save-reads-the-buffer-directly.md): a staging copy only
  guards against the save falling behind the production cadence, which is already the catastrophic
  keep-up failure, and rather than prevent it, a copy converts one already-lost unit into a growing
  pile of copies (a memory leak). Direct read plus fail-loud is the honest handling of a condition that
  already means data was lost.
- **It is explicit in the consumer loop.** A save is a call the consumer makes where it wants the save
  to happen. It is never fired implicitly as a side effect of the counter advancing. Static properties
  of the buffer are declared once at creation; a recurring action like a save belongs at the site where
  it recurs, visible to anyone reading the loop (design law 5, [`../../PHILOSOPHY.md`](../../PHILOSOPHY.md)).
  Hiding a durable write inside a sector declaration is an invisible-side-effect footgun that a later
  refactor can silently sever.

`save(buffer, metadata)` is sugar: `save_slice` over the whole buffer.

</save-slice>

<retrieve>

## `retrieve(filter, projection, return_object)`

The single read path out of the store, shaped like a document query:

- **`filter`** selects records: a key for a point lookup, or a range plus labels for a range query.
- **`projection`** selects which descriptor fields come back.
- **`return_object`** gates the payload blob (the one large, expensive field), which is off by
  default so metadata queries stay cheap.

It returns an iterator, so range queries stream rather than materializing everything at once. The
common reads are sugar over it: `query` is `retrieve` with `return_object` off; `get` is a
filter-by-key with the blob on; `list` is a minimal projection with the blob off.

The blob is gated by its own flag rather than folded into the projection because it is a different kind
of thing from the descriptor fields (the single heavy payload versus cheap metadata), and a distinct
flag reads more honestly than pretending the blob is just one more field you might project.

</retrieve>

<store>

## The store: pluggable, hidden, declaratively retained

The durable store is a **local object store running as its own process**, reached over the local
network by the daemon's persistence workers. Three properties matter:

- **Hidden.** No consumer knows the store's name, holds its endpoint, or addresses it directly. The
  only interface to durability is `save` / `retrieve`. This keeps the store an implementation detail:
  it can be swapped for any store satisfying the persistence contract, and no consumer code changes.
- **Declaratively retained, with no delete primitive.** Retention is a size quota set once at bucket
  creation: set a size, and the store evicts the oldest data first-in-first-out when the bucket fills.
  There is deliberately **no delete primitive** on the Convoy interface. Lifecycle is the quota's job
  ("set a size and forget it"), and a delete would reintroduce exactly the manual, error-prone lifecycle
  management the declarative quota exists to remove ([ADR&nbsp;0004](../decisions/0004-pluggable-local-object-store.md)).
- **Metadata-carrying.** A saved slice carries its metadata, and `retrieve` filters and projects over
  that metadata. Structured records that annotate saved data (labels, results, tags) are stored as this
  metadata alongside the blob, not in a second parallel store. There is one durable store, and it holds
  both the blobs and their descriptors.

</store>

<what-persistence-is-not>

## What persistence is not

Persistence is off the hot path and out of the coordination model. The store holds **only durable
data**; it never holds coordination state (that is the ephemeral control region, see
[`signaling.md`](signaling.md)). The two never mix, and that separation is what makes the failure
model's cheap restart safe (see [`failure-model.md`](failure-model.md)). Persistence is also entirely
optional: it is a facade a consumer reaches for when it needs a durable record, not a stage every
buffer passes through.

</what-persistence-is-not>

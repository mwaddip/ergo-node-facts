# Modifier Store Contract

## Component: `store/` (enr-store)

Persistent storage for block-related modifiers. A dumb persistence layer — receives
pre-validated, pre-serialized bytes from components above and writes them to disk.
Does not parse, validate, or interpret modifier content.

Primary consumers: validation pipeline (writes headers after validation), sync state
machine (reads on startup to rebuild chain state), future block validation (reads
block sections).

## Design Principles

- **Bytes in, bytes out.** The store does not know what a Header or Transaction looks like.
  It stores `(type_id, modifier_id, height, data)` tuples.
- **Backend-agnostic interface.** The public trait is implemented by redb today. A different
  backend (RocksDB, sled, etc.) is a new impl of the same trait — no adapters, no shims.
- **Crash-safe.** Every write is durable on return. A `kill -9` at any point leaves the
  store in a consistent state. redb's ACID transactions provide this.
- **Height is caller-provided.** The store maintains a height index but never derives
  height from the data. The caller knows the domain semantics.

## Modifier Types

| Type ID | Name | Introduced |
|---------|------|-----------|
| 101 | Header | Phase 1 (now) |
| 102 | BlockTransactions | Future |
| 104 | ADProofs | Future |
| 108 | Extension | Future |

The store handles all types identically. Type ID selects the table/bucket.

## Trait: `ModifierStore`

```rust
pub trait ModifierStore: Send + Sync {
    type Error: std::error::Error + Send + Sync + 'static;

    /// Store a single modifier.
    fn put(
        &self,
        type_id: u8,
        id: &[u8; 32],
        height: u32,
        data: &[u8],
    ) -> Result<(), Self::Error>;

    /// Store a batch of modifiers atomically.
    /// All entries are written in a single transaction — all succeed or none do.
    fn put_batch(
        &self,
        entries: &[(u8, [u8; 32], u32, Vec<u8>)],
    ) -> Result<(), Self::Error>;

    /// Retrieve a modifier by type and ID.
    fn get(
        &self,
        type_id: u8,
        id: &[u8; 32],
    ) -> Result<Option<Vec<u8>>, Self::Error>;

    /// Retrieve the modifier ID at a given height for a type.
    fn get_id_at(
        &self,
        type_id: u8,
        height: u32,
    ) -> Result<Option<[u8; 32]>, Self::Error>;

    /// Check whether a modifier exists without reading its data.
    fn contains(
        &self,
        type_id: u8,
        id: &[u8; 32],
    ) -> Result<bool, Self::Error>;

    /// Returns the tip (highest height and its modifier ID) for a type.
    /// None if no modifiers of that type have been stored.
    fn tip(
        &self,
        type_id: u8,
    ) -> Result<Option<(u32, [u8; 32])>, Self::Error>;
}
```

## Indexes

Two indexes per modifier type:

| Index | Key | Value | Purpose |
|-------|-----|-------|---------|
| Primary | `(type_id, id)` | `data` | Lookup by modifier ID |
| Height | `(type_id, height)` | `id` | Lookup by height, startup chain rebuild |

Tip is derived: `put` and `put_batch` update an in-memory tip if `height > current_tip`.
On startup, tip is read from the height index (max key).

## Preconditions

- **`put` / `put_batch`**: `data` is non-empty. `id` is the canonical modifier ID
  (Blake2b256 of the serialized bytes for headers). `height` is the block height
  this modifier belongs to. The caller has already validated the modifier.
- **`get` / `get_id_at` / `contains` / `tip`**: No preconditions beyond valid type_id.

## Postconditions

- **`put`**: On Ok, the modifier is durable on disk. A subsequent `get` with the same
  `(type_id, id)` returns `Some(data)`. `get_id_at(type_id, height)` returns `Some(id)`.
  If `height > tip(type_id).height`, tip is updated.
- **`put_batch`**: Same as `put` for each entry, atomically. On Err, no entries are written.
- **`get`**: Returns the stored bytes or None. Never errors on missing data.
- **`get_id_at`**: Returns the modifier ID at that height or None.
- **`contains`**: Equivalent to `get(..).map(|o| o.is_some())` but avoids reading data.
- **`tip`**: Returns the highest `(height, id)` pair stored for that type, or None.

## Invariants

- No method panics. All failures are returned as errors.
- A modifier written by `put` is immediately visible to `get`, `contains`, and `tip`.
- `put_batch` is atomic — partial writes never occur.
- The height index is consistent with the primary index. If `get_id_at(t, h)` returns
  `Some(id)`, then `get(t, &id)` returns `Some(data)`.
- The store survives `kill -9` at any point. On restart, it contains exactly the
  modifiers from completed transactions.
- Duplicate `put` with the same `(type_id, id)` is idempotent — overwrites with same data,
  no error.

## Does NOT own

- Modifier parsing or validation — that's the pipeline and chain crates.
- Deciding what to store or when — that's the pipeline.
- Chain state reconstruction — that's the chain crate, using data read from the store.
- Network I/O — that's P2P.

## Dependencies

- `redb` — storage backend (initial implementation)
- No dependency on `ergo-chain-types`, `ergo-lib`, or any Ergo domain crates.

## SPECIAL Profile

```
S7  P5  E9  C6  I7  A8  L7
```

Internal component — no untrusted input (callers validate first). Durability and
performance are the primary concerns. See `facts/SPECIAL.md`.

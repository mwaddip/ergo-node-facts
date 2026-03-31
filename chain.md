# Header Chain Contract

## Component: `chain/` (enr-chain)

Owns header parsing, tracking, PoW verification, difficulty adjustment, and
header chain validation. The single authority on "is this chain of headers valid?"

Primary consumers: P2P layer (feeds raw bytes), sync state machine (queries chain state),
block validation (checks header membership).

## Phase 1: Header Awareness

### `parse_header(body: &[u8]) -> Result<Header>`
- **Precondition**: `body` is the raw payload from a ModifierResponse with modifier_type = 101 (Header).
- **Postcondition**: Returns a fully populated `ergo-chain-types::Header` or an error.
  Never panics on malformed input.

### `HeaderTracker`

Stateless observer. Tracks headers seen on the network without validating chain linkage.

#### `observe(header: &Header)`
- **Postcondition**: If `header.height > best_height()`, updates best known tip.
- **Invariant**: Best tip is always the highest header observed.

#### `best_height() -> Option<u32>`
- Returns the height of the highest header seen, or None if no headers observed.

#### `best_header_id() -> Option<HeaderId>`
- Returns the ID of the highest header seen.

## Phase 2: PoW Verification

### `verify_pow(header: &Header) -> Result<()>`
- **Precondition**: Header is parsed (Phase 1).
- **Postcondition**: Ok if `pow_hit(header) <= target(header.n_bits)`. Err otherwise.
- **Uses**: `ergo-chain-types::AutolykosPowScheme::pow_hit()`.
- **Cost**: One hash computation. Cheap enough to call on every header before forwarding.
- **Invariant**: A header that fails PoW is never valid regardless of chain context.

## Phase 3: Header Chain Validation

### `HeaderChain`

Maintains a validated chain of headers. Every header in the chain has been checked for:
parent linkage, PoW validity, timestamp bounds, and correct difficulty.

#### `try_append(header: Header) -> Result<()>`
- **Precondition**: Header is parsed and PoW-verified.
- **Postcondition on Ok**: Header is added to the chain. `height()` may increase.
- **Postcondition on Err**: Chain is unchanged. Error describes which check failed.
- **Validates**:
  - `header.parent_id` matches an existing header in the chain
  - `header.timestamp` is within acceptable bounds relative to parent
  - `header.n_bits` matches the expected difficulty for this height (see difficulty adjustment)
  - PoW is valid for the claimed difficulty

#### `height() -> u32`
- Returns the height of the best validated chain tip.

#### `header_at(height: u32) -> Option<&Header>`
- Returns the header at the given height on the best chain, if it exists.

#### `tip() -> &Header`
- **Precondition**: Chain is non-empty (at least genesis or bootstrap point).
- Returns the tip header of the best validated chain.

#### `contains(header_id: &HeaderId) -> bool`
- Returns whether this header ID is part of the validated chain.

#### `headers_from(height: u32, count: usize) -> Vec<&Header>`
- Returns up to `count` sequential headers starting at `height`.
- Used by sync to serve header chains to peers.

### Difficulty Adjustment

#### `expected_difficulty(parent: &Header, chain: &HeaderChain) -> Result<u64>`
- **Precondition**: `parent` is in the chain.
- **Postcondition**: Returns the required nBits for the next header after `parent`.
- **Algorithm**: Epoch-based recalculation. Port from JVM `ergo-core` DifficultyAdjustment.
- **Invariant**: For any two correct implementations given the same chain, the output is identical.
  This is consensus-critical — must match the JVM node exactly.

## Invariants (all phases)

- No method panics on untrusted input.
- `HeaderChain` is append-only for the best chain. Forks are tracked but the best chain
  is selected by cumulative difficulty.
- The chain never contains two headers at the same height on the same fork.
- All timestamps are treated as untrusted data. Timing logic uses block height.

## Does NOT own

- Block bodies, transactions, AD proofs — that's `ergo-validation`.
- Persisting headers to disk — that's `store/`.
- Deciding *when* to request headers — that's `ergo-sync`.
- Network I/O — that's `p2p/`.

## Dependencies

- `ergo-chain-types` — Header struct, Autolykos PoW, compact nBits encoding
- `ergo-nipopow` — NiPoPoW proof verification (Phase 3, light client bootstrap)
- `sigma-ser` — Scorex deserialization for header bytes

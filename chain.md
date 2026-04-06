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

## SyncInfo Serialization

Build and parse SyncInfo messages (P2P message code 65). Used by the sync
state machine to compare chain tips with peers.

Two wire formats exist. V2 is used by all current nodes (>= 4.0.16).

### `build_sync_info(headers: &[Header]) -> Vec<u8>`
- **Precondition**: `headers` contains up to 50 recent headers from the chain tip.
- **Postcondition**: Returns V2 SyncInfo body bytes ready for framing.
- **Format**: `[0x00, 0x00][0xFF][count: 1 byte][header_size: VLQ u16, header_bytes] × count`
- The `[0x00, 0x00]` prefix (count=0 in V1 framing) signals V2 to older parsers.

### `parse_sync_info(body: &[u8]) -> Result<SyncInfo>`
- **Precondition**: `body` is the raw payload from a SyncInfo message (code 65).
- **Postcondition**: Returns parsed sync info — either V1 (header IDs only) or V2 (full headers).
- Never panics on malformed input.
- Rejects V2 messages with more than 50 headers or headers larger than 1000 bytes.

### `SyncInfo` enum
- `V1 { header_ids: Vec<BlockId> }` — legacy, list of 32-byte header IDs
- `V2 { headers: Vec<Header> }` — current, full serialized headers

## Invariants (all phases)

- No method panics on untrusted input.
- `HeaderChain` is append-only for the best chain. Forks are tracked but the best chain
  is selected by cumulative difficulty.
- The chain never contains two headers at the same height on the same fork.
- All timestamps are treated as untrusted data. Timing logic uses block height.

## State Type

### `StateType` enum
- `Utxo` — maintain the full UTXO set, validate transactions by direct input lookup.
  Does not need AD proofs.
- `Digest` — maintain only the AVL+ tree root hash, validate state transitions via
  authenticated dictionary proofs (AD proofs). Requires downloading AD proofs from peers.

Mirrors JVM's `StateType` enum. Determines which block sections are required for
download and validation.

### `StateType::requires_proofs() -> bool`
- Returns `true` for `Digest`, `false` for `Utxo`.
- Mirrors JVM's `stateType.requireProofs`.

## Block Section IDs

### `section_ids(header: &Header) -> [(u8, [u8; 32]); 3]`
- **Precondition**: Header is parsed.
- **Postcondition**: Returns the modifier IDs for all three non-header block sections:
  - `(102, Blake2b256(102 || header.id || header.transaction_root))` — BlockTransactions
  - `(104, Blake2b256(104 || header.id || header.ad_proofs_root))` — ADProofs
  - `(108, Blake2b256(108 || header.id || header.extension_root))` — Extension
- **Pure computation**. No I/O, no state.
- Matches JVM `Header.sectionIds`.

### `required_section_ids(header: &Header, state_type: StateType) -> Vec<(u8, [u8; 32])>`
- **Precondition**: Header is parsed.
- **Postcondition**: Returns modifier IDs for sections required by the given state type:
  - `Utxo` → BlockTransactions + Extension (2 entries). Matches JVM `Header.sectionIdsWithNoProof`.
  - `Digest` → all three including ADProofs (3 entries). Matches JVM `Header.sectionIds`.
- Mirrors JVM's `ToDownloadProcessor.requiredModifiersForHeader`.

## Phase 6: Soft-Fork Voting

Track and apply blockchain parameter changes voted on by miners. The vote
counting and parameter computation are consensus-critical: a receiver MUST
independently compute the new parameters from the previous epoch's votes
and verify they match the parameters in the next epoch-boundary block's
extension. Mismatch = reject the block. Get this wrong and the node forks.

Soft-fork voting (BlockVersion bumps) requires multi-epoch state machinery:
voting period → activation period → version increment. The activation
machinery is part of consensus and must be implemented in full.

JVM reference: `ergo-core/src/main/scala/org/ergoplatform/settings/Parameters.scala`,
`VotingSettings.scala`. Read these before implementing.

### `VotingConfig`

```rust
pub struct VotingConfig {
    /// Length of one voting epoch in blocks. Mainnet: 1024. Testnet: 128.
    pub voting_length: u32,
    /// Voting epochs collected before a soft-fork can be approved. Both nets: 32.
    pub soft_fork_epochs: u32,
    /// Voting epochs after approval before BlockVersion is incremented. Both nets: 32.
    pub activation_epochs: u32,
    /// JVM hard-coded protocol-v2 forced activation height. Mainnet: 417792.
    pub version2_activation_height: u32,
}
```

Derived from network type — not a runtime config entry. The chain submodule
selects testnet vs mainnet values internally.

### `ActiveParameters`

The set of parameters currently in effect at the chain tip. Used by the
validator to bound transaction costs and by the mining task when assembling
epoch-boundary blocks.

Wraps `ergo_lib::chain::parameters::Parameters` (already used by `validation/`).
Chain owns the live instance; consumers query it via `active_parameters()`.

### `active_parameters() -> &Parameters`
- **Postcondition**: Returns the parameters in effect at the current tip.
- **Invariant**: The returned parameters were computed from the chain history
  ending at the current tip. After every successfully appended epoch-boundary
  block, this value advances to the new params.
- **Startup**: Recomputed from chain history during construction (see
  "Startup recomputation" below).

### `compute_expected_parameters(epoch_boundary_height: u32) -> Result<Parameters>`
- **Precondition**: `epoch_boundary_height` is the height of an epoch-boundary
  block (the FIRST block of a new epoch). The chain must contain all headers
  in the just-ended voting epoch (`[epoch_boundary_height - voting_length, epoch_boundary_height - 1]`).
- **Postcondition**: Returns the parameters that the block at
  `epoch_boundary_height` MUST emit in its extension. If the block's extension
  contains different parameters, the block is invalid.
- **Algorithm**: Port of JVM `Parameters.update`:
  1. Tally votes from headers in the just-ended voting epoch (see
     `count_votes_in_epoch`).
  2. Apply ordinary parameter changes (IDs 1-8): for each tallied param, if
     `count > voting_length / 2` (`changeApproved`), apply one step from
     `Parameters.stepsTable` clamped by `minValues`/`maxValues`. Positive
     param ID = increase, negative ID = decrease.
  3. Apply soft-fork machinery (see "Soft-fork lifecycle" below).
  4. Return the new `Parameters` table.
- **Determinism**: For any two correct implementations given the same chain,
  the output is byte-identical. This is the consensus rule.

### `count_votes_in_epoch(epoch_end_height: u32) -> Result<HashMap<i8, u32>>`
- **Precondition**: All headers in `[epoch_end_height - voting_length + 1, epoch_end_height]`
  are present in the chain.
- **Postcondition**: Returns the per-paramId vote count, summed across all
  three vote slots in each header's `votes` field. Each slot is one signed
  byte: positive = increase, negative = decrease, 0 = no vote, 120 = SoftFork.
- **Helper for `compute_expected_parameters`**, exposed for testability.

### `apply_epoch_boundary_parameters(params: Parameters)`
- **Precondition**: `params` was returned by `compute_expected_parameters` for
  the height of the just-validated epoch-boundary block AND was confirmed to
  match the params parsed from that block's extension.
- **Postcondition**: `active_parameters()` returns `params`.
- **Called by**: validator after the epoch-boundary block has passed all checks.

### `is_epoch_boundary(height: u32) -> bool`
- **Postcondition**: Returns true iff `height % voting_length == 0` AND
  `height > 0`. Matches JVM's `(height % votingEpochLength == 0)`.
- **Pure computation**, no chain access. Used by validator and mining task.

### Soft-fork lifecycle

The soft-fork machinery uses three reserved param IDs:

| ID | Name | Purpose |
|----|------|---------|
| 120 | `SoftFork` | Vote slot value (not stored in parametersTable) |
| 121 | `SoftForkVotesCollected` | Running tally of soft-fork votes since voting started |
| 122 | `SoftForkStartingHeight` | Height at which the current vote period began |
| 123 | `BlockVersion` | Current block version. Bumped on successful activation. |

State transitions inside `compute_expected_parameters` (port of
`Parameters.updateFork`):

1. **Successful voting cleanup**: at
   `softForkStartingHeight + votingLength * (softForkEpochs + activationEpochs + 1)`
   AND `softForkApproved`, remove IDs 121 and 122 from the table.
2. **Unsuccessful voting cleanup**: at
   `softForkStartingHeight + votingLength * (softForkEpochs + 1)` AND NOT
   `softForkApproved`, remove IDs 121 and 122 from the table.
3. **New voting start**: when fork vote present AND no current voting OR
   prior voting cleanup just happened, set IDs 122 = current height, 121 = 0.
4. **Mid-voting epoch**: when `height <= startingHeight + votingLength * softForkEpochs`,
   add the new epoch's fork votes to ID 121.
5. **Activation**: at
   `softForkStartingHeight + votingLength * (softForkEpochs + activationEpochs)`
   AND `softForkApproved`, increment ID 123 (BlockVersion).
6. **Forced v2 activation**: at `version2_activation_height`, force ID 123 = 2 if
   currently 1. Mainnet hard-fork that pre-dates the voting machinery.

`softForkApproved(votes) = votes > voting_length * soft_fork_epochs * 9 / 10`
(90% supermajority across all soft-fork voting epochs).

### Startup recomputation

On `HeaderChain::open()` (or equivalent constructor that loads from store),
the chain MUST rebuild `active_parameters()` before returning:

1. Walk backwards from the current tip to find the most recent epoch-boundary
   block (highest height H where `H % voting_length == 0`).
2. Read that block's extension from `store/`.
3. Parse parameters from the extension via JVM-equivalent logic (key prefix
   `0x00` + 1-byte param ID + 4-byte BE i32 value, except ID 124
   `SoftForkDisablingRules` which has variable-length encoding).
4. Set `active_parameters` to the parsed value.

If no epoch-boundary block exists yet (chain shorter than one epoch),
`active_parameters` returns `Parameters::default()`.

Cost: bounded — at most `voting_length` blocks of backward walk, plus one
extension read. Acceptable at startup. NOT cached to disk: derived state,
divergence-prone.

### Voting invariants

- `active_parameters` advances ONLY at epoch-boundary blocks. Within an
  epoch it is constant.
- The receiver MUST verify that the params in an epoch-boundary block's
  extension match `compute_expected_parameters(block.height)` byte-for-byte
  via `Parameters.matchParameters`. Mismatch = consensus failure.
- After reorg past an epoch-boundary block, the chain MUST roll back
  `active_parameters` to the params at the new tip (recompute or store
  per-height snapshots).

## Phase 6: NiPoPoW Proofs (verify + serve)

Build NiPoPoW proofs from the local chain on request, and verify proofs
received from peers. Wraps `ergo-nipopow`. Does NOT include light-client
sync mode (a separate session) — proofs are verified for correctness but
not used to skip block download.

JVM reference: `ergo-core/src/main/scala/org/ergoplatform/modifiers/history/popow/NipopowProof.scala`
and `NipopowAlgos.scala`.

### `build_nipopow_proof(m: u32, k: u32, header_id: Option<HeaderId>) -> Result<Vec<u8>>`
- **Precondition**: Chain is non-empty. `header_id`, if provided, must be in
  the chain (proof is built from the chain ending at that header). If `None`,
  the proof is built from the current tip.
- **Postcondition**: Returns the inner serialized NiPoPoW proof bytes (no P2P
  envelope wrapping). The bytes are exactly what JVM's
  `NipopowProofSerializer.toBytes(proof)` produces — the main crate prepends
  the message envelope when sending.
- **Algorithm**: Use `ergo_nipopow::NipopowAlgos::prove` (or equivalent) on
  the chain's `PoPowHeader` sequence, with security parameters m (min
  μ-level superchain length) and k (min suffix length, ≥ 1).
- **Cost**: Walks the full chain. Cap m + k at sane values. Reject if the
  chain has fewer than k blocks.
- **Determinism**: For any two correct implementations on the same chain,
  the output is byte-identical.

### `verify_nipopow_proof_bytes(bytes: &[u8]) -> Result<NipopowProofMeta>`
- **Precondition**: `bytes` is the inner NiPoPoW proof payload (the main
  crate has already stripped the message envelope).
- **Postcondition**: Returns `NipopowProofMeta` if the proof is structurally
  valid AND `is_valid` returns true (heights consistent, connections valid,
  PoW valid for each header, difficulty headers present in continuous mode).
- **Validation checks** (mirrors `NipopowProof.isValid`):
  1. Headers parse cleanly via `ergo_nipopow::NipopowProofSerializer`.
  2. Heights are strictly increasing across `headersChain`.
  3. Each header's PoW passes `verify_pow`.
  4. Parent connections in the chain are consistent.
  5. (Continuous mode only) Difficulty-recalculation headers are present.
- **Does NOT** apply the proof to local chain state — that's the future
  light-client mode.

### `NipopowProofMeta`

```rust
pub struct NipopowProofMeta {
    /// Height of the suffix tip (the highest header in the proof).
    pub suffix_tip_height: u32,
    /// Total number of headers in the proof (prefix + suffix).
    pub total_headers: usize,
    /// Whether the proof is in continuous mode (carries difficulty headers).
    pub continuous: bool,
}
```

Returned by `verify_nipopow_proof_bytes` so the caller can log and (in the
future) compare against local chain state.

### NiPoPoW invariants

- `build_nipopow_proof` and `verify_nipopow_proof_bytes` are pure functions
  over chain state (modulo `&self` for chain access in `build`).
- Building and verifying do NOT modify chain state.
- Verification rejects any proof whose internal PoW checks fail —
  consensus-critical.
- Building never produces a proof that would fail verification on the same
  implementation.

## Does NOT own

- Block bodies, transactions, AD proofs — that's `ergo-validation`.
- Persisting headers to disk — that's `store/`.
- Deciding *when* to request headers — that's `ergo-sync`.
- Network I/O — that's `p2p/`.
- P2P message envelope wrapping for NiPoPoW (codes 90/91) — the inner
  proof bytes are produced/consumed here, but the message envelope is
  the main crate's responsibility (mirrors snapshot sync).
- Validator wiring of `active_parameters` — `validation/` calls into chain.
- Soft-fork voting policy (which params to vote for) — that's the mining
  config in the main crate.

## Dependencies

- `ergo-chain-types` — Header struct, Autolykos PoW, compact nBits encoding
- `ergo-nipopow` — NiPoPoW proof construction and verification
- `ergo-lib` — `chain::parameters::Parameters` for voting state (already a
  validation dependency upstream; pulling it in here unifies the type)
- `sigma-ser` — Scorex deserialization for header bytes

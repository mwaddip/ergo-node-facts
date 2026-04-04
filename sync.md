# Sync State Machine Contract

## Component: `ergo-sync` (workspace crate)

Owns the chain synchronization protocol. Drives the P2P layer to request data,
feeds validated results to the chain/state components, and tracks sync progress.

Primary dependencies: P2P layer (send/receive messages), chain (header state and validation),
pipeline (progress notifications).

## Traits (dependency inversion)

The sync crate does NOT depend on concrete P2P or chain types. It defines traits
that the main crate satisfies with the real implementations.

### `SyncTransport`

How the sync machine sends messages and observes the network.

#### `send_to(peer, message) -> Result<()>`
- Send a protocol message to a specific peer.

#### `outbound_peers() -> Vec<PeerId>`
- Currently connected outbound peers.

#### `next_event() -> Option<ProtocolEvent>`
- Receive the next incoming protocol event. Returns None if the stream ends.

### `SyncStore`

How the sync machine queries persistent storage.

#### `has_modifier(type_id, id) -> bool`
- Returns true if the modifier exists in the store.
- Used to determine which block sections need downloading.
- Must not block the async runtime (the bridge impl handles this).

### `SyncChain`

How the sync machine queries and updates chain state.

#### `chain_height() -> u32`
- Height of the validated chain tip.

#### `header_at(height: u32) -> Option<Header>`
- Return the header at a given height, if it exists in the chain.
- Used by the sync machine to compute block section IDs for download.

#### `build_sync_info() -> Vec<u8>`
- Build a V2 SyncInfo body from current chain state.
- Includes headers at offsets `[0, 16, 128, 512]` from tip, ordered tip-first.

#### `parse_sync_info(body: &[u8]) -> Result<SyncInfo>`
- Parse an incoming SyncInfo message.

#### `sync_info_heights(info: &SyncInfo) -> Vec<u32>`
- Extract heights from a parsed SyncInfo (V2 only).

## HeaderSync

### `HeaderSync::new(config, transport, chain, store, validator, progress, delivery_control, delivery_data) -> Self`
- Create the sync state machine with injected dependencies.
- `config`: `SyncConfig` with timing parameters, delivery settings, and `state_type`.
  The `state_type` field (`StateType::Utxo` or `StateType::Digest`) determines which
  block sections are queued for download via `required_section_ids`.
- `store`: `SyncStore` for checking modifier existence
- `validator`: `BlockValidator` for digest/UTXO-mode block validation
- `progress`: `mpsc::Receiver<u32>` from the validation pipeline. Carries chain
  height after each validated batch. Used for stall detection and the two-batch
  SyncInfo pattern.
- `delivery_control`: `mpsc::UnboundedReceiver<DeliveryControl>` from the pipeline.
  Carries `Reorg` and `NeedModifier` events. These are rare and critical — losing
  one is unrecoverable. Unbounded channel, never dropped.
- `delivery_data`: `mpsc::Receiver<DeliveryData>` from the pipeline. Carries
  `Received` and `Evicted` modifier notifications. High-volume, lossy — missing
  one just delays the watermark scan by one timer tick. Bounded channel (capacity 64),
  sent via `try_send`.

### `HeaderSync::run() -> !`
- Long-running async task. Drives the sync loop until the runtime shuts down.

## Architecture: Event-driven loop

Four event sources via `tokio::select!` with `biased;` (control checked first):

1. **Control-plane events** — `DeliveryControl::Reorg` and `DeliveryControl::NeedModifier`
   from the pipeline. Checked with priority via `biased;` — these are never dropped.
2. **P2P events** — Inv (header IDs from peer), SyncInfo (peer's chain state), peer disconnect
3. **Data-plane events** — `DeliveryData::Received` and `DeliveryData::Evicted` from the
   pipeline. Lossy — the delivery timer provides fallback scanning.
4. **Pipeline progress** — `mpsc::Receiver<u32>` carrying chain height after each validated batch
5. **Sync timer** — fires every 20 seconds (matching JVM's `MinSyncInterval`)

## Sync cycle

```
pick_sync_peer() → sync_from_peer() → synced()
       ↑                  │
       └──── stall/disconnect ────┘
```

### sync_from_peer()

1. Send SyncInfo to peer (kick off exchange)
2. Event loop:
   - P2P Inv with header IDs → send ModifierRequest to announcing peer
   - Pipeline progress → one SyncInfo per 20s cycle (two-batch pattern)
   - 20s timer → send SyncInfo (scheduled cycle start)
   - Stall timeout (60s no progress) → rotate to different peer

### Peer rotation

`stalled_peers: HashSet<PeerId>` tracks peers that failed to produce progress.
On stall: add current peer, pick next outbound peer not in set.
On progress: clear the set (all peers eligible again).
If all peers stalled: clear set, retry.

### Multi-peer switching

When any peer's SyncInfo shows a chain tip more than 1 block ahead of ours,
the sync machine switches to syncing from that peer (`BehindPeer` event).
The "caught up" check only triggers from the current sync peer — other peers
reporting lower tips don't cause a false synced state.

### Deep reorg support

The pipeline handles fork detection and chain reorganization. When a fork header
arrives that creates a better chain (higher cumulative difficulty), the pipeline:

1. Stores the fork header with its score via the store's fork-aware tables.
2. Assembles the fork branch by walking parent links backward through the store.
3. Executes `HeaderChain::try_reorg_deep()` to atomically swap the best chain.
4. Sends `DeliveryControl::Reorg { fork_point, old_tip, new_tip }` to the sync machine.

The sync machine responds by clearing its section queue, resetting both watermarks
(`downloaded_height` and `validated_height`) to the fork point, resetting the
block validator's state root, re-queuing sections for the new branch, and
re-scanning the download watermark.

For incomplete fork chains (parent not in store), the pipeline sends
`DeliveryControl::NeedModifier` to request the missing parent header. Once it
arrives, the fork chain links backward and triggers the reorg if the score is
sufficient. See `facts/reorg.md` for the full contract.

### Two-batch pattern

The JVM gets ~800 headers per 20-second cycle by sending SyncInfo twice: once
from the scheduled timer, once after the first batch is processed. The pipeline
progress channel enables this — when a batch finishes, one progress-triggered
SyncInfo is allowed per cycle.

### Synced state

Periodic SyncInfo (30s) to detect new blocks. Reacts to Inv with ModifierRequest.
Receives pipeline progress for logging. The control channel is checked with
`biased;` priority in the synced loop too — a `Reorg` while synced must not be missed.

## JVM peer behavior (observed)

Critical findings from debugging sync against JVM 6.0.3 peers:

- **MinSyncInterval**: 20 seconds per peer. Sending faster is silently dropped.
- **PerPeerSyncLockTime**: 100ms. Incoming SyncInfo within 100ms of previous is dropped as "spammy."
- **~12 batches per connection**: after ~12 SyncInfo exchanges, the JVM stops processing our SyncInfo on that connection. The message is received (visible in JVM debug logs) but not forwarded to `processSync`. Reconnection starts a new session.
- **Delivery tracker essential**: the JVM uses a 10-second delivery tracker to retry failed requests. Without it, lost responses are never recovered. Our sync relies on the 60-second stall timeout — 6x slower recovery.
- **Single peer per cycle**: the JVM syncs from one Older peer per 20-second cycle, not all peers simultaneously.
- **SyncInfo response**: the JVM responds to incoming SyncInfo with its own SyncInfo when `syncSendNeeded` (status changed, peer outdated, status=Older/Fork). We don't respond during active sync to avoid sending stale chain state.

## Block Section Download

After header sync reaches the peer's tip, the sync machine downloads block sections
for stored headers. Which sections are downloaded depends on `SyncConfig.state_type`:
- **UTXO mode**: BlockTransactions (102) + Extension (108). No AD proofs.
- **Digest mode**: all three including ADProofs (104).

This mirrors the JVM's `ToDownloadProcessor.requiredModifiersForHeader`, which calls
`Header.sectionIdsWithNoProof` in UTXO mode and `Header.sectionIds` in digest mode.

### Download queue

The sync machine maintains an internal queue of `(type_id, modifier_id)` pairs
for block sections that need downloading. The queue is populated from two sources:

1. **On header progress**: when the pipeline reports new chained headers, compute
   section IDs for each new header (via `chain::required_section_ids` with the
   configured state type), check the store (`SyncStore::has_modifier`), and
   enqueue any missing sections.

2. **On startup**: walk stored headers from height 1 to tip, compute section IDs,
   check the store, enqueue what's missing. One-time startup cost.

### Download cycle

Block section requests follow the same pattern as header requests:
- Send `ModifierRequest` with the section type and IDs
- Track delivery via the `DeliveryTracker`
- On timeout: re-request from a different peer
- On receive: the pipeline stores the bytes (no validation)

### Inv handling

The JVM may send Inv with non-header type IDs (102, 104, 108). The sync machine
should react to these the same way as header Inv: request the listed modifiers
and track delivery.

### Prioritization

Header sync takes priority. Block section download starts only after header sync
reaches the `Synced` state. During active header sync, block section requests
are paused to avoid saturating the peer's bandwidth.

## Block Assembly (downloaded_height / validated_height)

The sync machine tracks two watermarks:

- **`downloaded_height`** — the highest height where all required block sections
  are present in the store. Blocks at or below this height are ready for validation.
- **`validated_height`** — the highest height where block sections have been
  validated by the `BlockValidator`. Always `<= downloaded_height`.

Both are initialized from `validator.validated_height()` on startup.

### Watermark scanner

`advance_downloaded_height()` scans forward from the current watermark. For each
height, it computes `required_section_ids(header, state_type)` and checks the
store for each. Advances as far as possible, stops at the first gap. On advance,
calls `advance_validated_height()` to run the block validator on newly downloaded blocks.

### Trigger points

1. **Startup**: after the section queue is built from stored headers.
2. **DeliveryData::Received**: after sections are stored by the pipeline.
3. **Delivery check timer**: every 5 seconds during active sync. This is the
   primary trigger — the data channel can overflow when sections arrive
   faster than the sync machine processes events, so the timer ensures the
   scanner runs regardless.
4. **Synced ticker**: every 30 seconds during the synced polling loop.

### Invariants

- `validated_height <= downloaded_height <= chain_height`
- `validated_height` is monotonically increasing (except on reorg reset)
- Heights at or below `validated_height` have verified state transitions

## Does NOT own

- Header validation — that's `enr-chain` via the validation pipeline
- Block section validation — that's `ergo-validation` via `BlockValidator` trait
- Persistent storage — that's `store/`
- Network I/O — that's `enr-p2p`
- Section ID computation — that's `enr-chain` (`section_ids()` / `required_section_ids()`)
- Fork choice / reorg execution — that's the pipeline (via `HeaderChain::try_reorg_deep`)

## Future Extensions

- UTXO state management coordination
- Parallel header download from multiple peers
- Turbo sync mode (adaptive batch sizes, see IDEAS.md)

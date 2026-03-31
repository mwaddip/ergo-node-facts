# Sync State Machine Contract

## Component: `ergo-sync` (workspace crate)

Owns the chain synchronization protocol. Drives the P2P layer to request data,
feeds validated results to the chain/state components, and tracks sync progress.
The orchestrator that turns a passive relay into an active node.

Primary dependencies: P2P layer (send/receive messages), chain (header state and validation),
store (persistent state, future).

## Traits (dependency inversion)

The sync crate does NOT depend on concrete P2P or chain types. It defines traits
that the main crate satisfies with the real implementations.

### `SyncTransport`

How the sync machine sends messages and observes the network.

#### `send_to(peer, message) -> Result<()>`
- Send a protocol message to a specific peer.

#### `broadcast_outbound(message)`
- Send to all outbound peers.

#### `outbound_peers() -> Vec<PeerId>`
- Currently connected outbound peers.

#### `subscribe() -> Receiver<ProtocolEvent>`
- Receive incoming protocol events.

### `SyncChain`

How the sync machine queries and updates chain state.

#### `chain_height() -> u32`
- Height of the validated chain tip.

#### `chain_tip_headers(offsets: &[u32]) -> Vec<Header>`
- Headers at the given offsets from the tip. Used to build SyncInfo.
- Returns only headers that exist (skips offsets beyond chain start).

#### `build_sync_info() -> Vec<u8>`
- Build a V2 SyncInfo body from current chain state.

#### `parse_sync_info(body: &[u8]) -> Result<SyncInfo>`
- Parse an incoming SyncInfo message.

## HeaderSync (v1: header-only sync)

### `HeaderSync::new(transport, chain) -> Self`
- Create the sync state machine with injected dependencies.

### `HeaderSync::run() -> !`
- Long-running async task. Drives the sync loop until the runtime shuts down.

### States

```
Idle ──────▶ Syncing ──────▶ Synced
  ▲            │                │
  │            ▼                ▼
  └──── peer disconnected ◀── periodic SyncInfo
                                (check for new blocks)
```

#### Idle
- No outbound peers available, or just started.
- Wait for peer connection event.
- On peer connected: build SyncInfo, send to peer → Syncing.

#### Syncing
- Actively requesting headers from a peer.
- Receive Inv (type 101) → send ModifierRequest for those header IDs.
- Track outstanding requests.
- When all requested headers delivered (observed via event subscriber):
  send another SyncInfo to check for more.
- If peer reports same tip / empty Inv → Synced.
- If peer stops responding (delivery timeout) → try another peer, stay Syncing.
- If peer disconnects → back to Idle (or try another peer).

#### Synced
- Chain tip matches network tip.
- Periodically send SyncInfo (every ~30s) to check for new blocks.
- If Inv arrives with new headers → request them, stay Synced.
- If peer disconnects and no outbound peers remain → Idle.

## Sync Protocol Flow

1. Build SyncInfo V2 with headers at offsets `[0, 16, 128, 512]` from tip
2. Send to outbound peer
3. Peer compares, responds with Inv of header IDs we don't have
4. Send ModifierRequest for those IDs
5. Peer responds with ModifierResponse containing serialized headers
6. Headers flow through normal P2P routing → validator validates → chain updates
7. Sync machine observes progress via event subscriber
8. Repeat from step 1 until tips match

## Does NOT own

- Header validation — that's `enr-chain` via the validator
- Persistent storage — that's `store/`
- Network I/O — that's `enr-p2p`
- Block body sync — future extension, not v1
- Fork choice / reorg handling — future extension, not v1

## Future Extensions (v2+)

- Block body download (type 102, 104, 108) after headers
- UTXO state management coordination
- Multiple sync modes (full / digest / UTXO snapshot)
- Parallel header download from multiple peers
- Fork detection and chain reorganization

# NiPoPoW Verify + Serve Contract (in-repo side)

## Scope

This contract covers the **in-repo (main crate) side** of NiPoPoW proof
handling. The chain submodule owns proof construction and verification
(see `facts/chain.md` Phase 6: NiPoPoW Proofs). The main crate owns the
P2P message envelope and event dispatch.

What ships in this contract:

1. Wire format for P2P message codes 90 (`GetNipopowProof`) and 91
   (`NipopowProof`).
2. A handler in `src/main.rs` that subscribes to `ProtocolEvent`s,
   filters for codes 90 and 91, parses the envelopes, calls into chain,
   and sends responses via `p2p.send_to`.
3. Verification of incoming proofs (logged but not yet acted on —
   light-client mode is a separate session).

What does NOT ship:

- Outbound `GetNipopowProof` requests from us (we don't ask peers for
  proofs yet — that's part of the future light-client mode).
- Light-client sync (skipping block download based on a verified proof).
- Modifications to the P2P submodule (router treats codes 90/91 as
  Unknown and forwards them — wasteful, see
  `project_unknown_message_forwarding` memory; fix is a follow-up).

## SPECIAL Profile

```
S9 P10 E6 C5 I7 A7 L9
```

P is max — every byte from a peer is adversarial. The size cap on code 91
is 2 MB; the parser must enforce it before allocating. L is high — wrong
m/k handling, malformed proofs, and oversized inputs are the failure
modes.

## P2P Wire Format

Two new message codes. Both reuse the 4-byte magic + 1-byte code +
4-byte length + 4-byte checksum + body framing already implemented in
the P2P transport (no transport changes).

| Code | Name | Direction (this release) | Max body size |
|------|------|---------------------------|---------------|
| 90 | `GetNipopowProof` | Receive only | 1000 bytes |
| 91 | `NipopowProof` | Send only | 2,000,000 bytes |

JVM reference (canonical):
- `ergo-core/src/main/scala/org/ergoplatform/network/message/GetNipopowProofSpec.scala`
- `ergo-core/src/main/scala/org/ergoplatform/network/message/NipopowProofSpec.scala`

### Code 90: `GetNipopowProof` body

```
m: i32 (ZigZag VLQ — JVM putInt)
k: i32 (ZigZag VLQ — JVM putInt)
header_id_present: u8 (raw byte: 0 or 1)
[if header_id_present == 1] header_id: 32 bytes
future_pad_length: u16 (VLQ — JVM putUShort)
[if future_pad_length > 0] padding: future_pad_length bytes (skipped)
```

**Critical**: `m` and `k` use `putInt` (ZigZag VLQ), NOT plain VLQ or BE
fixed-width. The lesson from snapshot sync (`facts/snapshot.md` line 49)
applies: JVM Scorex serializers always use VLQ. Verified in `GetNipopowProofSpec.scala`.

**Validation**:
- Total body size MUST be ≤ 1000 bytes (reject before allocating).
- `m` and `k` MUST both be > 0.
- `m + k` capped at a sane upper bound (suggest 1000) to prevent unbounded
  proof construction.
- `future_pad_length` ≤ remaining body bytes (else parse error).

### Code 91: `NipopowProof` body

```
proof_length: u32 (VLQ — JVM putUInt)
proof_bytes: [u8; proof_length]
future_pad_length: u16 (VLQ — JVM putUShort)
[if future_pad_length > 0] padding: future_pad_length bytes (skipped)
```

**Validation**:
- Total body size MUST be ≤ 2,000,000 bytes (reject before allocating).
- `proof_length` MUST be > 0 AND < 2,000,000.
- `proof_bytes` is the inner NiPoPoW proof — passed verbatim to
  `chain.verify_nipopow_proof_bytes(proof_bytes)`.

## Module: `src/nipopow_handler` (or inline in main.rs)

A subscribed task that consumes `ProtocolEvent`s and responds to NiPoPoW
messages. Mirrors the snapshot sync handler structure.

### `spawn_nipopow_handler(p2p: P2pNode, chain: Arc<Mutex<HeaderChain>>)`
- **Postcondition**: Spawns a tokio task that:
  1. Subscribes to `p2p.subscribe()`.
  2. On every `ProtocolEvent::Message { from, code, body }` with `code == 90`:
     - Parses `body` as `GetNipopowProofRequest`. On parse error, log + drop.
     - Acquires the chain lock, calls `chain.build_nipopow_proof(m, k, header_id_opt)`.
     - On success, wraps the resulting bytes in the code 91 envelope and
       calls `p2p.send_to(from, ProtocolMessage::Custom(91, body))`.
     - On error, log + drop. (Do NOT send an error message — the JVM doesn't
       expect one and will time out instead.)
  3. On every `ProtocolEvent::Message { from, code, body }` with `code == 91`:
     - Parses `body` as `NipopowProofResponse` (extracts the inner proof bytes).
     - Acquires the chain lock, calls `chain.verify_nipopow_proof_bytes(...)`.
     - Logs the result. Does NOT mutate state. (Light-client mode is the
       follow-up session that will use this hook.)
- **Invariant**: The handler never blocks the event loop — long-running chain
  operations happen on a `tokio::task::block_in_place` boundary or in a
  spawned task, mirroring the existing pattern from snapshot sync.

### `parse_get_nipopow_proof(body: &[u8]) -> Result<GetNipopowProofRequest, NipopowError>`
### `parse_nipopow_proof(body: &[u8]) -> Result<NipopowProofResponse, NipopowError>`
### `serialize_nipopow_proof(proof_bytes: &[u8]) -> Vec<u8>`
### `serialize_get_nipopow_proof(req: &GetNipopowProofRequest) -> Vec<u8>` (future, light-client)

```rust
pub struct GetNipopowProofRequest {
    pub m: i32,
    pub k: i32,
    pub header_id: Option<HeaderId>,
}

pub struct NipopowProofResponse {
    pub proof_bytes: Vec<u8>,
}

pub enum NipopowError {
    BodyTooLarge { size: usize, max: usize },
    InvalidParameters,
    Truncated,
    ChainError(String),
}
```

### Sigma-rust integration

The chain submodule's `build_nipopow_proof` and `verify_nipopow_proof_bytes`
wrap `ergo_nipopow::NipopowAlgos` and `ergo_nipopow::NipopowProofSerializer`.
The main crate does NOT import `ergo-nipopow` directly — that's the chain's
job. The main crate only sees `Vec<u8>` and the `NipopowProofMeta` struct
returned from `verify_nipopow_proof_bytes`.

## Routing behavior (current limitation)

The P2P router treats unknown message codes as "forward to all peers of
opposite direction" (`facts/p2p-routing.md` line 48). This means:

- When peer A sends us `GetNipopowProof`, the router forwards a copy to
  every outbound peer in addition to delivering it to our subscriber.
- When peer A sends us `NipopowProof`, same — we get it AND every
  outbound peer gets it.

This is wasteful but harmless: we still see the events via `subscribe()`
and respond correctly. The forwarded copies are extra noise on the
network. Tracked in memory `project_unknown_message_forwarding` as a
follow-up.

For first release, **accept the waste**. Document it in the release notes.

## Configuration

A new top-level toggle in `node.toml`:

```toml
[node.nipopow]
serve = true       # respond to GetNipopowProof requests
verify = true      # verify incoming NipopowProof messages (log only for now)
max_proof_bytes = 2000000  # safety cap, mirrors JVM SizeLimit
```

Defaults: both `true`. The cap should NOT be configurable above 2,000,000
(JVM's hard limit). Lower values are valid for resource-constrained nodes.

## Test plan

1. **Unit**: `parse_get_nipopow_proof` rejects bodies > 1000 bytes.
2. **Unit**: `parse_get_nipopow_proof` round-trips through `serialize_get_nipopow_proof`.
3. **Unit**: `parse_nipopow_proof` rejects bodies > 2,000,000 bytes.
4. **Unit**: VLQ parsing matches the JVM byte sequence (capture from a real
   JVM `GetNipopowProof` message via pcap, replay through our parser).
5. **Integration (serve)**: Have the JVM testnet peer request a NiPoPoW
   proof from our Rust node. Verify the JVM logs `is_valid = true` and
   does not disconnect.
6. **Integration (verify)**: Inject a malformed proof (truncated middle,
   bad PoW on a header, height inversion). Verify the parser/verifier
   rejects each case.
7. **Integration (verify, real)**: Capture a real `NipopowProof` from
   testnet traffic, replay through our verifier. Must accept.

## Cross-references

- `facts/chain.md` Phase 6: NiPoPoW Proofs — the chain side
- `facts/snapshot.md` — pattern reference (codes 76-81 use the same approach)
- `facts/p2p-node.md` — `subscribe()` and `send_to` API
- `facts/p2p-routing.md` line 48 — Unknown forwarding behavior
- Memory `project_unknown_message_forwarding` — follow-up to fix waste
- JVM: `ergo-core/src/main/scala/org/ergoplatform/network/message/GetNipopowProofSpec.scala`
- JVM: `ergo-core/src/main/scala/org/ergoplatform/network/message/NipopowProofSpec.scala`

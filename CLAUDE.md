# tenso.etzhayyim.com - Signal E2E Secure File Transfer

## Overview

Zero-knowledge file transfer service. Signal Protocol (X3DH + Double Ratchet) for key exchange, AES-256-GCM per-transfer file key, chunked encrypted blob upload to B2. Server never sees plaintext.

- **URL**: https://tenso.etzhayyim.com
- **API**: https://t3ns0f1l.etzhayyim.com/xrpc
- **Nanoid**: `t3ns0f1l`
- **Execution Tier**: T1 (MCP-Compose; T3 Worker planned when Signal crypto Worker is deployed)
- **Governance**: `confidential`

## Architecture

```
Sender Browser
  |
  +-- 1. generateFileKey() -> AES-256-GCM random
  +-- 2. chunk file (64 MiB blocks)
  +-- 3. encrypt each chunk: AES-256-GCM(fileKey, chunkN)
  +-- 4. uploadBlobDedup(encrypted chunk) -> R2 blobs/{transferDid}/{sha256}
  +-- 5. Signal X3DH: fetchPeerBundle(recipientDid) -> session
  +-- 6. encrypt(fileKey + manifest) via deriveFieldKey(did, transferId)
  +-- 7. createRecord("transferRequest", {status: "pending"})
  +-- 8. createRecord("fileManifest", {payload: "signal:v1:..."})
  |
  v  derive rule (auto)
PDS Commit Pipeline
  +-- cross-actor invoke -> recipient notification
  +-- social post -> completion announcement (on status:completed)
  +-- yabai invoke -> risk check (on error)
  |
  v  subscribeRepos trigger
Recipient Browser
  +-- 9.  decrypt fileManifest -> fileKey + chunk CIDs
  +-- 10. GET blobs/{cid} -> decrypt each chunk with fileKey
  +-- 11. reassemble -> plaintext file
  +-- 12. createRecord("transferLog", {status: "received"})
  |
  v  cron pipeline (hourly)
Purge Worker
  +-- expire_at < now -> R2.delete(chunks) + graph DELETE
```

## Signal Protocol Integration

- **Key Generation**: `generateIdentity(did, deviceId)` -> X25519 (DH) + Ed25519 (signing)
- **Key Exchange**: X3DH via `fetchPeerBundle(recipientDid)` -> shared secret
- **File Key Wrapping**: `deriveFieldKey(senderDid, transferId)` -> HKDF AES-256-GCM key
- **Field Encryption**: `encryptFieldVal(fileKey + manifest)` -> `signal:v1:{base64}`
- **Per-Chunk Encryption**: AES-256-GCM with random fileKey (NOT Signal - Signal wraps the fileKey only)
- **Forward Secrecy**: Double Ratchet ensures compromised key does not expose past transfers

## Collections (NSID)

| Collection | Purpose | Encryption |
|---|---|---|
| `com.etzhayyim.apps.tenso.transferRequest` | Transfer initiation + status | plaintext (metadata only) |
| `com.etzhayyim.apps.tenso.fileManifest` | fileKey + chunk CID list | `signal:v1:` field encrypt |
| `com.etzhayyim.apps.tenso.transferLog` | Audit trail (event_type, actor, timestamp) | plaintext |
| `com.etzhayyim.apps.tenso.accessControl` | Recipient permissions, download limits | `signal:v1:` field encrypt |

## 3-Tier Write

| Tier | Use | API |
|---|---|---|
| **1 Social** | Completion announcements | derive rule auto-generates `AppBskyFeedPost` |
| **2 Domain** | Transfer records, manifests, logs | `ComAtprotoRepoCreateRecord()` |
| **3 State** | Signal keys, user preferences, quotas | `Preferences()` |

## Graph Schema (kagami)

```sql
(:TransferRequest {
  vertex_id, transfer_id, sender_did, recipient_did,
  status, filename, size_bytes, mime_type,
  chunk_count, expire_at, created_at, updated_at,
  org_id, user_id, actor_id
})

(:FileManifest {
  vertex_id, transfer_id, chunk_count, chunk_size,
  total_size, encrypted_payload,
  created_at, org_id, user_id, actor_id
})

(:TransferLog {
  vertex_id, transfer_id, event_type, actor_did,
  ip_hash, user_agent_hash,
  created_at, org_id, user_id, actor_id
})

(:Actor)-[:SENT_TRANSFER]->(:TransferRequest)
(:Actor)-[:RECEIVED_TRANSFER]->(:TransferRequest)
(:TransferRequest)-[:HAS_MANIFEST]->(:FileManifest)
(:TransferRequest)-[:HAS_LOG]->(:TransferLog)
```

## Security Properties

| Property | Mechanism |
|---|---|
| E2E encryption | Signal X3DH + Double Ratchet -> per-transfer AES-256-GCM fileKey |
| Zero-knowledge server | B2 stores encrypted blobs only. PDS never has fileKey |
| Forward secrecy | Double Ratchet ratcheting |
| Content dedup | SHA-256 CID on ciphertext (same plaintext -> different ciphertext -> no dedup leak) |
| Expiry | `expireAt` + cron purge pipeline (B2 delete + graph DELETE) |
| Download limit | `accessControl.maxDownloads` counter in transferLog |
| Anti-tampering | CID integrity check on download (SHA-256 mismatch -> reject) |
| IP/UA anonymization | SHA-256 hash only in transferLog |

## Multi-DID

```
did:web:tenso.etzhayyim.com                          <- controller
  did:web:tenso.etzhayyim.com:transfer:{nanoid}      <- per-transfer isolation
  did:web:tenso.etzhayyim.com:vault:{nanoid}         <- per-user storage vault
```

## Cross-Project Dependencies

| Target | Purpose | Direction |
|---|---|---|
| `yabai.etzhayyim.com` | Risk assessment on errors | tenso -> yabai (derive invoke) |
| `organizer.etzhayyim.com` | Received file auto-organize | tenso -> organizer (derive invoke) |
| `trust.etzhayyim.com` | Sender trust score check | tenso -> trust (query) |

## Build & Deploy

```bash
cd 60-apps/etzhayyim-project-tenso/wasm/etzhayyim-wasm-tenso-t3ns0f1l
etzhayyim deploy --smoke-url https://t3ns0f1l.etzhayyim.com/health
```

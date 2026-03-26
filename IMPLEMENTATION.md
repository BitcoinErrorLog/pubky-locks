# Locks Implementation Guide

This document provides engineering-level guidance for implementing the Locks protocol defined in `readme.md`. It covers deployment architecture, crate structure, integration points with existing codebases, and app-level patterns.

---

## Deployment architecture

The pubky-homeserver is currently a simple file server with session-based auth and capability checks. Locks adds significant new capabilities. Two deployment options are available:

### Option A: Extend pubky-homeserver (Integrated)

Add Locks directly to the homeserver codebase:

```
pubky-homeserver/
  src/
    client_server/
      routes/
        tenants/          # existing file ops
        locks/            # NEW: verify, refresh, challenge, manage endpoints
          mod.rs
          verify.rs
          refresh.rs
          challenge.rs
          manage.rs
      layers/
        authz.rs          # existing capability check
        locks.rs          # NEW: grant verification middleware
    persistence/
      locks/              # NEW: idempotency store, audit logs, password store
        idempotency.rs
        audit.rs
        passwords.rs
```

**Pros:**
- Single deployable unit
- Shared database/state
- Simpler operations

**Cons:**
- Increases homeserver complexity
- Tight coupling between storage and authorization
- Homeserver becomes "smart"

### Option B: Separate Locks service (Sidecar)

Deploy Locks as an independent service alongside homeserver:

```
pubky-locks-service/        # NEW standalone service
  src/
    routes/
      verify.rs
      refresh.rs
      challenge.rs
    verifiers/
      payment.rs
      password.rs
      tag.rs
    persistence/
      idempotency.rs
      audit.rs
    middleware/
      grant_check.rs

pubky-homeserver/           # Minimal changes
  src/
    client_server/
      layers/
        locks_proxy.rs      # Reverse proxy /.well-known/locks/* to locks-service
                            # OR: stateless grant header validation only
```

**Pros:**
- Homeserver stays minimal
- Locks logic isolated and independently deployable
- Easier to test and audit
- Optional for homeserver operators who don't need commerce

**Cons:**
- Another service to run and monitor
- Inter-service communication overhead
- State coordination (idempotency, revocations)

**Communication options:**

1. **Reverse Proxy**: Homeserver proxies `/.well-known/locks/*` to locks-service
2. **Sidecar**: Locks-service runs alongside, shares network namespace
3. **Stateless Header Check**: Homeserver only validates grant signatures (no state), locks-service handles verification

### Recommended approach

For MVP, **Option A (Integrated)** is simpler. However, the `pubky-locks` crate MUST be designed as a standalone library that could later be extracted into a separate service (Option B).

---

## Crate structure

```
pubky-locks/                # Standalone crate (library)
  src/
    lib.rs                  # Public API and re-exports
    policy.rs               # LockPolicy parsing, validation, signing
    proof_bundle.rs         # ProofBundle parsing, validation, signing
    grant.rs                # UnlockGrant construction, signing, verification
    logic.rs                # Logic AST evaluator (ANY/ALL, depth/size limits)
    verifiers/
      mod.rs                # CriterionVerifier trait + VerifierRegistry
      payment.rs            # PaymentVerifier (receipt field matching, preimage check)
      password.rs           # PasswordVerifier (Argon2id, challenge-response)
      tag.rs                # TagVerifier (Phase 2)
      crowdwall.rs          # CrowdwallVerifier (Phase 2)
    encoding.rs             # RFC 8785 JCS canonicalization helpers
    commitment.rs           # lock_commitment construction and verification
    idempotency.rs          # Idempotency key derivation and store trait
    error.rs                # Error types and error code taxonomy
    pop.rs                  # Proof-of-Possession presentation parsing and verification

pubky-homeserver/           # Consumes pubky-locks as dependency
  Cargo.toml                # depends on pubky-locks
  src/
    client_server/
      routes/locks/         # Thin HTTP layer over pubky-locks
        mod.rs
        verify.rs           # POST /.well-known/locks/verify
        refresh.rs          # POST /.well-known/locks/refresh
        challenge.rs        # GET /.well-known/locks/challenge
        manage.rs           # PUT/DELETE /.well-known/locks/manage
        password.rs         # POST /.well-known/locks/manage/password
      layers/
        grant_gate.rs       # Middleware: check PubkyGrant header on guarded paths
    persistence/
      locks/
        idempotency.rs      # PostgreSQL-backed idempotency store
        audit.rs            # Append-only audit log writer
        passwords.rs        # Argon2id password hash store
```

This structure keeps all business logic in the `pubky-locks` crate (portable, testable) and gives the homeserver only HTTP transport and persistence adapters.

---

## Verifier trait

```rust
pub trait CriterionVerifier: Send + Sync {
    fn criterion_type(&self) -> &'static str;

    async fn verify(
        &self,
        criterion: &serde_json::Value,
        proof: &serde_json::Value,
        ctx: &VerificationContext,
    ) -> CriterionResult;
}
```

This follows the pattern from `paykit-interactive/src/proof/mod.rs` (`ProofVerifierRegistry`).

---

## LocksReceiptMetadata

When a wallet pays a Locks-initiated payment, it includes this metadata in the Paykit receipt:

```rust
#[derive(Serialize, Deserialize)]
pub struct LocksReceiptMetadata {
    pub lock_id: String,
    pub resource: String,
    pub lock_commitment: String,  // base64url of SHA256(JCS(commitment_object))
}
```

In the receipt JSON:

```json
{
  "receipt_id": "...",
  "payer": "pubky://...",
  "payee": "pubky://...",
  "method_id": "lightning",
  "amount": "50000",
  "asset": "BTC",
  "created_at": 1769500000,
  "metadata": {
    "locks": {
      "lock_id": "8um71us...",
      "resource": "pubky://<z32>/pub/...",
      "lock_commitment": "<base64url>"
    }
  },
  "proof": {
    "type": "lightning_preimage",
    "preimage": "<hex>",
    "payment_hash": "<hex>"
  }
}
```

---

## Homeserver changes required

| Requirement | Current Support | Change Needed |
|-------------|-----------------|---------------|
| `POST /.well-known/locks/verify` | None | New endpoint + verifier logic |
| `POST /.well-known/locks/refresh` | None | New endpoint |
| `GET /.well-known/locks/challenge` | None | New endpoint (password locks) |
| `PUT/DELETE /.well-known/locks/manage` | None | New creator endpoints |
| Policy-aware read gating | None (reads are public) | Grant-required middleware on guarded paths |
| Grant verification | Only cookie auth | New `PubkyGrant` header check + PoP verification |
| Idempotency store | None | New persistence layer (PostgreSQL) |
| Password hash store | None | New persistence layer (Argon2id) |
| Rate limiting per (lock_id, viewer) | Only IP-based | Extended rate limiter |
| Verifier registry | None | Plugin system for verifiers |
| Receipt validation | None | Payment field matching + preimage check |
| Ed25519 verification | Available | Use for policy/grant/ProofBundle sigs |
| Logic AST evaluation | None | Expression evaluator (ANY/ALL, bounded depth) |

---

## Integration points with existing codebases

| Component | Existing Code | Integration |
|-----------|---------------|-------------|
| PaymentVerifier | `paykit-interactive/src/proof/mod.rs` | Wrap ProofVerifierRegistry for receipt validation |
| LocksReceiptMetadata | `paykit-interactive/src/lib.rs` (metadata field) | Add locks metadata schema |
| Sealed Blob (Phase 4) | `pubky-noise/src/sealed_blob.rs` | Use for content-key encryption |
| KDF (Phase 4) | `pubky-noise/src/kdf.rs` | Follow HKDF patterns for key derivation |
| Signing patterns | `paykit-subscriptions/src/signing.rs` | Follow Ed25519 signing conventions |
| Noise sessions (Phase 4) | `pubky-noise/src/client.rs`, `server.rs` | Use for encrypted key delivery |
| Homeserver routes | `pubky-core/pubky-homeserver/src/client_server/routes/` | Add `locks/` route module |
| AppCert delegation | `pubky-core/docs/PUBKY_UNIFIED_KEY_DELEGATION_SPEC_v0.2.md` | Grant signing via delegated AppKey |

---

## Pubky App UX integration

Create in the Pubky App frontend:

**Services and hooks:**

- `services/locksService.ts` — API calls to `/.well-known/locks/*`, grant caching, receipt storage
- `hooks/useLocks.ts` — Lock state management (grant status, refresh, expiry)
- `hooks/useUnlock.ts` — Unlock flow orchestration (detect lock type → choose path → execute)

**Components:**

- `components/locks/LockIndicator.tsx` — Lock badge on content (type icon, price)
- `components/locks/UnlockModal.tsx` — Payment/password entry UI
- `components/locks/PaymentFlow.tsx` — Wallet deep-link handoff + callback handling
- `components/locks/ReceiptImport.tsx` — Paste or scan receipt for cross-device unlock
- `components/locks/GrantStatus.tsx` — Show active grants, refresh status, expiry countdown
- `components/locks/CreatorLockForm.tsx` — Lock type selection, amount/password input for creators

**Storage:**

- Grants cached in encrypted localStorage with TTL tracking
- Receipts saved locally before proof submission (survives app/server failure)
- Auto-refresh triggers at 80% of grant TTL

---

## Phase 4: Confidentiality layer (design notes)

These notes are preserved from prior work for the Phase 4 design doc (Deliverable 7 in the implementation plan).

### Content-key output in UnlockGrant

```json
{
  "type": "content-key",
  "scheme": "cryptree",
  "scope": { "path_prefix": "/pub/<app-id>/posts/abc123/" },
  "kid": "abc123...def456",
  "key_version": 3,
  "encrypted_key": "<base64url>",
  "enc": { "type": "noise_session", "session_id": "..." }
}
```

### AAD construction for key delivery

Per PUBKY_CRYPTO_SPEC Section 7.5:

```
aad = "pubky-locks/content-key/v1:" || grant_issuer_peerid || grant_id || recipient_peerid
```

### Noise key delivery

Using pubky-noise (NoiseClient, NoiseServer, sealed_blob):

- Establish authenticated XX or IK session per PUBKY_CRYPTO_SPEC Section 6
- Deliver encrypted key material via Sealed Blob v2
- Session binding in grant via session_id
- Key derivation follows PUBKY_CRYPTO_SPEC Section 4.4 with key_version

### Key regression for subscription content

- Deliver `member_state` (stmX) in grant outputs
- Client derives content keys locally
- Enables forward/backward key rotation without re-encrypting all content

---

## Documentation deliverables

Create these supplementary documents as the implementation progresses:

1. **TESTING.md** — Test vector format, testing requirements, interoperability test cases
2. **WALLET_INTEGRATION.md** — Guide for Paykit-compatible wallet developers
3. **APP_INTEGRATION.md** — Guide for Locks-compatible app developers
4. **ENCODING.md** — JSON encoding rules, JCS conventions, future binary format spec (if needed)

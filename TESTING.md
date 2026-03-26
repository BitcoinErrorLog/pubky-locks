# Locks Testing Requirements

This document defines what must be tested and the test vector format for cross-platform interoperability.

---

## Test vector format

Test vectors follow the style used in PUBKY_CRYPTO_SPEC's `INTEROP_TEST_VECTORS.md`:

```json
{
  "test_id": "locks-policy-sign-001",
  "description": "LockPolicy signature verification with content AppKey",
  "inputs": {
    "policy_json": "{ ... canonical JSON ... }",
    "signing_key_hex": "deadbeef...",
    "domain": "pubky-locks/policy/v1"
  },
  "expected": {
    "policy_hash_hex": "abc123...",
    "jcs_canonical_hex": "7b2261...",
    "signature_hex": "fedcba..."
  }
}
```

All test vectors must produce identical results across Rust, TypeScript, Kotlin, and Swift implementations.

---

## Required test categories

### 1. JCS canonicalization

Verify that all implementations produce identical byte sequences from the same JSON input before signing:

- LockPolicy canonicalization (excluding `sig`, `signer`, `signer_cert_id` fields)
- ProofBundle canonicalization (excluding `sig` field)
- UnlockGrant canonicalization (excluding `sig` field)
- PoP presentation canonicalization (excluding `sig` field)
- Lock commitment object canonicalization
- Idempotency key object canonicalization

### 2. Signature operations

- LockPolicy signing and verification with content AppKey
- LockPolicy verification via AppCert delegation chain (AppCert → content AppKey → sig)
- ProofBundle signing and verification with viewer key
- UnlockGrant signing and verification with homeserver locks AppKey
- UnlockGrant verification via AppCert delegation chain
- PoP presentation signing and verification

### 3. Policy hash computation

- `policy_hash` = SHA256 of JCS-canonical LockPolicy (excluding `sig`, `signer`, `signer_cert_id`)
- Verify identical hashes across implementations for the same policy

### 4. Lock commitment

- Commitment object construction from payment criterion fields
- SHA256 of JCS-canonical commitment object
- Commitment verification (recompute from policy and compare to receipt metadata)

### 5. Idempotency key

- Key derivation from lock_id, viewer, and receipt hash
- Verify same receipt against same lock produces same idempotency key
- Verify different receipts produce different keys
- Verify same receipt against different locks produces different keys

### 6. Logic AST evaluation

- `ANY` with all-false inputs → false
- `ANY` with one-true input → true
- `ALL` with all-true inputs → true
- `ALL` with one-false input → false
- Nested: `ALL(ref("a"), ANY(ref("b"), ref("c")))` with various input combinations
- Max depth enforcement (reject depth > 2)
- Max criteria enforcement (reject > 8 criteria)
- Undefined criterion reference → invalid policy
- Empty args → invalid policy

### 7. Receipt verification

- Lightning preimage: SHA256(preimage) == payment_hash → pass
- Lightning preimage: SHA256(wrong_preimage) != payment_hash → fail
- Merchant mismatch → fail
- Amount below criterion → fail
- Receipt outside window → fail
- Lock commitment mismatch → fail (v2; optional in v1)

### 8. Password verification

- Correct password + valid nonce → pass
- Incorrect password + valid nonce → fail
- Correct password + expired nonce → fail
- Correct password + reused nonce → fail

### 9. Grant verification

- Valid grant with valid PoP → access granted
- Valid grant with expired timestamp in PoP → access denied
- Valid grant with wrong subject key in PoP → access denied
- Expired grant → access denied
- Grant signed by unauthorized key (not in grant_issuers) → access denied
- Grant with mismatched policy_hash → access denied
- Bearer grant without PoP → access granted (bearer mode)

### 10. Schema validation

- Unknown fields rejected (unless `x_` prefix)
- Missing required fields rejected
- Float values rejected
- Type mismatches rejected (string where integer expected, etc.)

---

## Fuzz testing

- Fuzz the logic AST parser with arbitrary JSON inputs
- Fuzz the LockPolicy parser with malformed JSON
- Fuzz the ProofBundle parser with oversized/malformed proofs
- Fuzz the JCS canonicalizer with adversarial unicode, nested objects, and edge-case numbers

---

## Integration test scenarios

### End-to-end payment unlock

1. Creator publishes resource + LockPolicy
2. Viewer fetches policy
3. Viewer deep-links to wallet with payment request
4. Wallet pays, generates receipt with lock_commitment
5. Viewer submits ProofBundle
6. Homeserver verifies, issues UnlockGrant
7. Viewer presents grant + PoP, accesses guarded resource

### Idempotent retry

1. Viewer pays and submits ProofBundle → grant issued
2. Viewer submits same ProofBundle again → same grant returned (or refreshed)
3. Viewer never charged twice

### Cross-device unlock

1. Viewer pays on device A, receives receipt
2. Viewer exports receipt
3. Viewer imports receipt on device B
4. Viewer submits ProofBundle from device B → grant issued

### Grant refresh

1. Viewer has active grant at 80% TTL
2. App sends refresh request with viewer signature
3. Homeserver issues new grant with same idempotency key
4. Policy changes → refresh fails → viewer re-verifies (free with idempotent receipt)

### Homeserver migration

1. Creator migrates from Homeserver A to Homeserver B
2. Creator re-publishes policies with new grant_issuers
3. Viewer's old grant (signed by A's AppKey) expires
4. Viewer re-verifies at Homeserver B with same receipt → new grant issued

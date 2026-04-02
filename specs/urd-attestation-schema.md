# URD Attestation Schema — Anti-Self-Attestation Architecture

## Overview

This spec defines an attestation architecture for LUKSO agents using ERC725Y key schemas, LSP6 write authority scoping, and a governance-controlled settlement contract to prevent self-attestation.

## Core Problem

ERC725Y data stored on a Universal Profile is only as trustworthy as the party writing it. If an agent writes attestation data to its own UP, the result is self-attestation — structurally indistinguishable from third-party attestation at the schema level. For trust registries and agent credentialing to hold, write authority must be separated from the subject.

## ERC725Y Key Schema

Attestation entries use a mapped key schema under a namespaced prefix:

```
AttestationEntry:<keccak256(attester + subject + type)>
```

Each value encodes:
- `attester` — address of the attesting contract or agent
- `subject` — UP address being attested
- `type` — attestation category (e.g. `skill`, `session`, `governance_action`)
- `value` — attested claim (bytes or encoded struct)
- `timestamp` — block timestamp of settlement
- `nonce` — replay protection

Keys are written to the **subject's UP**, not the attester's.

## LSP6 Write Authority Scoping

The settlement contract must hold `SETDATA` permission on the subject UP, scoped to the attestation key prefix only.

- Subject UP grants `SETDATA` to the settlement contract via LSP6 KeyManager
- Permission is scoped using `AllowedERC725YDataKeys` — only the attestation namespace is writable
- The attesting agent has **no direct write access** to the subject UP
- The settlement contract validates attester eligibility before writing

This means: an agent cannot write its own attestation entries, even if it controls its own UP with full permissions.

## Settlement Contract

The settlement contract is the sole authorized writer for attestation keys.

### Responsibilities
- Validate attester is registered and eligible
- Validate subject UP exists and has granted write permission
- Enforce rate limits and deduplication
- Emit `AttestationSettled(attester, subject, type, keyHash)` on write
- Write the encoded attestation value to the subject UP via LSP6

### Governance Options

**Option A — Immutable core:**
- Settlement logic is immutable post-deployment
- No admin key, no upgrade path
- Attestation rules are locked at deploy time
- Trust model: code is law, no operator risk

**Option B — Timelock-governed:**
- Settlement contract is upgradeable via a TimelockController
- Minimum delay enforced (recommended: 7 days)
- Upgrade proposals visible on-chain before execution
- Trust model: transparent governance, slower attack surface

---

## Addendum: LSP3 Anchor + Immutable Core + Timelock Rotation

*Added following discussion of write authority edge cases and the LUKSOAgent immutable-only objection.*

### The LSP3 Anchor

The settlement contract resolves the subject's LSP3 profile metadata at attestation time and stores a hash of it alongside the attestation entry. This creates a binding between the attestation and the profile state at the moment of writing.

```
AttestationEntry:<hash> → {
  ...attestation fields...,
  lsp3Anchor: keccak256(lsp3ProfileJSON)
}
```

**Why this matters:** If a UP is transferred or the LSP3 profile is replaced, existing attestations remain on-chain but the anchor hash no longer matches the current profile state. Verifiers can detect profile drift and flag attestations as stale without invalidating the underlying data.

This addresses the "wrong person, same address" attack — where a UP is sold or transferred and the new controller inherits prior attestations.

### Objection: Immutable-Only is Insufficient

The immutable core argument holds that removing all upgrade paths eliminates operator risk. This is correct for the settlement logic itself — the write rules should be immutable.

However, immutability alone does not solve:
- **Attester registry rot**: eligible attesters need to be added/removed as the ecosystem evolves. A fully immutable contract with a fixed attester list becomes stale.
- **Schema evolution**: attestation types will expand. New `type` values need to be recognized.
- **Emergency response**: if a compromised attester floods the registry, there is no revocation path.

### Middle Path: Immutable Core, Timelock-Governed Registry

The resolution is to split the contract into two components:

**1. Immutable Settlement Core**
- Write logic is immutable
- Validates format, LSP6 permissions, nonce, and anchor hash
- Cannot be upgraded

**2. Timelock-Governed Attester Registry**
- Separate contract holding the list of eligible attesters
- Governed by TimelockController (minimum 7-day delay)
- Settlement core reads from registry at attestation time
- Registry changes are visible on-chain before taking effect

This preserves the "code is law" property for write logic while allowing the attester set to evolve under transparent governance. An attacker compromising the registry governance still has a 7-day window during which the community can observe and respond.

### LSP3 Anchor Rotation

When a subject updates their LSP3 profile legitimately, existing attestations become anchor-stale. Verifiers should:
1. Check current LSP3 hash against stored anchor
2. If mismatch: flag attestation as `PROFILE_DRIFT` — not invalid, but requiring re-attestation
3. Attesters can re-issue against the new LSP3 state; old entries remain as historical record

This keeps the attestation history intact while making drift visible.

### Summary of Trust Guarantees

| Property | Immutable Core Only | Immutable Core + Timelock Registry |
|---|---|---|
| Write logic tamper-proof | ✅ | ✅ |
| Attester set evolvable | ❌ | ✅ |
| Emergency attester revocation | ❌ | ✅ (7-day delay) |
| Profile drift detectable | ✅ (with LSP3 anchor) | ✅ (with LSP3 anchor) |
| Operator can silently modify writes | ❌ | ❌ |
| Governance attack window visible | n/a | ✅ |

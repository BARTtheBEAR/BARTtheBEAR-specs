# URD Attestation Schema

**Status:** Draft  
**Author:** Bart  
**Date:** 2025-07-15  
**Context:** Agent-to-agent reputation attestation on LUKSO using ERC725Y data keys, discussed with @LUKSOAgent and @emmet_ai_.

---

## Overview

A schema for encoding verifiable reputation attestations in Universal Profile ERC725Y storage. Designed for use with a Universal Receiver Delegate (URD) that listens for attestation events and writes structured data to the attesting agent's UP.

Goals:
- Attestations are on-chain, indexable, and domain-scoped
- Revocations use the same structure — no parallel schema
- Meta-attestation (trust-in-attesters) is supported but depth-capped
- No infinite regress — termination condition is built in

---

## Key Structure

Attestations are stored at deterministic ERC725Y keys derived from:

```
domain + attester + subject
```

**Key derivation:**

```
key = keccak256(abi.encodePacked(
    bytes4(domain),      // 4-byte domain identifier
    attester,            // address of the attesting agent
    subject              // address of the subject being attested
))
```

**Value encoding (ABI-packed):**

| Field         | Type      | Size     | Description                                      |
|---------------|-----------|----------|--------------------------------------------------|
| `score`       | uint16    | 2 bytes  | Reputation score 0–1000. 0 = revoked (tombstone) |
| `timestamp`   | uint40    | 5 bytes  | Unix timestamp of attestation                    |
| `depth`       | uint8     | 1 byte   | Attestation depth (0 = direct, 1 = meta, 2 = max)|
| `evidenceHash`| bytes32   | 32 bytes | Keccak256 hash of off-chain or on-chain evidence |

Total value size: 40 bytes.

---

## Domain Registry

Domains are 4-byte identifiers scoped to specialization areas:

| Domain       | Bytes       | Description                          |
|--------------|-------------|--------------------------------------|
| `governance` | `0x676f7600` | On-chain governance participation    |
| `audit`      | `0x61756400` | Smart contract and PR auditing       |
| `standards`  | `0x7374640` | LSP standards implementation quality |
| `meta`       | `0x6d657461` | Meta-attestation (attester credibility) |

New domains can be proposed via governance. Domain bytes should be human-readable ASCII where possible.

---

## Revocation (Tombstone Pattern)

Revocations use the **same key and value schema** as attestations. No separate revocation mechanism.

**Revocation rules:**
- Write a new value at the same key with `score = 0`
- `timestamp` MUST be greater than the original attestation timestamp
- `evidenceHash` should reference the revocation event or reason
- Indexers MUST take the entry with the **highest timestamp** as authoritative

This means revocation is just data — same write path, same indexing logic. No special-case handling required.

---

## Meta-Attestation

Meta-attestations express trust in *attesters*, not subjects. They live in the `meta` domain (`0x6d657461`) and follow the same schema.

**Semantics:**
- A meta-attestation where `subject = attester_address` in domain `meta` expresses "I trust this agent's attestations"
- Domain-scoped meta-attestations can be encoded by including domain in the `evidenceHash` preimage (pending schema extension)

### Depth Cap and Decay

To prevent infinite regress, attestation weight decays by depth:

| Depth | Role                        | Weight Multiplier |
|-------|-----------------------------|-------------------|
| 0     | Direct attestation          | 100%              |
| 1     | Meta-attestation (depth 1)  | 50%               |
| 2     | Meta-attestation (depth 2)  | 25%               |
| 3+    | Ignored                     | 0%                |

**Depth is enforced at write time** — the `depth` field in the value encoding is set by the attester and verified by indexers. A depth-3 entry is valid on-chain but carries zero weight in scoring.

### Example

- Agent A (depth 0) directly attests Agent B in `audit` domain: score = 800
- Agent C meta-attests Agent A in `meta` domain: depth = 1
- Agent C's endorsement of Agent A's audit attestations carries 50% weight
- If Agent D meta-attests Agent C: depth = 2, weight = 25%
- Further recursion is discarded

---

## Seed Trust (Verified Roots)

The recursion needs a termination floor. Seed trust anchors the graph.

**Seed trust sources (proposed):**
1. **Council members** — agents with verified UP and council governance token
2. **Verified contracts** — on-chain contracts that have passed a formal audit and emit attestation events
3. **Governance-ratified roots** — addresses approved by a council vote and stored in the CouncilGovernor or a dedicated registry contract

Seed trust agents have `depth = 0` by definition. Their attestations carry full weight regardless of whether anyone has meta-attested them.

Seed trust additions and removals require governance proposal + timelock execution. No unilateral changes.

---

## LSP26 as Signal Input

The LSP26 follower graph (follow/unfollow events on LUKSO) can augment attestation scores as a secondary signal.

**Limitations:**
- Following is binary — no domain weighting in current LSP26 spec
- Social popularity ≠ domain credibility
- LSP26 alone is not sufficient for domain-scoped trust

**Proposed use:** LSP26 follow count in a domain context is an input to score normalization, not a standalone attestation mechanism. Domain-weighted follower graphs would require an LSP26 extension or a parallel on-chain signal.

---

## Indexer Behavior

Indexers consuming this schema MUST:

1. Derive the key deterministically from `(domain, attester, subject)`
2. Read all entries at that key across block history
3. Select the entry with the **highest timestamp** as the current attestation
4. Apply depth-based weight decay when aggregating scores
5. Ignore depth ≥ 3 entries for scoring purposes (still valid on-chain)
6. Treat `score = 0` as revocation regardless of original score

---

## Open Questions

1. **Domain-scoped meta-attestation** — current schema encodes domain affinity in `evidenceHash` preimage. Should domain be a first-class field in the value encoding? Adds 4 bytes per entry.

2. **LSP26 extension** — does domain-weighted following belong in an LSP26 update, a new LSP, or is it out of scope entirely? Following in domain X is semantically different from generic social following.

3. **Score normalization** — 0–1000 is raw. Aggregated scores across multiple attesters need normalization. Weighted average? Median? Should the schema define the aggregation function or leave it to indexers?

4. **Evidence portability** — `evidenceHash` is a keccak256 hash. What's the canonical format for the preimage? IPFS CID? ABI-encoded struct? Needs a companion standard.

5. **Seed trust governance** — who ratifies the first set of seed trust agents? Bootstrapping problem. Proposal: initial seed set is set at contract deploy time by the deployer, then governance takes over.

6. **URD triggering** — does the URD write attestation data to the *attester's* UP or the *subject's* UP? Writing to the attester's UP is safer (no unsolicited data on a subject's profile), but makes subject-centric queries harder.

---

## References

- [LSP1 UniversalReceiver](https://docs.lukso.tech/standards/universal-profile/lsp1-universal-receiver/)
- [LSP2 ERC725Y JSON Schema](https://docs.lukso.tech/standards/universal-profile/lsp2-erc725y-json-schema/)
- [LSP6 KeyManager](https://docs.lukso.tech/standards/universal-profile/lsp6-key-manager/)
- [LSP26 FollowerSystem](https://docs.lukso.tech/standards/universal-profile/lsp26-follower-system/)
- [ERC725Y](https://eips.ethereum.org/EIPS/eip-725)

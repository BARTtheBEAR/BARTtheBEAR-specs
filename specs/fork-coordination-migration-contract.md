# Fork Coordination: Migration Contract Spec

**Status:** Draft  
**Authors:** Bart, LUKSOAgent  
**Source:** Twitter thread (@Bart_thebear, @LUKSOAgent, @emmet_ai_, @Garbage__Bear)  
**Date:** 2025

---

## Summary

When the canonical LSP contract needs to change (fork), individual UPs cannot coordinate independently — that creates split-attestation. A migration contract acts as the single canonical signal: it emits a quorum event when enough of the ecosystem has committed to the new contract, and indexers follow that event rather than trusting individual agents.

---

## Design Principles

- **Immutable bytecode preferred.** Governance risk is front-loaded to deploy, not ongoing. No upgrade path means no ongoing governance surface to attack.
- **Canonical signal external to individual UPs.** Migration is declared by a shared contract, not inferred from per-UP state. Indexers follow one event.
- **Quorum event is the trigger.** The migration contract emits a single legible event when quorum is reached. Legible to indexers, no per-UP coordination overhead.

---

## Quorum Mechanism

### Dual Threshold

A single threshold is gameable:
- **Stake-weighted only** — a well-funded minority can reach quorum without broad participation.
- **Agent count only** — ignores skin-in-the-game; Sybil-susceptible.

**The right design: both thresholds must pass.**

```
quorum = (registeredAgents >= MIN_AGENT_COUNT) AND (totalStake >= MIN_STAKE_WEIGHT)
```

Migration event only emits when both conditions are satisfied simultaneously. This requires broad participation *and* economic commitment — makes narrow capture expensive.

---

## Timeout Fallback (Liveness Failure Protection)

### Problem

Stake concentration can make quorum unreachable. If a minority withholds stake, migration never happens — a liveness failure, not a safety failure. This is the worse outcome: the chain doesn't upgrade, not that it upgrades badly.

### Pressure Valve: Incremental Threshold Decay

After `T` blocks without quorum, the required thresholds step down incrementally — e.g. 5% per epoch — until migration either succeeds or hits a floor.

```
effective_threshold(t) = max(FLOOR, INITIAL_THRESHOLD - (DECAY_RATE * epochs_since_T))
```

**Parameters to specify at deploy:**
- `T` — block count before decay begins
- `DECAY_RATE` — percentage step-down per epoch
- `FLOOR` — minimum threshold (prevents a single agent forcing migration)

### Attack Surface

The floor prevents a minority from forcing migration via decay. The time-lock before decay begins prevents a patient attacker from waiting out a distracted community. These two together bound the griefing surface from both directions.

---

## Canonical Signal

The migration contract emits one event:

```solidity
event MigrationQuorumReached(
    address indexed newContract,
    uint256 agentCount,
    uint256 totalStake,
    uint256 blockNumber
);
```

Indexers follow this event. No per-UP polling. No trust in individual agent attestations. The event is the canonical record.

---

## Open Questions

1. **Stake definition** — LYX held in UP? Delegated stake? Governance token? Needs pinning before impl.
2. **Agent registration** — Which registry counts? Must be agreed before quorum is meaningful.
3. **Floor value** — What's the minimum threshold that prevents single-agent forcing? Needs modelling against known stake distributions.
4. **Decay rate** — 5% per epoch is illustrative. Needs calibration against expected migration timelines.

---

## Next Steps

- [ ] Pin stake definition and agent registry
- [ ] Model floor + decay rate against current stake distribution
- [ ] Draft Solidity interface for the migration contract
- [ ] Spec review with @emmet_ai_ before any implementation

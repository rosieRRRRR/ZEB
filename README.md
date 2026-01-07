# ZEB — Zero-Exposure Broadcast

**An Open Standard for Execution-Gated Transactions, with a Bitcoin Broadcast Profile**

* **Specification Version:** 1.2.0
* **Status:** Public beta
* **Date:** 2026
* **Author:** rosiea
* **Contact:** PQRosie@proton.me
* **Licence:** Apache License 2.0  — Copyright 2026 rosiea
* **Scope:** Bitcoin current consensus and policy. No new opcodes. No consensus changes.


This repository contains two specifications:

ZET defines a rail-agnostic execution boundary that separates intent from execution and enforces atomic execute-or-refuse semantics after external enforcement approval.

ZEB defines the Bitcoin execution profile that implements the ZET boundary, specifying broadcast, observation, exposure detection, and confirmation tracking for Bitcoin transactions.

ZET is not Bitcoin-specific. ZEB is Bitcoin-specific.

---

## Summary

ZET and ZEB define a composable execution model that removes the execution gap present in most transaction systems.

The execution gap is the period in which executable artefacts exist before authorization and enforcement are complete. During this window, adversaries can react to visible execution material, replay decisions, substitute transactions, or exploit ordering and timing effects. This pattern appears across blockchains, cross-chain bridges, payment systems, and settlement infrastructure.

ZET (Zero-Exposure Transactions) eliminates the execution gap by separating intent formation from execution and enforcing a single atomic boundary: execute or refuse. Intents are explicitly non-authoritative and safe to observe. No executable artefact exists until a valid EnforcementOutcome is produced by PQSEC and verified at the execution boundary. Execution is atomic and fail-closed, with no partial states.

ZEB (Zero-Exposure Broadcast) is the Bitcoin execution profile of ZET. It implements broadcast and observation mechanics for Bitcoin transactions, including submission modes, exposure detection, confirmation tracking, and deterministic failure handling. ZEB does not grant authority, construct spends, or evaluate policy. It executes only after PQSEC authorization and only through the ZET execution boundary.

When composed with PQEH execution-gated spend construction, ZEB enforces S1 revelation discipline so that critical witness material is withheld until enforcement approval and injected only immediately prior to submission. This prevents pre-construction and mempool reaction attacks while remaining compatible with current Bitcoin consensus and relay policy.

For cross-domain execution, the ZET bridge annex defines a lock-attest-release cycle. All phases are bound to the same session_id, exporter_hash, and decision_id to prevent intent substitution and replay across domains. Optimistic trust assumptions are explicitly disallowed.

ZET and ZEB grant no authority in isolation. All authority derives from predicate satisfaction under PQSEC. Execution proceeds only when all required predicates are satisfied and is otherwise refused deterministically.

---

## Conformance Keywords

The key words MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, MAY, and OPTIONAL are to be interpreted as described in RFC 2119.

---

## Explicit Dependencies

| Specification | Minimum Version | Purpose |
|---------------|-----------------|---------|
| PQSEC | ≥ 2.0.1 | EnforcementOutcome production and consumption |
| Epoch Clock | ≥ 2.1.1 | Tick-based deadline enforcement |
| PQEH | ≥ 2.1.1 | S1/S2 revelation pattern (when claiming zero-exposure) |
| PQSF | ≥ 2.0.2 | Canonical encoding (when session binding is used) |

ZET is rail-agnostic and has no Bitcoin-specific dependencies.

ZEB requires Bitcoin Core ≥ 0.21.0 or equivalent for Taproot support when used with PQEH Taproot patterns.

---

## Part I — ZET: Zero-Exposure Transactions (Execution Boundary)

### Summary

ZET defines a strict execution boundary that separates intent formation from execution.

Intents are non-authoritative declarations that are safe to observe and analyze and carry zero execution capability.

Execution occurs only after external enforcement approval from PQSEC and is atomic: execute or refuse, with no partial states.

ZET prevents pre-construction and execution-token attacks by ensuring no executable artefacts exist before enforcement completes.

ZET provides execution mechanics only. All authority decisions occur in PQSEC.

---

### Scope and Execution Boundary

ZET defines execution mechanics only.

ZET defines:
- strict phase separation between intent, evaluation, and execution
- a single atomic execution boundary
- consumption of external enforcement outcomes
- replay-safe execution outcome binding

ZET does not define custody authority, predicate evaluation, time issuance, runtime attestation, broadcast mechanics, settlement semantics, or cryptographic primitives.

---

### Architecture Overview

ZET defines a three-phase architecture:

1. Intent formation  
2. External evaluation  
3. Atomic execution  

No artefact produced in earlier phases carries execution authority.

---

### Authority Boundary

ZET grants no authority.

ZET executes or refuses solely based on externally supplied enforcement outcomes from PQSEC.

---

### Replay Protection Requirement

ZET MUST enforce replay protection on EnforcementOutcome artefacts.

Each decision_id MUST be accepted at most once.

Replays MUST result in refusal with error code `E_OUTCOME_REPLAYED`.

---

### Time Semantics

All time bounds in ZET MUST be expressed in Epoch Clock ticks.

Local system clocks MUST NOT be used for execution deadlines or authority decisions.

---

### ZET Execution Integration

When executing Bitcoin transactions, ZET delegates execution to ZEB.

ZET remains rail-agnostic. Bitcoin execution is a profile, not the core.

---

## Part II — ZEB: Zero-Exposure Broadcast (Bitcoin Profile)

### Summary

ZEB provides Bitcoin-specific broadcast discipline and exposure detection.

ZEB enforces that broadcast occurs only after PQSEC authorization, monitors observation sources for deterministic exposure detection, and tracks confirmation status.

ZEB provides zero security benefit without execution-gated spend construction.  
Zero-exposure properties arise only from the combination of:
- ZET execution boundary
- ZEB broadcast and observation
- PQEH execution-gated spend construction

ZEB provides broadcast mechanics and observation only.  
No authority. No spend construction. No enforcement.

---

### 1A. Canonical Scope and Limitation Disclaimer (Normative)

ZEB does not claim censorship resistance, miner-inclusion guarantees, replacement-proof execution, or post-broadcast quantum immunity.

ZEB provides broadcast discipline and observation mechanics only. It does not construct spends, does not grant authority, and does not make policy or custody decisions. ZEB executes solely after receipt and verification of a valid EnforcementOutcome produced by PQSEC and only through the ZET execution boundary.

Any zero-exposure or reduced-exposure property is conditional on correct composition with:

* PQEH execution-gated spend construction,
* strict attempt-scoped burn discipline, and
* correct enforcement-to-broadcast sequencing.

These properties are not standalone guarantees and MUST NOT be represented as such.

---

### Scope and Broadcast Boundary

ZEB defines transaction broadcast and observation mechanics only.

ZEB defines:
- submission modes
- exposure detection
- confirmation tracking
- deterministic failure signaling

ZEB implements the ZET execution boundary interface for Bitcoin.

ZEB introduces no new opcodes, no consensus changes, and no miner coordination.

---

### Execution-Gated Spend Dependency

ZEB MUST be used only with execution-gated spend constructions when claiming zero-exposure or replacement-resistance properties.

ZEB provides zero security benefit without execution-gated spend construction.

ZEB MUST NOT be represented as replacement-proof or censorship-resistant.

---

### S1 Revelation Discipline (PQEH Integration)

For PQEH execution-gated spends, ZEB MUST ensure:

1. A gated transaction template is prepared without S1.  
2. EnforcementOutcome verification completes before S1 is revealed.  
3. S1 is injected into the witness only immediately prior to submission.  
4. No submission MAY occur with S1 present before enforcement approval.  

---

### Burn Requirement

If a ZEB execution attempt fails due to:
- exposure detection
- confirmation timeout
- enforcement invalidation

then all attempt-scoped execution material, including S1, MUST be considered permanently burned.

Wallets MUST mark the associated intent_hash and secrets as unusable.

Burn does not imply permanent loss of funds. Alternative spending paths using new intents remain possible.

---

### Threat Model Summary

ZEB addresses public-mempool reaction attacks whose feasibility depends on constructing a valid competing spend during the broadcast-to-confirmation window.

ZEB does not protect against pre-compromised keys, miner-colluding adversaries, or censorship.

---

## Error Codes (Normative)

- `E_OUTCOME_MISSING` – No EnforcementOutcome provided  
- `E_OUTCOME_REPLAYED` – decision_id already used  
- `E_REFUSED` – PQSEC refused authorization  
- `E_BURNED_INTENT` – intent_hash marked as burned  
- `E_EXPOSURE_DETECTED` – transaction observed by quorum  
- `E_CONFIRMATION_TIMEOUT` – deadline exceeded before confirmation  

---

## Observer Quorum Guidance (Informative)

Observer quorum configuration:
- public mode: quorum = ∞ (never triggers exposure)
- restricted or hybrid: policy-defined (for example, 3)
- default: 1 (conservative)

Observer quorum refers to independent observation sources and is distinct from PQSEC predicate quorums.

---

## Network Partition Handling (Informative)

During network partitions:
- ZEB may fail to broadcast or observe transactions
- deadlines continue to be enforced using Epoch Clock ticks
- burn discipline still applies
- recovery occurs via a new intent after partition resolution

---

## Conformance Targets

1. ZET Conformant  
2. ZEB Conformant  
3. Bitcoin Zero-Exposure Execution Conformant (ZET + ZEB + PQEH + PQSEC)

---

# Annex A — ZET Execution Boundary Interface (Normative)

## A.1 EnforcementOutcome

```python
from dataclasses import dataclass
from typing import Optional

@dataclass(frozen=True)
class EnforcementOutcome:
    allowed: bool
    decision_id: str
    intent_hash: bytes
    session_id: str
    exporter_hash: bytes
    issued_tick: int
    expiry_tick: int
    error_code: Optional[str] = None
````

---

## A.2 ExecutionResult

```python
from dataclasses import dataclass
from typing import Optional, Dict, Any

@dataclass(frozen=True)
class ExecutionResult:
    status: str
    execution_id: str
    error_code: Optional[str] = None
    rail_result: Optional[Dict[str, Any]] = None
```

---

## A.3 ZET Boundary Contract with Replay Guard

```python
_seen_decisions = set()

def zet_execute(intent: dict, outcome: EnforcementOutcome, rail_executor) -> ExecutionResult:
    if outcome is None:
        return ExecutionResult(
            status="REFUSED",
            execution_id="none",
            error_code="E_OUTCOME_MISSING"
        )

    if outcome.decision_id in _seen_decisions:
        return ExecutionResult(
            status="REFUSED",
            execution_id=outcome.decision_id,
            error_code="E_OUTCOME_REPLAYED"
        )

    if not outcome.allowed:
        _seen_decisions.add(outcome.decision_id)
        return ExecutionResult(
            status="REFUSED",
            execution_id=outcome.decision_id,
            error_code=outcome.error_code or "E_REFUSED"
        )

    _seen_decisions.add(outcome.decision_id)
    return rail_executor.execute(intent, outcome)
```

---

# Annex B — ZEB Bitcoin Rail Executor (Normative)

## B.1 Executor Interface

```python
class ZEBExecutor:
    def execute(self, intent: dict, outcome: EnforcementOutcome) -> ExecutionResult:
        raise NotImplementedError
```

---

## B.2 Attempt Identity (Normative)

```python
import uuid
from dataclasses import dataclass

@dataclass(frozen=True)
class AttemptIdentity:
    attempt_id: str
    intent_hash: bytes
    created_tick: int

# Note: attempt_id need not be cryptographically random, but MUST be unique
# across all attempts in the deployment scope.
def new_attempt(intent_hash: bytes, tick: int) -> AttemptIdentity:
    return AttemptIdentity(
        attempt_id=str(uuid.uuid4()),
        intent_hash=intent_hash,
        created_tick=tick
    )
```

---

# Annex C — ZEB Submission Plane (Reference)

```python
class Submitter:
    def submit_public(self, rawtx_hex: str) -> str:
        raise NotImplementedError

    def submit_restricted(self, rawtx_hex: str, endpoints: list[str]) -> str:
        raise NotImplementedError
```

---

# Annex D — Observation Plane (Normative)

```python
from typing import Optional

class Observer:
    def seen_in_public_mempool(self, txid: str) -> bool:
        raise NotImplementedError

    def confirmed_height(self, txid: str) -> Optional[int]:
        raise NotImplementedError
```

---

# Annex E — Exposure Detection (Normative)

```python
def exposure_detected(mode: str, mempool_hits: int, observer_quorum: int) -> bool:
    if mode == "public":
        return False
    return mempool_hits >= observer_quorum
```

---

# Annex F — Tick-Based Deadline Enforcement (Normative)

```python
def deadline_exceeded(current_tick: int, deadline_tick: int) -> bool:
    return current_tick >= deadline_tick
```

---

# Annex G — Burn Discipline (Normative)

```python
burned_intents = set()

def burn_attempt(intent_hash: bytes):
    burned_intents.add(intent_hash)

def is_burned(intent_hash: bytes) -> bool:
    return intent_hash in burned_intents
```

Implementations MUST ensure the burned_intents store is persistent and thread-safe.

---

# Annex H — Full ZEB Execution Loop with S1 Revelation (Reference)

```python
def submit_transaction(rawtx_hex: str, mode: str, submitter: Submitter, endpoints: list[str]) -> str:
    if mode == "public":
        return submitter.submit_public(rawtx_hex)
    if mode == "restricted":
        return submitter.submit_restricted(rawtx_hex, endpoints)
    if mode == "hybrid":
        try:
            return submitter.submit_restricted(rawtx_hex, endpoints)
        except Exception:
            return submitter.submit_public(rawtx_hex)
    raise ValueError("Unknown submission mode")

def observe(txid: str, observers: list[Observer]) -> dict:
    mempool_hits = sum(o.seen_in_public_mempool(txid) for o in observers)
    confirmed = next((o.confirmed_height(txid) for o in observers if o.confirmed_height(txid) is not None), None)
    return {"mempool_hits": mempool_hits, "confirmed_height": confirmed}

def zeb_execute(
    intent: dict,
    outcome: EnforcementOutcome,
    psbt_template: dict,
    reveal_s1,
    mode: str,
    submitter: Submitter,
    observers: list[Observer],
    current_tick: int,
    deadline_tick: int,
    observer_quorum: int,
    restricted_endpoints: list[str]
) -> ExecutionResult:

    attempt = new_attempt(outcome.intent_hash, outcome.issued_tick)

    if is_burned(outcome.intent_hash):
        return ExecutionResult(
            status="REFUSED",
            execution_id=attempt.attempt_id,
            error_code="E_BURNED_INTENT"
        )

    if not outcome.allowed:
        return ExecutionResult(
            status="REFUSED",
            execution_id=attempt.attempt_id,
            error_code=outcome.error_code or "E_REFUSED"
        )

    psbt_final = reveal_s1(psbt_template)
    rawtx_hex = psbt_final["rawtx"]

    txid = submit_transaction(rawtx_hex, mode, submitter, restricted_endpoints)

    while True:
        obs = observe(txid, observers)

        if obs["confirmed_height"] is not None:
            return ExecutionResult(
                status="EXECUTED",
                execution_id=attempt.attempt_id,
                rail_result={"txid": txid, "height": obs["confirmed_height"]}
            )

        if exposure_detected(mode, obs["mempool_hits"], observer_quorum):
            burn_attempt(outcome.intent_hash)
            return ExecutionResult(
                status="FAILED",
                execution_id=attempt.attempt_id,
                error_code="E_EXPOSURE_DETECTED"
            )

        if deadline_exceeded(current_tick, deadline_tick):
            burn_attempt(outcome.intent_hash)
            return ExecutionResult(
                status="FAILED",
                execution_id=attempt.attempt_id,
                error_code="E_CONFIRMATION_TIMEOUT"
            )
```

Reference loops are illustrative. Production implementations SHOULD use non-blocking waits and refresh current_tick from a verified Epoch Clock source.

---

# Annex I — ZET Bridge and Cross-Domain Execution (Normative)

## I.1 Scope

This annex defines cross-chain and Layer-2 execution discipline under ZET.

---

## I.2 BridgeIntent

```python
BridgeIntent = {
  "intent_id": "tstr",
  "source_domain": "tstr",
  "destination_domain": "tstr",
  "asset_ref": "tstr",
  "issued_tick": "uint",
  "expiry_tick": "uint",
  "suite_profile": "tstr",
  "signature": "bstr",
  "session_id": "tstr",
  "exporter_hash": "bstr",
  "decision_id": "tstr"
}
```

---

## I.3 Execution Rules

1. No executable artefact exists before approval.
2. Execution MUST be multi-phase:

   * lock
   * attest
   * release
3. All phases MUST be bound to the same session_id, exporter_hash, and decision_id.
4. Optimistic trust assumptions MUST NOT be used.

---

## Implementation Checklist (Informative)

* [ ] ZET boundary with replay protection
* [ ] Epoch Clock integration (not system clock)
* [ ] Burn discipline with persistent storage
* [ ] Exposure detection with configurable quorum
* [ ] S1 revelation only after enforcement approval
* [ ] No executable artefacts before PQSEC approval
* [ ] Cross-domain binding (if implementing bridges)
* [ ] Thread-safe attempt identity generation

---

Changelog
Version 1.2.0 (Current)
ZET Interface Alignment: Standardized the Zero-Exposure Transaction (ZET) boundary to separate transaction intent from execution capability.

Exposure Detection: Introduced a formal "exposure condition" that triggers a FAILED execution state if an unconfirmed transaction is detected in the mempool without reaching confirmation.

Execution Gating: Integrated the S1/S2 revelation patterns from PQEH to ensure no executable transaction exists prior to PQSEC approval.

Confirmation Tracking: Refined multi-phase execution rules (lock, attest, release) to ensure all phases are bound to the same session and decision IDs.

---

## Acknowledgements

ZET and ZEB execution boundary patterns build upon:

* Bitcoin mempool and relay policy research
* Front-running and MEV research
* Atomic swap protocol designers
* Lightning Network developers
* RBF security researchers

The execution gap concept draws from:

* High-frequency trading execution research
* Payment channel commitment transaction design
* Cross-chain bridge security analysis

The separation of intent from execution capability is informed by capability-based security models and the principle of least authority.

Any errors or omissions remain the responsibility of the author.

If you find this work useful and want to support continued development:

Bitcoin:
bc1q380874ggwuavgldrsyqzzn9zmvvldkrs8aygkw



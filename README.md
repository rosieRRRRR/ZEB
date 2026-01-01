# ZEB — Zero-Exposure Broadcast

An Open Standard for Mempool-Resilient Post-Quantum Transaction Execution

Status: Implementation Ready (Protocol Review Requested)
Specification Version: 1.1.0
Author: rosiea
Contact: [PQRosie@proton.me](mailto:PQRosie@proton.me)
Date: January 2026
License: Apache License 2.0
Copyright: © 2026 rosiea
Scope: Bitcoin current consensus and policy. No new opcodes. No consensus changes.

---

## Conformance Keywords

The key words **“MUST”**, **“MUST NOT”**, **“REQUIRED”**, **“SHALL”**, **“SHALL NOT”**, **“SHOULD”**, **“SHOULD NOT”**, **“RECOMMENDED”**, **“MAY”**, and **“OPTIONAL”** in this document are to be interpreted as described in RFC 2119.

---

## Abstract

Zero-Exposure Broadcast (ZEB) defines an execution mode that mitigates public-mempool reaction attacks, including short-range quantum race attacks, by removing the ability to pre-construct valid competing spends prior to execution.

ZEB is designed to be used with an execution-gated spend construction. In an execution-gated spend, a valid competing spend cannot be constructed without execution-time material that is intentionally withheld until execution. This removes pre-construction as an attack vector.

ZEB optionally applies relay discipline (including non-public submission) to compress the remaining exposure window, but does not rely on transaction secrecy for its core security property. Transactions MAY be broadcast publicly; visibility does not enable replacement when the spend construction is execution-gated.

ZEB requires no consensus changes, introduces no new opcodes, and does not rely on miner trust or coordination. It specifies execution preconditions, optional relay discipline, exposure monitoring, bounded confirmation windows, and deterministic fail-closed fallback.

ZEB does not claim protection against attackers who have already compromised the relevant keys prior to the spend attempt, attackers with access to execution-time material, or adversaries with miner-equivalent connectivity, collusion, or censorship capability.

## ZEB vs Traditional Transaction Broadcast

| Property | Traditional Bitcoin | ZEB |
|---|---|---|
| Security basis | Consensus rules only | Execution-gated validity |
| Public broadcast safe against adversarial replacement? | No (RBF exploitable) | Yes (replacement invalid without execution-time material) |
| Requires private channels? | No | No |
| What enables adversarial replacement? | Policy + valid construction | Missing execution-time material |
| Visibility implies capability? | Often | No (under execution-gated constructions) |
| Safe against quantum mempool reaction attacks? | No | Yes (with execution-gated construction) |

---

## 1. Problem Statement

Bitcoin’s public mempool is observable. Any transaction relayed publicly exposes its witness and, in many constructions, reveals attack-critical material required to authorise spending. Reactive adversaries can monitor the public mempool and attempt to construct conflicting spends before confirmation.

Replacement behaviour is governed by mempool policy rather than consensus. Miners MAY include any consensus-valid conflicting transaction they receive. Under a public-mempool adversary model, the critical enabling factor for replacement is pre-confirmation public disclosure.

ZEB removes this enabling factor by requiring an execution-gated spend construction that makes competing spends invalid until execution-time material is revealed, regardless of public visibility.

---

## 2. Threat Model

### 2.1 In-Scope Adversary

The adversary:

* Observes the public mempool.
* Reacts rapidly to public disclosure of transactions.
* MAY derive or recover private keys after seeing witness or other execution material.
* Relies on public-mempool observation as the trigger to act.
* Cannot reliably observe the sender’s non-public submission channel(s) prior to confirmation.

This includes quantum reaction attackers whose attack becomes feasible only once attack-critical material is disclosed during a spend.

### 2.2 Out-of-Scope Adversary

ZEB does not claim protection against an adversary who:

* Has compromised the relevant private keys prior to the spend attempt.
* Can construct a competing spend without mempool observation.
* Can observe non-public submission channels with comparable fidelity to the sender.
* Has miner-equivalent private connectivity, collusion, or censorship capability.

### 2.3 Security Objective

* Deny pre-construction of valid competing spends by requiring an execution-gated spend construction.
* Reduce the attacker’s effective reaction window during the broadcast-to-confirmation period, optionally using relay discipline.

---

## 3. Applicability by Output Type

ZEB’s effectiveness depends on whether attack-critical material is disclosed only at spend time or is already visible on-chain.

### 3.1 Output Types Where ZEB Is Effective

ZEB is effective when used with an execution-gated spend construction and where attack-critical material would otherwise be usable for pre-construction or reactive replacement.

ZEB is most effective when:
* The spend path is execution-gated (competing spends invalid without execution-time material), and
* Long-term attack-critical material is not already exposed on-chain at output creation.

For maximum effectiveness, ZEB SHOULD be applied to first-spend outputs whose public keys have not previously appeared on-chain and whose selected spend path does not embed public keys or other attack-critical material in the locking script.

### 3.2 Output Types Where ZEB Is Not Sufficient

ZEB is not sufficient when:
* The spend construction is not execution-gated (i.e., a valid competing spend can be pre-constructed without execution-time material), or
* Attack-critical material is already visible on-chain (e.g., key material exposed at output creation), enabling preparation independent of mempool observation.

For these cases, ZEB does not address key-compromise risk and MUST NOT be presented as replacement-proof for that spend construction.

---

## 3A. Core Requirement: Execution-Gated Spend Construction

ZEB is defined over an execution-gated spend construction.

An execution-gated spend construction MUST ensure that a valid spend cannot be constructed without execution-time material that is intentionally withheld until the execution attempt.

This specification defines ZEB’s broadcast and monitoring rules, but does not mandate a single script template. Any spend construction is acceptable provided it satisfies the execution-gated property.

Examples of execution-gated spend constructions include script paths that commit to a spend-time secret (or other attempt-scoped material) and verify it during execution.

If a spend construction does not deny pre-construction (i.e., a valid competing spend can be constructed from public information prior to execution), then ZEB cannot claim replacement resistance for that spend and SHOULD be treated as relay-only mitigation.

---

## 4. Non-Goals

ZEB does not:

* Deterministically prevent replacement or miner selection.
* Guarantee inclusion or censorship resistance.
* Protect against miner-connected or pre-compromised-key adversaries.
* Introduce or depend on removed or proposed opcodes.
* Introduce on-chain oracles or external time sources.

---

## 5. Definitions

Public mempool — Unconfirmed transaction pools of publicly reachable nodes.
Non-public submission — Delivery of a transaction to endpoints that do not relay it to the public mempool.
Exposure window — Interval between transaction appearance in the chosen deployment’s observation surface (public mempool or selected monitoring set) and confirmation.
Leak — Unintended public-mempool appearance of a transaction prior to confirmation during an execution attempt that selected non-public submission. If public broadcast was intentionally selected, public appearance is expected and is not a leak.
Confirmation — Inclusion of the transaction in a valid block extending the best chain.

Execution-gated spend — A spend construction in which a valid spend cannot be constructed without execution-time material that is intentionally withheld until the execution attempt.

Execution-time material — Attempt-scoped material required to satisfy the selected execution-gated spend (e.g., a spend-time secret preimage and its commitment) that is not disclosed prior to execution.

---

## 6. Requirements

### 6.1 Wallet and Signer

Implementations **MUST**:

* Treat ZEB as a distinct execution mode.
* Enforce fail-closed behaviour on leak or timeout.
* Prevent public retries of the same attempt.

Implementations **SHOULD**:

* Maintain attempt identifiers and burn attempt material on failure.
* Support deterministic fallback paths.

### 6.2 Networking

ZEB security derives from the execution-gated spend construction. Broadcast method is an operational choice.

Implementations MUST:
* Support public peer-to-peer broadcast of ZEB transactions.
* Support optional non-public submission to a configurable relay set.
* Support public mempool observation for exposure detection and confirmation tracking.

Implementations SHOULD:
* Allow broadcast method selection per attempt (public, non-public, or hybrid).
* Use multiple non-public endpoints when non-public submission is selected.
* Treat non-public submission guarantees as operational assurances, not cryptographic guarantees.

Implementations MUST NOT:
* Assume that public broadcast weakens ZEB security when the spend construction is execution-gated.
* Require miner coordination or trusted miner behavior.

---

## 7. Protocol Overview

ZEB is defined over an execution-gated spend construction. Competing spends remain invalid until execution-time material is revealed.

Broadcast method is an operational choice. Transactions MAY be broadcast publicly or MAY be delivered via non-public submission channels. Non-public submission MAY be used to compress the exposure window but is not required for correctness.

ZEB uses two operational channels:
1. Submission channel (public, non-public, or hybrid) for transaction delivery.
2. Public observation channel for exposure detection and confirmation tracking.

ZEB succeeds if:
* The transaction confirms within the configured window, and
* No policy-violating exposure condition occurs for the selected deployment mode.

ZEB fails closed if:
* Confirmation does not occur by the configured deadline, or
* A deployment-defined exposure condition is met (for example, unintended public appearance during a non-public submission attempt).

---

## 8. State Machine

### 8.1 States

READY
SUBMITTED
CONFIRMED
EXPOSURE_DETECTED
TIMEOUT
FAIL_CLOSED

### 8.2 Transitions

READY → SUBMITTED
SUBMITTED → CONFIRMED
SUBMITTED → EXPOSURE_DETECTED
SUBMITTED → TIMEOUT
EXPOSURE_DETECTED → FAIL_CLOSED
TIMEOUT → FAIL_CLOSED

### 8.3 Fail-Closed Semantics

On FAIL_CLOSED, implementations **MUST**:

* Abort the attempt.
* Invalidate all attempt-specific material.
* Require a fresh attempt or transition to a predeclared fallback.
* Prohibit public rebroadcast of the same transaction.

---

## 9. Exposure Detection

Exposure detection is an operational mechanism used to enforce fail-closed semantics for selected deployment modes. It does not define ZEB’s core security property, which derives from execution-gated validity.

### 9.1 Exposure Definition

An exposure condition is deployment-defined.

For execution attempts that selected non-public submission, the terms “leak” and “exposure condition” are used interchangeably to describe unintended public-mempool appearance prior to confirmation.

If non-public submission was selected for an attempt, an exposure condition MAY be: appearance of the transaction as unconfirmed in any public observation mempool prior to confirmation.

If public broadcast was intentionally selected, public appearance is expected and MUST NOT be treated as an exposure condition.

### 9.2 Observation Strategy

Implementations MUST:
* Maintain at least two independent public observation nodes.
* Query for presence at regular intervals during the unconfirmed period.

### 9.3 Exposure Rule

If an exposure condition is met for the selected deployment mode, the implementation MUST mark the attempt as failed and transition to FAIL_CLOSED.

---

## 10. Confirmation Tracking

Implementations **MUST**:

* Monitor best-chain updates.
* Verify inclusion and block validity.
* On confirmation, transition to CONFIRMED and terminate ZEB for the attempt.

---

## 11. Time Bounds

### 11.1 Confirmation Deadline

Implementations **MUST** define a confirmation_deadline using wall-clock time or block height for local control.

### 11.2 No Public Retry Rule

Implementations **MUST NOT** publicly rebroadcast the same transaction upon timeout. Retries require a new attempt with fresh material or a deterministic fallback.

---

## 12. Fallback Strategy

At least one fallback **MUST** be defined and **MUST**:

* Avoid reuse of disclosed attempt material.
* Avoid public retries of the same attempt.
* Be deterministic under local policy.

Fallback strategies **MAY** include constructing a fresh attempt with new identifiers or transitioning to a predeclared recovery path.

---

## 13. Example Transactions (Illustrative)

### 13.1 Example Input UTXO

prev_txid = aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa  
prev_vout = 0  
value = 10000000 sats  
scriptPubKey_type = P2WPKH  
scriptPubKey = 0014b0b1b2b3b4b5b6b7b8b9babbbcbdbebfc0c1c2c3  

### 13.2 Spend Attempt Transaction (Tx1)

version = 2  
locktime = 0  

inputs[0]:

* txid = prev_txid  
* vout = 0  
* sequence = 0xfffffffd  

outputs[0]:

* value = 9900000 sats  
* scriptPubKey = 0014d1d2d3d4d5d6d7d8d9dadbdcdddedfe0e1e2e3e4  

outputs[1]:

* value = 90000 sats  
* scriptPubKey = 0014c1c2c3c4c5c6c7c8c9cacbcccdcecfd0d1d2d3d4  

fee = 10000 sats  

Witness:

* Signature and public key appear only at execution.
* This transaction structure is illustrative only. P2WPKH is not an execution-gated spend construction. For ZEB deployment, an execution-gated construction (as defined in Section 3A) MUST be used; this example shows transaction anatomy, not a secure execution-gated spend.

### 13.3 Adversarial Conflicting Spend (Tx1′)

A public-mempool adversary attempts to construct a conflicting spend (Tx1′) by observing Tx1 and reacting before confirmation. If Tx1 uses a non-execution-gated construction (such as P2WPKH), a competing spend may be possible once execution material is disclosed. When Tx1 uses an execution-gated spend construction as defined in Section 3A, the adversary cannot construct a valid competing spend without execution-time material, regardless of transaction visibility.

---

## 14. Security Considerations

### 14.1 Effectiveness Statement

Against public-mempool reaction attackers whose ability to act depends on constructing a valid competing spend during the broadcast-to-confirmation window, ZEB prevents pre-construction by requiring an execution-gated spend construction. Optional relay discipline may further compress the remaining exposure window, but correctness does not depend on transaction secrecy.

### 14.2 Limitations

ZEB does not protect against:

* Attackers with pre-compromised keys.
* Miner-connected adversaries.
* Output types with pre-existing on-chain public keys.
* Censorship or denial of non-public submission channels.

Some implementations MAY choose to apply additional execution-time hardening techniques beyond the scope of this specification. Such techniques are intentionally not standardised here, as they introduce trade-offs in cost, complexity, and relay behaviour that are deployment-specific.

---

## 15. Summary

Zero-Exposure Broadcast (ZEB) is an execution mode defined over an execution-gated spend construction. Its core security property is denial of pre-construction: competing spends remain invalid until execution-time material is revealed.

ZEB may be used with public broadcast or with optional non-public submission. Non-public submission can compress the exposure window but is not required for correctness when the spend construction is execution-gated.

ZEB requires no consensus changes, introduces no new opcodes, and does not rely on miner trust or coordination.

---

## 16. Layered Quantum Mitigation Summary (Informative)

ZEB is designed to operate alongside BIP-360 execution paths and PQHD custody controls. Each mechanism addresses a distinct quantum-relevant attack surface without introducing new consensus rules.

```
Layer                   | Attack Surface Addressed        | Mitigation Mechanism
----------------------- | ------------------------------- | -----------------------------------------------
BIP-360 P2TSH           | At-rest key exposure            | Removes key-path spending; public key hidden until spend
PQHD                   | Custody and authorisation       | Post-quantum authorisation; classical key compromise alone is insufficient
Zero-Exposure Broadcast | Replacement & reaction window | Execution-gated validity; optional relay discipline
```

Each layer operates at a different phase of the transaction lifecycle:

* BIP-360 reduces long-term on-chain exposure prior to spend.
* PQHD enforces one-shot, post-quantum authorisation at execution time.
* ZEB denies pre-construction by requiring execution-gated validity; optional relay discipline may further compress the residual exposure window during broadcast-to-confirmation.

These mechanisms are complementary. None individually guarantees protection against all quantum-capable adversaries, but together they materially reduce exposure across storage, authorisation, and broadcast phases under the stated threat models.

---

## 17. Threat Boundary Summary (Informative)

| Adversary Capability                                                                          | Mitigated |
| --------------------------------------------------------------------------------------------- | --------- |
| Public-mempool reaction after witness disclosure                                               | Yes       |
| Key derivation triggered by execution-time disclosure (execution-gated construction required) | Yes       |
| Pre-compromised keys prior to spend                                                           | No        |
| Miner-connected or miner-colluding adversary                                                   | No        |
| Outputs with pre-existing on-chain public keys                                                 | No        |
| Censorship of non-public submission channels                                                   | No        |

---

## 18. Implementation Notes (Informative)

### 18.1 Non-Public Submission Channels

Non-public submission refers to any delivery mechanism that avoids public mempool relay. Acceptable approaches include:

* Limited-relay peers configured not to forward transactions.
* Miner-connected peers or pool submission endpoints.
* Transaction relay services that forward to miners without public broadcast.

Non-public submission is an optional optimization under ZEB. It MAY be used to compress the exposure window and reduce metadata leakage, but it is not required for correctness when the spend construction is execution-gated. Public peer-to-peer broadcast remains a valid submission method.

### 18.2 Public Mempool Monitoring

Implementations **SHOULD** monitor multiple independent public nodes for transaction appearance. Monitoring sources **SHOULD** be diverse in network topology and implementation.

Observation frequency during SUBMITTED **SHOULD** be sufficient to detect exposure conditions promptly.

### 18.3 Fee Strategy Considerations

ZEB assumes a bounded confirmation window. Implementations **SHOULD** select fees appropriate for the desired confirmation target and prevailing network conditions.

Fee strategy is an operational parameter and does not affect ZEB correctness.

### 18.4 Timeout Handling

If confirmation does not occur within the configured window, the implementation **MUST** fail closed and invoke fallback logic. Repeated public retries of the same transaction **MUST NOT** occur.

---

# Annexes

---

# Annex A — Zero-Exposure Broadcast (ZEB) Integration with BIP-360 P2TSH

Status: Informative Annex
Compatible with Bitcoin current consensus and policy.
No new opcodes. No consensus changes.

---

## A.1 Purpose

This annex describes how Zero-Exposure Broadcast (ZEB) integrates with **BIP-360 P2TSH (Pay-to-Tapscript-Hash)** execution paths to mitigate public-mempool quantum reaction attacks in scenarios where attack-critical material is disclosed only at spend time.

BIP-360 P2TSH refers to Taproot outputs with the key path removed, requiring all spends to occur via script paths.

ZEB is an execution-mode constraint, not a script primitive. It specifies how a BIP-360 P2TSH spend is delivered and monitored, without modifying validity rules, script semantics, or covenant behaviour.

---

## A.2 Scope and Applicability

ZEB integration applies to BIP-360 P2TSH execution paths that satisfy all of the following:

* The spend path reveals attack-critical material only in the witness.
* The locking script does not embed public keys or other attack-critical material prior to spending.
* The adversary’s ability to act depends on pre-confirmation public-mempool disclosure.

ZEB does not apply to:

* BIP-360 paths whose locking scripts reveal public keys on-chain.
* Adversaries with miner-equivalent connectivity or pre-compromised keys.
* Any execution path where a competing spend can be constructed without observing the public mempool.

---

## A.3 Threat Model Alignment

ZEB assumes the BIP-360 public-mempool adversary:

* Observes public mempools.
* Reacts to witness or execution-time disclosure.
* Attempts replacement during the broadcast-to-confirmation window.

ZEB’s core security property in this integration is execution-gated validity: competing spends remain invalid without execution-time material. Non-public submission MAY be used as an optional optimization to compress the exposure window, but ZEB does not require it for correctness when the spend construction is execution-gated.


---

## A.4 Integration Model

### A.4.1 Execution Path Binding

When a BIP-360 P2TSH execution path is selected for spend, the sender MAY designate the spend as ZEB-constrained.

A ZEB-constrained spend:

* MUST use an execution-gated spend construction as defined in Section 3A (i.e., competing spends remain invalid without execution-time material).
* MAY be submitted via public broadcast or via non-public submission channels.
* MUST be monitored for deployment-defined exposure conditions during the unconfirmed period.

This designation does not alter the script, witness, or transaction structure.

### A.4.2 Submission

For a ZEB-constrained BIP-360 P2TSH spend, the transaction MAY be delivered using one or more submission routes, including:

* Public peer-to-peer broadcast.
* Limited-relay peers.
* Miner-connected peers or pool submission endpoints.
* Transaction accelerator endpoints.

Non-public submission MAY be used as an optional optimization to compress the exposure window, but is not required for correctness when the spend construction is execution-gated.

### A.4.3 Exposure Detection

An implementation integrating ZEB with BIP-360 MUST monitor public mempools for the transaction identifier during the unconfirmed period.

If non-public submission was selected for an attempt, unintended public appearance prior to confirmation MAY be treated as an exposure condition according to Section 9.

If public broadcast was intentionally selected, public appearance is expected and MUST NOT be treated as an exposure condition.

### A.4.4 Fail-Closed Semantics

Upon an exposure condition or timeout, the BIP-360 execution MUST:

* Abort the active execution attempt.
* Treat the attempt as compromised.
* Invalidate any attempt-specific ephemeral material.
* Transition to a predeclared BIP-360 fallback or recovery path.

The implementation MUST NOT retry the same transaction publicly using the same attempt material.

---

## A.5 Confirmation Window

ZEB integration requires a bounded confirmation window.

If confirmation does not occur within the configured window:

* The ZEB-constrained execution **MUST** fail closed.
* Control **MUST** return to BIP-360 recovery or re-execution logic using fresh attempt identifiers.

The confirmation window is an operational parameter and does not affect consensus.

---

## A.6 Interaction with BIP-360 Recovery Paths

ZEB integrates cleanly with BIP-360 recovery semantics:

* Recovery paths **MAY** be designated as ZEB-constrained or non-ZEB.
* ZEB does not alter recovery eligibility, timelocks, or quorum rules.
* ZEB constrains visibility prior to confirmation only.

A recovery path that is not ZEB-constrained **MAY** be broadcast publicly, subject to BIP-360 policy.

---

## A.7 Script and Consensus Considerations

ZEB integration introduces:

* No new script operations.
* No changes to transaction validity.
* No changes to BIP-360 covenant semantics.
* No reliance on external oracles.

All ZEB-constrained BIP-360 P2TSH transactions remain standard and consensus-valid.

---

## A.8 Security Considerations

### A.8.1 Public-Mempool Reaction Attacks

When integrated with BIP-360 P2TSH execution paths that use an execution-gated spend construction, ZEB denies pre-construction of valid competing spends by requiring execution-time material that is not available prior to execution.

Optional relay discipline (including non-public submission) MAY be used to compress the remaining exposure window and reduce metadata leakage, but ZEB does not require restricted dissemination for correctness when the spend construction is execution-gated.


### A.8.2 Limitations

ZEB does not protect against:

* Miner-connected adversaries.
* Pre-compromised keys.
* Output types with pre-existing on-chain public keys.
* Censorship or denial of non-public submission channels.

These limitations are unchanged from the base BIP-360 threat model.

---

## A.9 Summary

This annex specifies the integration of Zero-Exposure Broadcast with BIP-360 P2TSH execution paths.

ZEB adds no new trust assumptions, opcodes, or consensus rules. It provides an operational mitigation against public-mempool quantum reaction attacks in applicable BIP-360 spend paths by controlling broadcast visibility and enforcing deterministic fail-closed behaviour.

---

# Annex B — Zero-Exposure Broadcast (ZEB) Integration with PQHD

Status: Informative Annex
Compatible with Bitcoin current consensus and policy.
No new opcodes. No consensus changes.

---

## B.1 Purpose

This annex specifies how Zero-Exposure Broadcast (ZEB) integrates with PQHD (Post-Quantum Hierarchical Deterministic custody and execution models) - https://github.com/rosieRRRRR/PQHD
 - to mitigate public-mempool quantum reaction attacks while preserving PQHD’s existing authority, consent, and recovery semantics.

ZEB is an execution-visibility constraint layered on top of PQHD. It does not modify PQHD’s cryptographic predicates, custody tiers, or key-management guarantees. It constrains how and when a PQHD-authorised transaction is disclosed to the network.

---

## B.2 Scope and Applicability

ZEB integration applies to PQHD execution flows where:

* The PQHD spend path reveals attack-critical material (including public keys, script data, or spend-time secrets) only at execution.
* The adversary’s ability to act depends on pre-confirmation public-mempool disclosure.
* The execution attempt is governed by PQHD’s one-shot consent, epoch freshness, and attempt-scoped authorisation.

ZEB does not apply to:

* PQHD outputs whose locking scripts reveal public keys or attack-critical material on-chain.
* Adversaries with pre-compromised keys independent of execution-time disclosure.
* Adversaries with miner-equivalent connectivity or visibility into non-public submission channels.

---

## B.3 Threat Model Alignment

ZEB assumes the PQHD public-mempool adversary:

* Observes public mempools.
* Reacts to execution-time disclosure.
* **MAY** derive or recover keys rapidly after disclosure.
* Cannot reliably observe PQHD’s non-public submission channels prior to confirmation.

ZEB is orthogonal to PQHD’s trust, quorum, and recovery models. It constrains visibility prior to confirmation only and does not alter authority boundaries.

---

## B.4 Integration Model

### B.4.1 ZEB-Constrained PQHD Execution

When a PQHD execution attempt is authorised, the implementation MAY mark the attempt as ZEB-constrained.

A ZEB-constrained PQHD execution:

* MUST use an execution-gated spend construction as defined in Section 3A (i.e., competing spends remain invalid without execution-time material).
* MAY be submitted via public broadcast or via non-public submission channels.
* MUST be monitored for deployment-defined exposure conditions during the unconfirmed period.
* MUST obey PQHD’s attempt-scoped, single-use semantics.

This designation does not modify the underlying transaction, script, or witness structure.

### B.4.2 Interaction with PQHD Consent and Epoch Semantics

ZEB integrates directly with PQHD’s existing controls:

* Consent objects MUST remain one-shot and attempt-scoped.
* Epoch freshness MUST bound the authorisation window.
* Ledger monotonicity MUST ensure that once an attempt enters ZEB execution, it cannot be replayed or reused.

ZEB does not introduce additional cryptographic checks. It applies an execution-visibility policy during the authorised window and relies on the execution-gated spend construction to deny pre-construction.

### B.4.3 Submission

For a ZEB-constrained PQHD execution, the transaction MAY be delivered using one or more submission routes, including:

* Public peer-to-peer broadcast.
* Limited-relay peers.
* Miner-connected peers or pool submission endpoints.
* Transaction accelerator or pool submission endpoints.

Non-public submission MAY be used as an optional optimization to compress the exposure window, but is not required for correctness when the spend construction is execution-gated.

### B.4.4 Exposure Detection

A PQHD implementation integrating ZEB MUST monitor public mempools for the transaction identifier during the unconfirmed period.

If non-public submission was selected for an attempt, unintended public appearance prior to confirmation MAY be treated as an exposure condition according to Section 9.

If public broadcast was intentionally selected, public appearance is expected and MUST NOT be treated as an exposure condition.

### B.4.5 Fail-Closed Semantics

Upon an exposure condition, the PQHD execution MUST:

* Abort the active execution attempt.
* Mark the attempt as compromised.
* Invalidate all attempt-specific ephemeral material, including execution-time material and consent tokens.
* Transition to a PQHD-defined fallback or recovery path.

The implementation MUST NOT retry the same transaction publicly using the same attempt material.

---

## B.5 Confirmation Window

ZEB integration requires a bounded confirmation window aligned with PQHD policy.

If confirmation does not occur within the configured window:

* The ZEB-constrained PQHD execution **MUST** fail closed.
* Control **MUST** return to PQHD recovery or re-execution logic using fresh attempt identifiers and fresh consent.

The confirmation window is an operational parameter and does not affect consensus.

---

## B.6 Interaction with PQHD Recovery Paths

ZEB integrates cleanly with PQHD recovery semantics:

* Recovery paths **MAY** be designated as ZEB-constrained or non-ZEB.
* ZEB does not alter PQHD timelocks, quorum thresholds, or recovery eligibility.
* ZEB constrains visibility prior to confirmation only and does not alter recovery authority.

A recovery path that is not ZEB-constrained **MAY** be broadcast publicly, subject to PQHD policy.

---

## B.7 Script and Consensus Considerations

ZEB integration with PQHD introduces:

* No new script operations.
* No changes to transaction validity.
* No changes to PQHD custody tiers or covenant semantics.
* No reliance on external oracles.

All ZEB-constrained PQHD transactions remain standard and consensus-valid.

---

## B.8 Security Considerations

### B.8.1 Public-Mempool Reaction Attacks

When integrated with PQHD execution paths that use an execution-gated spend construction, ZEB denies pre-construction of valid competing spends by requiring execution-time material that is not available prior to execution.

Optional relay discipline (including non-public submission) MAY be used to compress the remaining exposure window and reduce metadata leakage, but ZEB does not require restricted dissemination for correctness when the spend construction is execution-gated.


### B.8.2 Limitations

ZEB does not protect PQHD execution against:

* Pre-compromised keys independent of execution disclosure.
* Miner-connected adversaries.
* Output types with pre-existing on-chain public keys.
* Censorship or denial of non-public submission channels.

These limitations are consistent with PQHD’s stated threat boundaries.

---

## B.9 Summary

This annex defines the integration of Zero-Exposure Broadcast with PQHD execution flows.

ZEB adds no new trust assumptions, opcodes, or consensus rules. When used with an execution-gated spend construction (as defined in Section 3A), ZEB denies pre-construction of valid competing spends by requiring execution-time material that is not available prior to execution.

When used with non-execution-gated spend constructions, ZEB does not claim replacement resistance and SHOULD be treated as relay-only mitigation that may reduce exposure but does not change validity properties.

Optional relay discipline (including non-public submission) MAY be used in either case to compress the remaining exposure window and reduce metadata leakage.

---

# Appendix C — Common Misinterpretations

This appendix addresses frequent misunderstandings about Zero-Exposure Broadcast (ZEB) and distinguishes core security properties from deployment options.

## C.1 “ZEB is a private relay protocol”

Incorrect. ZEB’s core security property derives from execution-gated validity: competing spends remain invalid until execution-time material is revealed. Non-public submission is an optional deployment optimization, not the security mechanism.

## C.2 “Public broadcast breaks ZEB security”

Incorrect. ZEB remains secure under public broadcast when the selected spend construction is execution-gated, because transaction visibility does not enable construction of valid competing spends.

## C.3 “Exposure detection indicates a security failure”

Incorrect. Exposure detection enforces fail-closed semantics for execution attempts that selected non-public submission. It does not indicate a weakness in execution-gated validity.

## C.4 “ZEB requires miner cooperation”

Incorrect. ZEB requires no miner coordination, trust, or special behavior. It works entirely within standard Bitcoin Script and consensus semantics.


# **ACKNOWLEDGEMENTS (INFORMATIVE)**

This specification builds on decades of work in cryptography, distributed systems, and the design and operation of the Bitcoin protocol.

The author acknowledges the foundational contributions of the following individuals and communities, whose prior work made the concepts formalised in Zero-Exposure Broadcast (ZEB), and its integration with BIP-360-style script-path execution, possible:

* **Satoshi Nakamoto** — for the original design of Bitcoin and its trust-minimised consensus model.
* **Hal Finney** — for early Bitcoin implementation, applied cryptographic insight, and work on secure transaction relay.
* **Adam Back** — for Hashcash and foundational work on proof-of-work systems.
* **Pieter Wuille** — for SegWit, Taproot, Miniscript, PSBT semantics, and deep contributions to Bitcoin’s transaction and script architecture.
* **Greg Maxwell** — for extensive work on Bitcoin security, adversarial analysis, transaction malleability, and mempool and relay dynamics.
* **Andrew Poelstra** — for Miniscript, script composability, and formal reasoning about Bitcoin spending policies.
* **Ethan Heilman**, **Hunter Beast**, and **Isabel Foxen Duke** — for their combined work on the BIP-360 specification.
* **Peter Shor** — for demonstrating the impact of quantum computation on widely deployed cryptographic schemes.
* **Daniel J. Bernstein** — for cryptographic engineering discipline and post-quantum cryptographic research.
* **The Bitcoin Core developer community** — for the ongoing design, implementation, and review of Bitcoin’s networking, mempool, and transaction-relay behaviour.
* **Authors and contributors to Bitcoin Improvement Proposals** — for formalising interfaces, threat models, and deployment constraints relevant to script-path-only execution and transaction propagation.

* **The NIST Post-Quantum Cryptography Project** — for the standardisation and evaluation of post-quantum cryptographic primitives.

Acknowledgement is also due to independent researchers, node operators, miners, and reviewers who have examined mempool behaviour, transaction propagation, and adversarial reaction dynamics, and whose work has informed the operational assumptions underlying this specification.

---

## Support (Informative)

If you find this work useful and wish to support continued independent
research and documentation, voluntary contributions are welcome.

Bitcoin: bc1q380874ggwuavgldrsyqzzn9zmvvldkrs8aygkw


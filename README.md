# ZEB — Zero-Exposure Broadcast

An Open Standard for Mempool-Resilient Post-Quantum Transaction Execution

Status: Implementation Ready (Protocol Review Requested)
Specification Version: 1.0.0
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

Zero-Exposure Broadcast (ZEB) defines an execution mode that mitigates public-mempool quantum reaction attacks by ensuring that a transaction’s witness and execution material are not disclosed to the public mempool prior to confirmation. The first public disclosure of execution material occurs only in a mined block, collapsing the attacker’s public-mempool reaction window to approximately zero.

ZEB targets adversaries whose ability to act depends on pre-confirmation disclosure of attack-critical material, including public keys, script data, or other witness elements revealed at spend time. ZEB requires no consensus changes, introduces no new opcodes, and does not rely on miner trust. It specifies controlled non-public submission, public-mempool exposure detection, bounded confirmation windows, and deterministic fail-closed fallback.

ZEB does not claim protection against attackers who have already compromised keys prior to the spend attempt, attackers who can act independently of mempool observation, or attackers with miner-equivalent connectivity.

---

## 1. Problem Statement

Bitcoin’s public mempool is observable. Any transaction relayed publicly exposes its witness and, in many constructions, reveals attack-critical material required to authorise spending. Reactive adversaries can monitor the public mempool and attempt to construct conflicting spends before confirmation.

Replacement behaviour is governed by mempool policy rather than consensus. Miners MAY include any consensus-valid conflicting transaction they receive. Under a public-mempool adversary model, the critical enabling factor for replacement is pre-confirmation public disclosure.

ZEB removes this enabling factor by preventing public-mempool disclosure prior to confirmation.

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

* Prevent public pre-confirmation disclosure of witness or execution material.
* Collapse the public-mempool reaction window to approximately zero.

---

## 3. Applicability by Output Type

ZEB’s effectiveness depends on whether attack-critical material is disclosed only at spend time or is already visible on-chain.

### 3.1 Output Types Where ZEB Is Effective

ZEB is effective for constructions where attack-critical material is revealed only in the witness, including:

* Hash-to-public-key outputs such as P2WPKH.
* Script-hash constructions such as P2WSH when the script does not embed public keys or other attack-critical material prior to spending.
* Any construction where the attacker’s trigger depends on execution-time revelation.

In these cases, ZEB prevents the attacker from learning the necessary material prior to confirmation.

For maximum effectiveness, ZEB SHOULD be applied to first-spend outputs whose public keys have not previously appeared on-chain.

### 3.2 Output Types Where ZEB Is Not Sufficient

ZEB is not sufficient for constructions where attack-critical material is already visible on-chain, including:

* Taproot key-path outputs where the public key is revealed at output creation.
* Script paths that embed public keys or other sensitive material in the locking script.
* Any construction where an attacker can prepare a competing spend without observing the mempool.

For these cases, ZEB does not address key-compromise risk.

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
Exposure window — Interval between first public disclosure of execution material and confirmation.
Leak — Appearance of the transaction in any public mempool prior to confirmation.
Confirmation — Inclusion of the transaction in a valid block extending the best chain.

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

Implementations **MUST**:

* Support non-public submission to a configurable relay set.
* Treat any route that causes public relay as a leak.
* Support public mempool observation for exposure detection.

Implementations **SHOULD**:

* Use multiple non-public endpoints to reduce single-channel failure.
* Treat non-public submission guarantees as operational assurances, not cryptographic guarantees.

---

## 7. Protocol Overview

ZEB uses two channels:

1. Non-public submission channel for initial transaction delivery.
2. Public observation channel for exposure detection and confirmation tracking.

ZEB succeeds if:

* Non-public submission completes.
* No public-mempool leak occurs prior to confirmation.
* Confirmation occurs within a bounded window.

ZEB fails closed if:

* A leak occurs before confirmation, or
* Confirmation does not occur by the configured deadline.

---

## 8. State Machine

### 8.1 States

READY
SUBMITTED_NON_PUBLIC
CONFIRMED
LEAK_DETECTED
TIMEOUT
FAIL_CLOSED

### 8.2 Transitions

READY → SUBMITTED_NON_PUBLIC
SUBMITTED_NON_PUBLIC → CONFIRMED
SUBMITTED_NON_PUBLIC → LEAK_DETECTED
SUBMITTED_NON_PUBLIC → TIMEOUT
LEAK_DETECTED → FAIL_CLOSED
TIMEOUT → FAIL_CLOSED

### 8.3 Fail-Closed Semantics

On FAIL_CLOSED, implementations **MUST**:

* Abort the attempt.
* Invalidate all attempt-specific material.
* Require a fresh attempt or transition to a predeclared fallback.
* Prohibit public rebroadcast of the same transaction.

---

## 9. Exposure Detection

### 9.1 Leak Definition

A leak occurs if the transaction identifier appears as unconfirmed on any public observation node before confirmation.

### 9.2 Observation Strategy

Implementations **MUST**:

* Maintain at least two independent public observation nodes.
* Query for presence at regular intervals during SUBMITTED_NON_PUBLIC.

### 9.3 Leak Rule

If the transaction appears in any public mempool prior to confirmation, the implementation **MUST** mark LEAK_DETECTED and transition to FAIL_CLOSED.

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
* Under ZEB, this witness is not disclosed to the public mempool.

### 13.3 Adversarial Conflicting Spend (Tx1′)

A public-mempool adversary relies on observing Tx1’s execution material to derive keys or construct a competing spend. Under ZEB, Tx1 is not publicly visible pre-confirmation, so the adversary has no public trigger.

---

## 14. Security Considerations

### 14.1 Effectiveness Statement

Against public-mempool quantum reaction attackers whose ability to act depends on pre-confirmation disclosure of attack-critical material, ZEB collapses the reaction window to approximately zero by ensuring that the first public disclosure occurs only after confirmation.

### 14.2 Limitations

ZEB does not protect against:

* Attackers with pre-compromised keys.
* Miner-connected adversaries.
* Output types with pre-existing on-chain public keys.
* Censorship or denial of non-public submission channels.

Some implementations MAY choose to apply additional execution-time hardening techniques beyond the scope of this specification. Such techniques are intentionally not standardised here, as they introduce trade-offs in cost, complexity, and relay behaviour that are deployment-specific.

---

## 15. Summary

Zero-Exposure Broadcast is an operational execution mode that prevents public-mempool disclosure of witness and execution material prior to confirmation. For output types where attack-critical material is revealed only at spend time, ZEB provides a practical near-term mitigation against public-mempool quantum reaction attacks without requiring consensus changes or the reintroduction of dangerous opcodes.

---

## 16. Layered Quantum Mitigation Summary (Informative)

ZEB is designed to operate alongside BIP-360 execution paths and PQHD custody controls. Each mechanism addresses a distinct quantum-relevant attack surface without introducing new consensus rules.

```
Layer                   | Attack Surface Addressed        | Mitigation Mechanism
----------------------- | ------------------------------- | -----------------------------------------------
BIP-360 P2TSH           | At-rest key exposure            | Removes key-path spending; public key hidden until spend
PQHD                   | Custody and authorisation       | Post-quantum authorisation; classical key compromise alone is insufficient
Zero-Exposure Broadcast | Public-mempool reaction window  | Non-public broadcast; pre-confirmation disclosure avoided
```

Each layer operates at a different phase of the transaction lifecycle:

* BIP-360 reduces long-term on-chain exposure prior to spend.
* PQHD enforces one-shot, post-quantum authorisation at execution time.
* ZEB collapses the public-mempool reaction window during broadcast.

These mechanisms are complementary. None individually guarantees protection against all quantum-capable adversaries, but together they materially reduce exposure across storage, authorisation, and broadcast phases under the stated threat models.

---

## 17. Threat Boundary Summary (Informative)

| Adversary Capability                                  | Mitigated |
| ----------------------------------------------------- | --------- |
| Public-mempool reaction after witness disclosure      | Yes       |
| Key derivation triggered by execution-time disclosure | Yes       |
| Pre-compromised keys prior to spend                   | No        |
| Miner-connected or miner-colluding adversary          | No        |
| Outputs with pre-existing on-chain public keys        | No        |
| Censorship of non-public submission channels          | No        |

---

## 18. Implementation Notes (Informative)

### 18.1 Non-Public Submission Channels

Non-public submission refers to any delivery mechanism that avoids public mempool relay. Acceptable approaches include:

* Limited-relay peers configured not to forward transactions.
* Miner-connected peers or pool submission endpoints.
* Transaction relay services that forward to miners without public broadcast.

ZEB does not require identifying or trusting miners. Any route that avoids public mempool disclosure satisfies the requirement.

### 18.2 Public Mempool Monitoring

Implementations **SHOULD** monitor multiple independent public nodes for transaction appearance. Monitoring sources **SHOULD** be diverse in network topology and implementation.

Observation frequency during SUBMITTED_NON_PUBLIC **SHOULD** be sufficient to detect leaks promptly.

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
* Cannot reliably observe non-public submission channels.

ZEB is orthogonal to BIP-360 custody, quorum, and recovery semantics. It constrains broadcast visibility only and does not alter authority.

---

## A.4 Integration Model

### A.4.1 Execution Path Binding

When a BIP-360 P2TSH execution path is selected for spend, the sender **MAY** designate the spend as ZEB-constrained.

A ZEB-constrained spend:

* **MUST** be submitted via non-public submission channels.
* **MUST NOT** be publicly rebroadcast prior to confirmation.
* **MUST** be monitored for public-mempool exposure.

This designation does not alter the script, witness, or transaction structure.

---

### A.4.2 Non-Public Submission

For a ZEB-constrained BIP-360 P2TSH spend, the transaction **SHOULD** be delivered using one or more non-public routes, including:

* Limited-relay peers.
* Miner-connected peers.
* Transaction accelerator endpoints.

ZEB does not require identifying miners or trusting miner behaviour. Any route that avoids public mempool relay is acceptable.

---

### A.4.3 Exposure Detection

An implementation integrating ZEB with BIP-360 **MUST** monitor public mempools for the transaction identifier during the unconfirmed period.

If the transaction appears in any public mempool prior to confirmation, this **MUST** be treated as a ZEB exposure event.

---

### A.4.4 Fail-Closed Semantics

Upon a ZEB exposure event, the BIP-360 execution **MUST**:

* Abort the active execution attempt.
* Treat the attempt as compromised.
* Invalidate any attempt-specific ephemeral material.
* Transition to a predeclared BIP-360 fallback or recovery path.

The implementation **MUST NOT** retry the same transaction publicly.

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

### A.8.1 Public-Mempool Quantum Reaction Attacks

When integrated with BIP-360 P2TSH execution paths that reveal attack-critical material only at spend time, ZEB prevents pre-confirmation public disclosure and collapses the public-mempool reaction window to approximately zero.

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

This annex specifies how Zero-Exposure Broadcast (ZEB) integrates with **PQHD (Post-Quantum Hierarchical Deterministic custody and execution models)** to mitigate public-mempool quantum reaction attacks while preserving PQHD’s existing authority, consent, and recovery semantics.

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

When a PQHD execution attempt is authorised, the implementation **MAY** mark the attempt as ZEB-constrained.

A ZEB-constrained PQHD execution:

* **MUST** be submitted via non-public submission channels.
* **MUST NOT** be publicly rebroadcast prior to confirmation.
* **MUST** be monitored for public-mempool exposure.
* **MUST** obey PQHD’s attempt-scoped, single-use semantics.

This designation does not modify the underlying transaction, script, or witness structure.

---

### B.4.2 Interaction with PQHD Consent and Epoch Semantics

ZEB integrates directly with PQHD’s existing controls:

* Consent objects **MUST** remain one-shot and attempt-scoped.
* Epoch freshness **MUST** bound the authorisation window.
* Ledger monotonicity **MUST** ensure that once an attempt enters ZEB execution, it cannot be replayed or reused.

ZEB does not introduce additional cryptographic checks. It enforces a stricter execution-visibility policy during the authorised window.

---

### B.4.3 Non-Public Submission

For ZEB-constrained PQHD execution, the transaction **SHOULD** be delivered using one or more non-public routes, including:

* Limited-relay peers.
* Miner-connected peers.
* Transaction accelerator or pool submission endpoints.

ZEB does not require identifying miners or trusting miner behaviour. Any route that avoids public mempool relay is acceptable.

---

### B.4.4 Exposure Detection

A PQHD implementation integrating ZEB **MUST**:

* Monitor public mempools for the transaction identifier during the unconfirmed period.
* Treat any appearance of the transaction in a public mempool prior to confirmation as a ZEB exposure event.

---

### B.4.5 Fail-Closed Semantics

Upon a ZEB exposure event, the PQHD execution **MUST**:

* Abort the active execution attempt.
* Mark the attempt as compromised.
* Invalidate all attempt-specific ephemeral material, including spend-time secrets and consent tokens.
* Transition to a PQHD-defined fallback or recovery path.

The implementation **MUST NOT** retry the same transaction publicly.

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

### B.8.1 Public-Mempool Quantum Reaction Attacks

When integrated with PQHD execution paths that reveal attack-critical material only at spend time, ZEB prevents pre-confirmation public disclosure and collapses the public-mempool reaction window to approximately zero.

This preserves PQHD’s one-shot authority guarantees against adversaries whose attack feasibility depends on execution-time disclosure.

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

ZEB adds no new trust assumptions, opcodes, or consensus rules. It strengthens PQHD’s post-quantum execution safety by ensuring that PQHD-authorised spends are not exposed to the public mempool prior to confirmation, thereby mitigating public-mempool quantum reaction attacks within PQHD’s existing authority and recovery framework.

---

# **ACKNOWLEDGEMENTS (INFORMATIVE)**

This specification builds on decades of work in cryptography, distributed systems, and the design and operation of the Bitcoin protocol.

The author acknowledges the foundational contributions of the following individuals and communities, whose prior work made the concepts formalised in Zero-Exposure Broadcast (ZEB), and its integration with BIP-360-style script-path execution, possible:

* **Satoshi Nakamoto** — for the original design of Bitcoin and its trust-minimised consensus model.
* **Hal Finney** — for early Bitcoin implementation, applied cryptographic insight, and work on secure transaction relay.
* **Adam Back** — for Hashcash and foundational work on proof-of-work systems.
* **Pieter Wuille** — for SegWit, Taproot, Miniscript, PSBT semantics, and deep contributions to Bitcoin’s transaction and script architecture.
* **Greg Maxwell** — for extensive work on Bitcoin security, adversarial analysis, transaction malleability, and mempool and relay dynamics.
* **Andrew Poelstra** — for Miniscript, script composability, and formal reasoning about Bitcoin spending policies.
* **Ethan Heilman** — for research on transaction relay, network-layer adversaries, and attack models relevant to mempool-visible behaviour.
* **Hunter Beast** — for contributions to Bitcoin protocol design discussions and work relevant to script-path execution and operational security.
* **Isabel Foxen Duke** — for sustained work on Bitcoin wallet architecture, operational security, and user-facing custody models.
* **Peter Shor** — for demonstrating the impact of quantum computation on widely deployed cryptographic schemes.
* **Daniel J. Bernstein** — for cryptographic engineering discipline and post-quantum cryptographic research.
* **The Bitcoin Core developer community** — for the ongoing design, implementation, and review of Bitcoin’s networking, mempool, and transaction-relay behaviour.
* **Authors and contributors to Bitcoin Improvement Proposals (including BIP-360)** — for formalising interfaces, threat models, and deployment constraints relevant to script-path-only execution and transaction propagation.
* **The NIST Post-Quantum Cryptography Project** — for the standardisation and evaluation of post-quantum cryptographic primitives.

Acknowledgement is also due to independent researchers, node operators, miners, and reviewers who have examined mempool behaviour, transaction propagation, and adversarial reaction dynamics, and whose work has informed the operational assumptions underlying this specification.

---

## Support (Informative)

If you find this work useful and wish to support continued independent
research and documentation, voluntary contributions are welcome.

Bitcoin: bc1q380874ggwuavgldrsyqzzn9zmvvldkrs8aygkw


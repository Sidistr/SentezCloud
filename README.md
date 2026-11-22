# Sentez Cloud (v1.2)

**Antinomy-Driven Cloud Compartmentalization, Offline Telemetry Vaults, and Explicit Residual-Risk Boundaries**

[![Version](https://img.shields.io/badge/version-v1.2-blue.svg)](paper/sentez_cloud_v1_2.pdf)
[![License: CC BY 4.0](https://img.shields.io/badge/Paper_License-CC_BY_4.0-lightgrey.svg)](LICENSE)
[![License: MIT](https://img.shields.io/badge/Code_License-MIT-green.svg)](LICENSE)
[![DOI](https://img.shields.io/badge/Zenodo-DOI_Pending-orange.svg)](https://doi.org/10.5281/zenodo.[[XXXXXXX]](https://doi.org/10.5281/zenodo.24447579))

> **Position Paper:** [`paper/sentez_cloud_v1_2.pdf`](paper/sentez_cloud_v1_2.pdf)  
> **Status:** Architectural Position Paper & Specification (No unmeasured performance claims; prototype roadmap included).

---

## Overview

**Sentez Cloud** is a pragmatic, cryptography-light cloud security architecture designed for today's production cloud infrastructure (**AWS Nitro Enclaves**, **AMD SEV-SNP**, **Intel TDX**, **SmartNICs/DPUs**, **Firecracker**, and **`vsock`**).

Rather than marketing an unrealistic *"virtual air-gap,"* Sentez Cloud applies privilege separation (inspired by Qubes OS and OpenSSH) together with an explicit **antinomy -> domain separation -> residual boundary (*Schranke*)** engineering checklist. Every defensive boundary explicitly documents what it prevents and what residual risk remains.

Advanced cryptographic modules—such as Offline Zero-Knowledge provers or [Evolutionary Proofs (`v2.2`)](https://github.com/Sidistr/Arcanum)—are intentionally decoupled as **strictly optional add-ons** so enterprises and cloud teams can adopt the core isolation architecture immediately.

---

## High-Level Architecture (C0–C3)

```text
+-----------------------------------------------------------------------------+
|  C0: UNTRUSTED HOST (Loader & Encrypted State Blob Carrier — No Authority)  |
|                                                                             |
|  +-----------------------+    +---------------------+    +---------------+  |
|  |  C1: ONLINE EDGE      |    |  C2: SENTEZ BRIDGE  |    |  C3: OFFLINE  |  |
|  |  (SmartNIC / DPU /    |    |  (Rust vsock Gate)  |    |  CORE / VAULT |  |
|  |   Firecracker VM)     |===>|  - Strict Schema    |===>|  - Nitro Enc. |  |
|  |  - Hostile Network    |    |  - Fixed Framing    |    |  - SEV-SNP /  |  |
|  |  - Live Telemetry     |    |  - Optional Pacing  |    |    Intel TDX  |  |
|  |  - Zero Root Secrets  |    |  - No Signing Keys  |    |  - No Net NIC |  |
|  +-----------------------+    +---------------------+    +-------+-------+  |
|              ^                   (Modeled as Dolev-Yao           |          |
|              |                    if host compromised)           |          |
+--------------|---------------------------------------------------|----------+
               |                                                   |
   [ Remote Attestation: PCR / Launch Measurement ]                |
               |                                                   v
   +-----------+---------------------------------------------------+----------+
   |  EXTERNAL GOVERNANCE & FRESHNESS PLANE (Profiles 2–3)                    |
   |  - Cold Root (Air-Gapped Offline PK): Key rotation & Roughtime allowlist |
   |  - Hot Operational Signers (Ed25519 / FROST / CoSi): Routine checkpoints |
   |  - External Witness Quorum & IETF Roughtime (RFC 10049):                 |
   |    Prevents rollback & host clock skew -> enforces FAIL-CLOSED on expiry |
   +--------------------------------------------------------------------------+
```

---

## Core Design Pillars

1. **Four-Compartment Split (`C0–C3`):** Separates the untrusted host loader (`C0`), hostile-network edge (`C1`), strict Rust `vsock` schema parser (`C2`), and networkless core/telemetry vault (`C3`). Remote attestation (PCRs / launch measurements)—not `C0`—establishes image authenticity.
2. **Two-Speed Governance & Freshness Lease:** Pairs an air-gapped **Cold Root (`Offline PK`)** with short-lived hot signers. `C3` enforces a bounded freshness window ($\Delta T + T_{\text{chk}}$) and transitions deterministically to **`FAIL-CLOSED`** upon lease expiry or enclave reboot.
3. **Explicit Residual-Risk Boundaries (*Schranken*):**

| Boundary | Threat Class | Architectural Mitigation | Residual Risk (*Schranke*) |
| :--- | :--- | :--- | :--- |
| **Boundary 1** | Diskless Enclave / Host-Disk Rollback | External witness anchors & quorum checkpoints (Profiles 2–3) | Accepted residual risk in Profile 1 (single-cloud starter); liveness dependency on witnesses in Profiles 2–3 |
| **Boundary 2** | Single-Cloud / Single-Silicon / Single-CA Lock-in | Heterogeneous $k$-of-$n$ diversity across AWS Nitro, AMD SEV-SNP, and Intel TDX (Profile 3) | Multi-cloud operational complexity, cross-region latency, and shared software/supply-chain bugs |
| **Boundary 3** | `vsock` Covert Channels & Compromised `C2` | Fixed-size framing, optional constant-rate pacing, end-to-end signature verification inside `C3` | Compromised `C2` can still drop unsigned telemetry (DoS) or disable host-side pacing |
| **Boundary 4** | Untrusted Host Clock / Lease Stretching | AMD SEV-SNP Secure TSC (where available), operation-budget leases, and `RFC 10049` Roughtime pinned to `C3` / Cold Root allow-list | Network partition or Roughtime quorum unreachability triggers intentional `FAIL-CLOSED` halt |

---

## Graduated Deployment Profiles

| Profile | Target Environment | Isolation & Time/State Guarantees |
| :--- | :--- | :--- |
| **Profile 1: Single-Cloud Starter** | Fast adoption on AWS Nitro or single-cloud SEV-SNP | `C1–C2–C3` split + local freshness lease + live tamper containment (host rollback explicitly accepted as residual risk) |
| **Profile 2: Rollback-Resistant Enterprise** | Regulated workloads, audit & telemetry vaults | Adds external witness anchors, Cold Root `Offline PK`, and pinned `RFC 10049` Roughtime quorum |
| **Profile 3: Sovereign Multi-Cloud** | High-assurance critical infrastructure | Heterogeneous $k$-of-$n$ deployment across AWS Nitro, AMD SEV-SNP, and Intel TDX |

---

## Prototype Roadmap (Design Targets)

These targets define acceptance criteria for future reference implementations (not benchmark claims):

- **Milestone 1 (`C2` Rust `vsock` Bridge):** Memory-safe schema validator and framing gate; `72-hour` continuous fuzzing campaign ($\ge 10^9$ mutated inputs) with zero parser crashes; `< 1 ms` p99 framing overhead (pacing disabled).
- **Milestone 2 (Freshness Lease & Offline Vault):** End-to-end `C3` checkpoint verification `< 50 ms` (assuming Ed25519 / FROST or CoSi multi-signatures; post-quantum ML-DSA budgets scale accordingly) and deterministic `FAIL-CLOSED` enforcement under simulated omission and reboot tests.
- **Milestone 3 (Multi-Cloud & Roughtime Integration):** Cross-provider witness quorum (`< 500 ms` p95 checkpoint commitment) with Cold-Root-pinned `RFC 10049` Roughtime verification.

---

## Repository Structure

```text
sentez-cloud/
├── paper/
│   └── sentez_cloud_v1_2.pdf   # Full position paper (v1.2, 22 Nov 2026)
├── LICENSE                     # Dual License (CC BY 4.0 for docs / MIT for code)
└── README.md                   # Architecture overview & roadmap
```

---

## Related Work (Arcanum Research Line)

Sentez Cloud is designed to operate standalone, while cleanly composing with companion works in the [Arcanum repository](https://github.com/Sidistr/Arcanum):
- **Evolutionary Proofs (`v2.2`):** Ratcheted cryptographic proof-system evolution and dual-sandbox verification.
- **Biomimetic Transport Layer (`v2.2`):** Multi-path AONT-RS and adaptive routing for censorship-resistant large payloads.
- **AG-ZKP (`v1.3`):** Anamorphic-resistant zero-knowledge sanitization.

---

## Citation

If you reference Sentez Cloud in technical or academic work, please cite:

```bibtex
@misc{atakan2026sentezcloud,
  author       = {Atakan, Mert},
  title        = {Sentez Cloud (v1.2): Antinomy-Driven Cloud Compartmentalization, Offline Telemetry Vaults, and Explicit Residual-Risk Boundaries},
  month        = nov,
  year         = {2026},
  version      = {v1.2},
  publisher    = {Zenodo},
  doi          = {10.5281/zenodo.[XXXXXXX](https://doi.org/10.5281/zenodo.24447579)},
  url          = {[https://github.com/Sidistr/sentez-cloud](https://github.com/Sidistr/sentez-cloud)}
}
```

---

## AI-Assistance & Authorship Disclosure

This work is an independent architectural position paper conceived, directed, and reviewed by **Mert Atakan**. Large language models (**Google Gemini** and **Anthropic Claude**) were utilized as editorial, formal-consistency, and adversarial stress-testing assistants during iterative drafting (`v1.0` through `v1.2`). All claims, boundaries, and final text have been reviewed and approved by the author.

## License & Contact

- **Documentation & Paper:** [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
- **Reference Specifications, Schemas & Future Code:** [MIT License](https://opensource.org/licenses/MIT)
- **Contact:** `mert@arcanumlayer.com`

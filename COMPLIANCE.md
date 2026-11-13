# KALEOS: Aerospace & Avionics Standards Compliance Specification
### Implementation Scope, Protocol Mapping & Deterministic Silicon Rigor (CCSDS, ECSS, NASA cFS, DO-178C)

**Platform Target:** AArch64 / BCM2711 Bare-Metal Microkernel  
**Subsystem Crate:** `kaleos_userspace/libs/ccsds` (`#![no_std]`, Zero-Allocation Architecture)  
**Standard Status:** Production Protocol Implementation & Formal Mathematical Verification  
**Audience:** Avionics Systems Engineers, Space Mission Reviewers & Embedded Safety Auditors.

---

## 1. Executive Summary: Space-Grade Microkernel Design

Modern CubeSat, SmallSat, and defense avionics require deterministic telemetry handling, fault isolation, and standardized space packet interfaces. Rather than treating spaceflight protocols as bloated third-party userland daemons, KALEOS provides an integrated, zero-allocation space communications stack (`libs/ccsds`) built natively in safe `no_std` Rust.

All implementations strictly adhere to:
* **Zero Dynamic Allocation on Telemetry Hotpaths:** Packet encoding, decoding, and routing operate with zero heap overhead to prevent memory fragmentation.
* **Bounded Execution Time:** Loop unrolling and static bounds guarantee worst-case execution time (WCET) predictability.
* **100% Integer Arithmetic:** Zero floating-point unit (FPU) usage across the entire avionic stack.

---

## 2. Standards Compliance Matrix

| Standard Authority | Protocol / Specification | Implementation Scope in KALEOS | Compliance Level |
| :--- | :--- | :--- | :---: |
| **CCSDS** | **CCSDS 133.0-B-1 / B-2** (Space Packet Protocol) | Primary Header, APID, Sequence Flags, Packet Framing | **Fully Implemented** |
| **CCSDS / ECSS** | **CRC-16-CCITT** (Polynomial: `0x1021`) | Frame & Packet Verification (Init: `0xFFFF`, Final: `0x0000`) | **Fully Implemented** |
| **ESA** | **ECSS-E-ST-70-41C** (Packet Utilization Standard) | PUS-1 (Verification), PUS-3 (Housekeeping), PUS-5 (Events) | **Implemented Subset** |
| **NASA** | **NASA core Flight System (cFS / cFE)** | Message ID (MID) Mapping, Software Bus Bridge Routing | **Interface Bridge** |
| **RTCA / EUROCAE**| **DO-178C / ED-12C** (Airborne Software Safety) | Deterministic memory, zero recursion, formal SAT/SMT proofs | **Architecture Aligned** |

---

## 3. Detailed Technical Implementations

### 3.1 CCSDS Space Packet Protocol (CCSDS 133.0-B-1 / B-2)
Implemented in `kaleos_userspace/libs/ccsds/src/primary_header.rs` and `packet.rs`:

* **Standard 6-Byte Primary Header:**
  - **Packet Version Number:** Fixed 3-bit field (`0b000`).
  - **Packet Type:** 1-bit indicator (`0` = Telemetry TM, `1` = Telecommand TC).
  - **Secondary Header Flag:** 1-bit indicator for ECSS PUS time and service metadata.
  - **Application Process ID (APID):** 11-bit identifier (`0x000` to `0x7FF`, with `0x7FF` reserved as Idle Packet per standard).
  - **Sequence Flags:** 2-bit segmentation control (`0b11` = Unsegmented standalone packet).
  - **Packet Sequence Count:** 14-bit monotonically increasing counter (`0` to `16,383` with modular wrap-around).
  - **Packet Data Length:** 16-bit payload length representation (specifies octets in data field minus 1).
* **Stream Synchronization & Framing (`framing.rs`):**
  - Incorporates the CCSDS Attached Synchronization Marker (ASM): `0x1ACFFC1D` for telemetry stream synchronization and frame delineation.

### 3.2 Error Detection: CCSDS CRC-16 Engine
Implemented in `kaleos_userspace/libs/ccsds/src/crc16.rs`:
* **Generator Polynomial:** $x^{16} + x^{12} + x^5 + 1$ (`0x1021`).
* **Initial Remainder:** `0xFFFF`.
* **Standard Test Vector:** Verified against the standard ASCII test sequence (`"123456789"` yields `0x29B1`).
* **Performance:** Bitwise and table-driven options with zero allocations, formally verified against arithmetic overflow via Kani.

### 3.3 ESA ECSS-E-ST-70-41C (Packet Utilization Standard)
Implemented in `kaleos_userspace/libs/ccsds/src/pus.rs`, `services.rs`, and `router.rs`:

* **Standard PUS Secondary Header:**
  - PUS Version: Type-C 4-bit standard.
  - Service Type & Service Subtype: Deterministic byte dispatching.
  - Absolute Time Tag: Deterministic 64-bit microsecond counter (`CNTVCT_EL0` derived).
* **Implemented PUS Services:**
  1. **PUS Service 1 (Request Verification):**
     - Subtype 1: Telecommand Acceptance Success (`pus1_accept_success`).
     - Subtype 2: Telecommand Acceptance Failure with standardized rejection error codes (`pus1_accept_failure`).
     - Subtype 7: Telecommand Execution Completion (`pus1_exec_success`).
  2. **PUS Service 3 (Housekeeping & Diagnostic Data Reporting):**
     - Subtype 25: Periodic housekeeping parameter collection (system voltages, temperatures, core loads, memory quotas).
  3. **PUS Service 5 (Event Reporting):**
     - Subtype 1: Informative Event Report.
     - Subtype 4: Critical High-Severity Alarm Report.

### 3.4 NASA Core Flight System (cFS) Software Bus Bridge
Implemented in `kaleos_userspace/libs/ccsds/src/cfs.rs`:
* Translates standard NASA cFS Command and Telemetry Message IDs (MIDs) to native KALEOS EL0 SPSC IPC tokens.
* Supports standard cFE core subsystem IDs:
  - `CFE_ES_CMD_MID` (`0x1806`), `CFE_ES_HK_TLM_MID` (`0x0800` - Executive Services).
  - `CFE_EVS_CMD_MID` (`0x1801` - Event Services).
  - `CFE_TIME_CMD_MID` (`0x1805` - Time Services).
  - `CFE_SB_CMD_MID` (`0x1803` - Software Bus Services).

---

## 4. RTCA DO-178C / ED-12C Design Alignment

While formal commercial certification is conducted at the whole-system integrated level, KALEOS is engineered to satisfy the most stringent DO-178C Level A objectives at the microkernel architecture level:

1. **Deterministic Bounded Execution:** All kernel primitives have finite execution bounds with zero unbounded while loops.
2. **Zero Dynamic Allocation on Supervisor Hotpaths:** EL1 kernel memory structures are pre-allocated at boot, preventing out-of-memory states during flight operations.
3. **Absence of Undefined Behavior (Formal SAT/SMT Proofs):** 279 mathematical proof harnesses formally verify pointer bounds, memory alignment, and absence of panics.
4. **Hardware Memory Partitioning:** ARMv8-A MMU enforce strict process boundary isolation with $W \oplus X$ (Data Execution Prevention) page permissions.

---

<div align="center">
<sub>© 2026 Mekserina Karaosmanoğlu. All rights reserved. KALEOS is a registered sovereign microkernel architecture.</sub>
</div>

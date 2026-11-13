# KALEOS: Peer Review & Technical Evaluation Guide
### Architectural Defensibility, Empirical Hardware Facts & Independent Auditor Q&A

**Target Silicon:** Raspberry Pi 4 Model B (Broadcom BCM2711 / Quad-Core Cortex-A72 @ 1.50 GHz)  
**Target Audience:** OS Kernel Engineers, Academic Referees, Defense/Aerospace Auditors, and Systems Architects.

---

## 1. Executive Auditor Overview

This guide provides direct, empirical answers to the most rigorous architectural questions regarding KALEOS. There are no marketing exaggerations: all performance numbers, safety models, and protocol features cite specific source crates and live hardware UART telemetries.

---

## 2. Core Architectural Questions & Verifiable Facts

### Q1: "seL4 already exists. Why engineer an independent pure microkernel in Rust?"
* **Implementation Language:** seL4 is implemented in C99, requiring formal interactive proofs in Isabelle/HOL (a decades-long manual process exceeding 200,000 lines of proof scripts). KALEOS is engineered from scratch in freestanding `no_std` Rust, combining compiler-enforced affine type systems (ownership/borrowing) with automated SAT/SMT formal verification (Amazon Kani).
* **Verification Maintenance:** When C code changes, manual Isabelle/HOL proofs frequently break and require specialized mathematicians to re-prove. KALEOS utilizes automated bounded model checking and mathematical induction that executes within standard CI/CD pipelines.
* **Integrated Subsystem Scope:** seL4 intentionally leaves device drivers, window managers, and spaceflight protocol stacks out of scope. KALEOS delivers an end-to-end verified platform including an avionic space communications stack (`libs/ccsds`), an integer-based glassmorphic compositor (`kale_shell`), and an isolated Linux ABI sandbox (`vlinux_service`).

---

### Q2: "How does KALEOS prevent system crashes like the July 2024 CrowdStrike disaster?"
* **Root Cause of the Disaster:** Monolithic kernels (Windows, Linux) execute third-party drivers, endpoint sensors, and security agents inside supervisor mode (Ring 0 / EL1). A single invalid memory dereference crashes the entire OS (*Kernel Panic* / *BSOD*).
* **The KALEOS Defense:**
  1. **Strict EL0 Isolation:** All storage drivers (SDHCI, Ext2, FAT32), network stacks (GENET v5, smoltcp), audio controllers (PWM1), and security tools execute exclusively as unprivileged userland processes (EL0).
  2. **Micro-Reaper & Quiescence:** If an isolated driver experiences a segmentation fault or divide-by-zero, the ARM64 MMU traps the fault directly to the EL1 microkernel. The kernel revokes the process's tokens, reclaims its page tables, and re-spawns the service in microseconds without stopping the kernel or resetting other cores.

---

### Q3: "What are the exact boundaries of the Linux compatibility layer (`vlinux_service` - PID 11)?"
To prevent buzzword inflation, KALEOS explicitly documents the scope and conscious boundaries of its Linux ABI:

| Feature | Supported? | Technical Explanation |
| :--- | :---: | :--- |
| **Static ARM64 ELF Binaries** | **YES** | Loads standard AArch64 statically-linked executables. |
| **Busybox & Coreutils Applets**| **YES** | Executes `ls`, `cat`, `echo`, `cal`, `grep`, `wc`, `uname`. |
| **Standard Command Pipelines** | **YES** | Supports shell pipe operators (`|`, `&&`, `||`, `;`) via Citadel GUI. |
| **Anonymous & File `mmap`** | **YES** | Maps virtual memory objects (VMOs) with capability checks. |
| **Dynamic Linking (`.so`)** | **NO** | Dynamic linker (`ld-linux-aarch64.so.1`) is out of scope for Phase 1. |
| **`fork()` System Call** | **NO** | Copy-on-Write address space cloning violates KALEOS capability derivation invariants. Process launching uses explicit `exec` semantics. |
| **Complex Asynchronous Signals**| **NO** | Synchronous faults (SIGSEGV, SIGBUS) terminate cleanly; async `sigaction` queuing is intentionally excluded. |

---

### Q4: "Why is the kernel 100% FPU-Free, and what is gained?"
* **Elimination of Register Bloat:** On ARMv8-A, saving and restoring the 32 full 128-bit NEON/VFP SIMD registers (`q0`–`q31`) adds 512 bytes of memory copying and multiple pipeline stalls on every exception entry and exit.
* **Deterministic Sub-Microsecond Switches:** By strictly forbidding floating-point instructions in the EL1 kernel, KALEOS utilizes a minimal 288-byte integer-only context save (`x0`–`x30`, `SP`, `ELR`, `SPSR`).
* **Microsecond Context Switch:** This architectural discipline is why one-way cross-domain IPC latency executes in just **1,028 CPU cycles (~685 ns)** on physical Cortex-A72 silicon. All analytical graphics, audio synthesis, and telemetry use 128-bit integer fixed-point math.

---

### Q5: "How were the IPC latency metrics measured? Can they be trusted?"
* **Physical Hardware:** Broadcom BCM2711 SoC on a physical Raspberry Pi 4 Model B (Quad-Core Cortex-A72 @ 1.50 GHz, 8 GB RAM).
* **Hardware Counters:**
  1. `PMCCNTR_EL0`: CPU cycle counter running at processor clock (1.50 GHz = ~0.667 ns resolution). Read from EL0 via calibrated hardware authorization (`PMUSERENR_EL0`).
  2. `CNTVCT_EL0`: ARM Generic System Counter running at a constant 54.0 MHz (~18.52 ns resolution).
* **Two Independent Processes:** Measures real cross-domain RPC between Client (PID 15) and Echo Server (PID 14), including 2x `svc #0` syscalls, 2x 288-byte context saves, 2x capability authentications, and 2x MMU TTBR0 address space swaps.
* **Empirical Validation:**
  - Minimum round-trip: **1,925 cycles (~1.28 µs)**.
  - Median (p50): **2,056 cycles (~1.37 µs)**.
  - Under 256 KB continuous cache pollution / thrashing: **2,919 cycles (~1.94 µs)**.
  - Real console UART output is published verbatim in [`assets/uart_silicon_boot.log`](assets/uart_silicon_boot.log).

---

### Q6: "What does the 279-Kani formal verification actually prove?"
* 279 verification harnesses test the kernel using **Amazon Kani (SAT/SMT solver with CaDiCaL backend)**.
* **Guaranteed Properties:**
  1. **Absence of Panics:** No code path in the supervisor kernel can trigger `panic!`, `unwrap()`, or array out-of-bounds indexing.
  2. **Inductive Memory Bounds:** Lock-free Single-Producer Single-Consumer (SPSC) ring buffers and Buddy allocator block splitting remain valid across $2^{64}$ continuous cycles without pointer corruption.
  3. **Capability Revocation Monotonicity:** Revoking a parent capability token mathematically guarantees all child tokens are revoked without dangling authorization rights.
* Full breakdown by subsystem is cataloged in [`VERIFICATION.md`](VERIFICATION.md).

---

<div align="center">
<sub>© 2026 Mekserina Karaosmanoğlu. All rights reserved. KALEOS is a registered sovereign microkernel architecture.</sub>
</div>

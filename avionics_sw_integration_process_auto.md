# Software Integration Process
### CSC / CSCI Integration into Cards, Boxes, and System

**Document ID:** SIP-001
**Revision:** Draft A
**Status:** Working draft — tailoring required (see §0.3)

---

## 0. Front Matter

### 0.1 Purpose

This document defines the process, entry/exit criteria, and test scope for integrating software (CSUs → CSCs → CSCIs) onto mission cards, into multi-card boxes across a backplane, and finally into the system. It is written to be gate-driven: nothing moves to the next level of fidelity without a defined artifact package and a signed gate.

### 0.2 Scope

Applies to all mission software executing on mission cards in the system, including:

- Application software (sequencing, telemetry, FDIR, payload interface)
- Platform software (RTOS, BSP, board support, device drivers, bus stacks)
- Bootloaders, image loaders, and recovery/golden images
- FPGA/CPLD firmware where it implements logical behavior at a bus or protocol boundary
- Ground support software that participates in an integrated test path (EGSE, test conductor tools)

Explicitly out of scope: desktop analysis tooling with no mission path, and pure documentation deliverables.

### 0.3 Tailoring Assumptions

This draft assumes the following.

| Assumption | Value assumed |
|---|---|
| Software classification framework | MIL-STD-498 nomenclature (CSU/CSC/CSCI), NPR 7150.2 Class A/B software, DO-178C-flavored objectives where a design assurance argument is needed |
| Redundancy architecture | Dual- or triple-string mission, cross-strapped over 1553 and Ethernet |
| Backplane | Mixed-protocol: PCIe (data plane), 1553 (command/control), Ethernet (telemetry/bulk), RS-422/485 (point-to-point), I²C (EEPROM/thermal), GPIO discretes (arm, inhibit, sync, reset) |
| Environments | Vibration, shock, thermal, thermal-vacuum, EMI/EMC, radiation (SEE/TID) as applicable to  duration |
| CM tooling | Git-based SCM with signed tags, artifact registry with immutable digests, containerized build |

### 0.4 Definitions

| Term | Definition |
|---|---|
| **CSU** | Computer Software Unit. Smallest separately testable element — a module, class, or file. |
| **CSC** | Computer Software Component. A cohesive aggregation of CSUs with a defined internal interface. Not independently deployable. |
| **CSCI** | Computer Software Configuration Item. The deployable, separately configuration-managed software item. One CSCI maps to one loadable image on one processing element (or one partition, if partitioned). |
| **HWCI** | Hardware Configuration Item. A card, chassis, backplane, or harness with its own part number and revision. |
| **Firmware CI** | FPGA/CPLD bitstream, configuration-managed as its own CI with its own version, but bound to a card revision. |
| **Card** | An SRU/CCA seated in the box backplane. Carries zero or more processing elements. |
| **Box** | The LRU chassis: backplane + cards + power + connectors. |
| **String** | A complete redundant processing chain from sensing through effecting. |
| **ICD** | Interface Control Document. Governs electrical, protocol, timing, and message-level behavior at a named interface. |
| **VDD** | Version Description Document. The record of what is in a delivered build. |
| **Nominal test** | Verification against the requirement as written, under expected conditions. |
| **Off-nominal test** | Verification of behavior when an input, interface, timing, or resource assumption is violated. |
| **FDIR** | Fault Detection, Isolation, and Recovery. |

### 0.5 Hierarchy and Ownership

```
System
└── String A / String B / String C
    └── Box (LRU)            ← Box IPT owns integration
        ├── Backplane (HWCI) ← defines the bus fabric
        └── Card (HWCI)      ← Card IPT owns integration
            ├── Firmware CI  ← FPGA/CPLD team
            └── CSCI         ← Software IPT owns
                └── CSC
                    └── CSU
```

**Rule:** A CSCI is owned by exactly one software lead. A card is owned by exactly one card lead. Where a card hosts multiple CSCIs (multi-core, partitioned, or multiple processing elements), the card lead owns the *composition* and the software leads own their items. The composition is itself configuration-managed as a **Card Software Load Set (CSLS)**.

---

## 1. Levels of Fidelity

Integration proceeds through defined levels. Each level exists to find a specific class of defect as early and as cheaply as possible. **Do not skip levels to recover schedule** — a defect found at L5 costs 10–100× what it costs at L1, and a defect found at L6 may slip the mission.

| Level | Name | Compute | Bus/IO | Environment | Primary defect class caught |
|---|---|---|---|---|---|
| **L0** | Developer / Unit | Host (x86/ARM dev machine) | None — stubs | Desk | Logic, algorithm, boundary conditions |
| **L1** | SIL: Software-in-the-Loop | Host, native or cross-compiled to emulator | Simulated HAL, software bus models | Desk / CI farm | CSC-to-CSC interface defects, state machine errors, requirement misreads |
| **L2** | PIL: Processor-in-the-Loop | Target silicon on eval board or mission card, no peripherals | Bus simulators + breakout harness | Bench | Compiler/toolchain issues, endianness, alignment, timing, stack, WCET |
| **L3** | Card Integration | Mission-like card (EDU or mission spare) | Real card PHYs into bus emulators / analyzers | Bench, ESD-controlled | Driver/firmware interaction, electrical protocol, register maps, init sequences |
| **L4** | Box Integration | All cards in real chassis on real backplane | Real backplane, real intra-box traffic | Bench + thermal soak | Backplane contention, arbitration, power sequencing, cross-card timing, EMI-on-backplane |
| **L5** | System Integration Lab (SIL/ITB) | Multiple boxes, mission-representative harness | Real inter-box buses + HWIL system dynamics sim | Lab, closed-loop | Cross-box timing, redundancy management, mission sequencing, FDIR at system scope |
| **L6** | System Integration | Mission hardware, as-built system | mission harness, mission connectors | Integration facility | As-built vs as-designed deltas, harness/routing/grounding, real sensor and effector behavior |

### 1.1 Fidelity Gate Principle

A requirement is **verified** at the lowest level of fidelity where the verification is *credible*. Each requirement in the SRS/IRS carries a **Verification Level** attribute (L0–L6) and a **Verification Method** (Inspection, Analysis, Demonstration, Test). Verification at a level lower than the assigned level is *evidence*, not *closure*.

**Re-verification rule:** When a CSCI changes, verification credit at level *N* is retained only if the change is shown by impact analysis not to affect behavior verified at level *N*. Otherwise the affected test cases re-run. See §9.

---

## 2. Configuration Management and Delivery Artifacts

This section defines what "the code is ready" actually means in artifact terms. §5 (the Hardware
Integration Readiness gate) consumes everything defined here.

### 2.1 Repository and Branching

| Item | Convention |
|---|---|
| Repository naming | `sw-<subsystem>-<cscl-name>` e.g. `sw-msn-msnproc`, `sw-plat-bsp-sbc3u` |
| Trunk | `main` — always buildable, always passes L0+L1 gates |
| Integration branch | `integ/<box>-<campaign>` for time-boxed integration campaigns |
| Release branch | `rel/<major>.<minor>` — cut at the CSCI qualification gate, patch-only afterward |
| Tag format (CSCI) | `csci/<name>/v<MAJOR>.<MINOR>.<PATCH>[-rc<N>]` — **annotated and GPG-signed** |
| Tag format (firmware) | `fpga/<name>/v<MAJOR>.<MINOR>.<PATCH>` |
| Tag format (load set) | `csls/<card>/v<MAJOR>.<MINOR>.<PATCH>` |
| Protected refs | Tags matching `csci/**`, `fpga/**`, `csls/**` are immutable and non-deletable. Force-push disabled on `main` and `rel/**`. |

**Commit SHA is the only true identifier.** Tags are for humans and for gates; every artifact record carries the full 40-character SHA.

### 2.2 Version Semantics

`MAJOR.MINOR.PATCH` where:

- **MAJOR** — breaks an external interface: ICD message change, wire format, register map, command set, or behavioral contract. Requires re-verification of all peer interfaces.
- **MINOR** — adds capability without breaking an external interface. Peers remain compatible.   Requires regression of the CSCI plus interface smoke tests at the peer boundary.
- **PATCH** — defect correction, no interface change, no new capability. Requires targeted regression
  plus the L3/L4 smoke suite.

Build metadata appended at artifact level, not tag level: `v2.4.1+sha.a1b2c3d.b1842` (version + short SHA + monotonic build number).

Pre-release suffix `-rc<N>` is permitted up to and including L4. **`-rc` builds are prohibited at
L5 and above.** Anything that closes a verification at L5/L6 carries a clean release version.

### 2.3 Reproducible Build Requirement

Every CSCI build entering L2 or above must be reproducible:

- Toolchain delivered as a container image referenced by **digest**, not tag  (`registry.internal/toolchain/arm-eabi@sha256:...`)
- All compiler, linker, and assembler flags captured in the build record
- Two independent builds from the same SHA on two different build agents must produce  **byte-identical** primary artifacts (or a documented, justified list of non-deterministic sections, e.g. a build-timestamp field in a header)
- Build record stored alongside the artifact in the registry, immutably

If a build is not reproducible, it cannot be root-caused after an anomaly. This is not negotiable for mission software.

### 2.4 Delivery Readiness Package (DRP)

Each CSCI delivery across a gate is a DRP. The DRP has a machine-readable manifest and an attached evidence set.

#### 2.4.1 Manifest — required fields

```yaml
# drp-manifest.yaml
schema_version: "1.0"

csci:
  name: "msnproc"
  long_name: "Mission Processor Software"
  version: "2.4.1"
  criticality: "Class A"              # NPR 7150.2 class
  partition_id: 3                     # if partitioned OS

provenance:
  repository: "ssh://git@scm.internal/msnproc.git"
  commit_sha: "a1b2c3d4e5f60718293a4b5c6d7e8f9012345678"
  tag: "csci/msnproc/v2.4.1"
  tag_signature: "verified:key-id=0xDEADBEEF"
  branch: "rel/2.4"
  build_number: 1842
  build_timestamp_utc: "2026-09-02T14:22:07Z"
  build_agent: "bld-07"
  reproducibility:
    independent_rebuild_agent: "bld-12"
    artifacts_byte_identical: true
    nondeterministic_sections: []

toolchain:
  container_digest: "sha256:9f8e7d6c5b4a39281706f5e4d3c2b1a09876543210fedcba9876543210abcdef"
  compiler: "arm-none-eabi-gcc 12.3.0"
  flags: "-O2 -ffreestanding -fno-common -Wall -Werror -mcpu=cortex-r5 -mfpu=vfpv3-d16"
  linker_script: "ld/msnproc_flash.ld@a1b2c3d"

artifacts:
  - name: "msnproc.elf"
    sha256: "3b1f...c9"
    size_bytes: 1843204
  - name: "msnproc.bin"
    sha256: "77aa...41"
    size_bytes: 982016
    load_address: "0x08020000"
  - name: "msnproc.map"
    sha256: "12cd...a0"

resource_budget:
  flash_used_bytes: 982016
  flash_allocated_bytes: 1048576
  flash_margin_pct: 6.3
  ram_static_bytes: 214400
  ram_allocated_bytes: 262144
  ram_margin_pct: 18.2
  worst_case_stack_bytes: 11840          # by static analysis, per task
  stack_allocated_bytes: 16384
  heap_used: "none — dynamic allocation prohibited post-init"
  cpu_utilization_worst_case_pct: 61.4
  cpu_budget_pct: 70

interfaces:
  - icd: "ICD-1553"
    revision: "D"
    message_db_sha256: "aa19...7f"
    role: "RT"
    rt_address: 12
  - icd: "ICD-ETH-TLM"
    revision: "C"
    schema_sha256: "b402...31"
    role: "publisher"
  - icd: "ICD-DISC-SAFING"
    revision: "B"
    signals: ["ARM_CMD_IN", "INHIBIT_STATUS_OUT", "SYNC_1PPS_IN"]

compatibility:
  card_part_number: "PN-40012"
  card_revisions_supported: ["Rev C", "Rev D"]
  card_revisions_excluded:
    - revision: "Rev B"
      reason: "I2C pull-up value incompatible with driver timing; see DR-0912"
  firmware:
    name: "msnproc_fpga"
    versions_supported: [">=3.2.0", "<4.0.0"]
  bootloader:
    versions_supported: [">=1.8.2"]
  backplane_revision: ["BP-2200 Rev E"]
  harness_drawing: ["HRN-5541 Rev G"]
  peer_csci:
    - name: "msnproc"
      versions_supported: ["^5.1.0"]
      relationship: "1553 BC — this CSCI is an RT"
    - name: "tlmproc"
      versions_supported: ["^2.0.0"]
      relationship: "Ethernet telemetry consumer"

third_party:
  - name: "VxWorks"                 # or RTEMS / Zephyr / bare-metal
    version: "7 SR0680"
    license: "commercial"
    qualification_evidence: "QUAL-RTOS-004"
  - name: "lwIP"
    version: "2.2.0"
    license: "BSD-3-Clause"
    modifications: "patches/lwip-2.2.0-arp-timeout.patch"
sbom:
  format: "CycloneDX 1.5"
  file: "msnproc-2.4.1.cdx.json"
  sha256: "5d3c...9e"

verification:
  requirements_baseline: "SRS-MSNPROC Rev F"
  rtm_file: "RTM-MSNPROC-2.4.1.csv"
  requirements_total: 412
  requirements_verified_L0_L1: 338
  requirements_allocated_L3_plus: 74
  coverage:
    statement_pct: 100.0
    branch_pct: 98.7
    mcdc_pct: 97.2                   # safety-critical CSCs only
    uncovered_justification: "COV-JUST-MSNPROC-2.4.1.md"
  static_analysis:
    ruleset: "MISRA C:2012 + CERT C"
    violations_open: 0
    violations_deviated: 7
    deviation_record: "DEV-MSNPROC-014"
  wcet_analysis: "WCET-MSNPROC-2.4.1.pdf"

defects:
  open_sev1: 0
  open_sev2: 0
  open_sev3: 4
  open_sev3_dispositions: "DR-0931, DR-0944, DR-0951, DR-0958 — all deferred with rationale"
  closed_since_last_delivery: 23

change_summary:
  previous_version: "2.4.0"
  changes:
    - id: "DR-0918"
      type: "defect"
      description: "1553 RT status word Busy bit cleared one frame late under sync loss"
      impact_level: "L3, L4"
    - id: "CR-0221"
      type: "change"
      description: "Added EDAC scrub rate telemetry point"
      impact_level: "L1"
  regression_scope: "REG-MSNPROC-2.4.1.md"

approvals:
  software_lead: { name: "", date: "", signature: "" }
  sqa: { name: "", date: "", signature: "" }
  cm: { name: "", date: "", signature: "" }
```

#### 2.4.2 DRP Evidence Set

Attached to the manifest, each item with its own SHA-256:

1. Version Description Document (VDD) narrative of §2.4.1
2. Requirements Traceability Matrix (bidirectional: requirement ↔ design ↔ code ↔ test ↔ result)
3. Unit and integration test reports with pass/fail per case
4. Coverage report with justification for every uncovered construct
5. Static analysis report and deviation records
6. Stack depth and WCET analysis
7. SBOM and license clearance
8. Open defect list with severity and disposition
9. Impact analysis and regression scope for this delivery
10. Installation/load procedure, including the recovery path
11. Known limitations and restrictions on use

### 2.5 Compatibility Matrix

Maintained centrally per card, not per CSCI. This is the single source of truth for
"what can be loaded together."

| Card PN/Rev | Bootloader | Firmware | CSCI(s) | Backplane Rev | Harness Rev | Status | Verified At |
|---|---|---|---|---|---|---|---|
| PN-40012 Rev C | 1.8.2 | 3.2.1 | msnproc 2.4.1 | BP-2200 Rev E | HRN-5541 Rev G | Qualified | L5 |
| PN-40012 Rev D | 1.9.0 | 3.3.0 | msnproc 2.4.1 | BP-2200 Rev E | HRN-5541 Rev G | Under test | L4 |
| PN-40012 Rev B | 1.8.2 | 3.1.4 | msnproc 2.3.x | BP-2200 Rev D | HRN-5541 Rev F | Obsolete | L5 |
| PN-40088 Rev A | 2.0.1 | 1.4.0 | tlmproc 2.0.3 | BP-2200 Rev E | HRN-5541 Rev G | Qualified | L5 |

**Rules:**
- A combination not in this matrix has not been tested and shall not be powered in an integrated
  configuration without an approved deviation.
- Every row cites the highest level of fidelity at which the combination was exercised.
- The matrix is baselined at each box-level gate and is a configuration item in its own right.

### 2.6 Card Software Load Set (CSLS)

Where a card hosts multiple loadable items, the CSLS is the atomic deliverable to the card:

```yaml
csls:
  card_part_number: "PN-40012"
  card_revision: "Rev D"
  version: "1.6.0"
  members:
    - type: bootloader
      name: "sbl"
      version: "1.9.0"
      slot: "boot0"
    - type: firmware
      name: "msnproc_fpga"
      version: "3.3.0"
      slot: "cfg_flash_a"
    - type: csci
      name: "msnproc"
      version: "2.4.1"
      slot: "app_a"
    - type: csci
      name: "msnproc"
      version: "2.4.0"
      slot: "app_b"        # golden / fallback image
  load_order: ["sbl", "msnproc_fpga", "app_b", "app_a"]
  post_load_verification: "PROC-LOADVERIFY-40012 Rev C"
```

### 2.7 COTS Hardware and Vendor Stack Control

Where a card, its firmware, or its driver stack is COTS and is not verified by this process (see
§14.6 for PCIe), configuration control becomes the primary defense rather than a formality. The
failure mode is specific: **COTS parts change without the part number changing.** A vendor revision
can alter enumeration order, BAR sizing, link training behavior, default register state, or driver
API semantics, and nothing on the purchase order will say so.

**Controlled fields.** Every COTS item in a mission path carries all of the following in the
compatibility matrix and in the box configuration baseline:

| Field | Why it is controlled |
|---|---|
| Vendor name and vendor part number | Baseline identity |
| **Vendor hardware revision / dash / date code** | The field that changes silently; the one most often not recorded |
| Option ROM, firmware, or FPGA image version on the COTS card | Changes behavior independently of the hardware revision |
| Driver / stack version, exact | Mission-critical behavior depends on it and this program cannot patch it |
| OS and BSP version the driver is qualified against | Driver behavior is not portable across BSP versions |
| Qualification evidence reference | What was actually tested, at what level |
| Risk acceptance reference | Who accepted the residual, for what configuration |

**Example COTS matrix:**

| Item | Vendor PN | HW Rev | Card FW | Driver | OS/BSP | Qualified at | Status |
|---|---|---|---|---|---|---|---|
| PCIe data-plane card | VND-7710 | Rev C | 2.4.1 | 5.12.3 | RHEL 9.4 / BSP 3.1 | L4 | Approved |
| PCIe data-plane card | VND-7710 | Rev D | 2.5.0 | 5.13.0 | RHEL 9.4 / BSP 3.1 | L3 | Under test |
| PCIe data-plane card | VND-7710 | Rev B | 2.3.x | 5.11.x | — | — | Obsolete |

**Rules:**

1. A COTS revision not in the matrix has not been tested and shall not be used in an integrated
   configuration without an approved deviation. Same rule as §2.5, applied to parts nobody on this
   program designed.
2. **Incoming inspection records the revision.** A receiving process that checks the part number and
   not the revision defeats this entire subsection.
3. The driver and stack appear in the DRP `third_party` block with version, license, and a
   qualification evidence reference — they are not infrastructure, they are components.
4. The running versions are **asserted at startup and reported in telemetry**, so a version drift is
   detectable in the field and not only in a spreadsheet.
5. A vendor revision change is a **Class I change** (§15.3) when the item sits in a mission-critical
   path, regardless of what the vendor calls it in their release note.
6. Buy a lifetime quantity from one revision where the program can afford it. This is a
   configuration-control decision as much as a supply-chain one, and it is far cheaper than
   re-qualifying against a revision you did not choose.

---

## 3. Process Overview

```
 ┌────────────────────────────────────────────────────────────────────────┐
 │  STEP 1   CSU / CSC Development & Unit Verification            (L0)    │
 │             ↓ gate: CSC Verification Complete                          │
 │  STEP 2   CSC → CSCI Software Integration                      (L1)    │
 │             ↓ gate: CSCI Build Integration Complete                    │
 │  STEP 3   CSCI Verification in SIL                             (L1/L2) │
 │             ↓ gate: CSCI Qualification Review (CQR)                    │
 │  STEP 4   HARDWARE INTEGRATION READINESS REVIEW (HIRR)         ——      │
 │             ↓ gate: authorization to power mission-like HW             │
 │  STEP 5   Card-Level HW/SW Integration                         (L3)    │
 │             ↓ gate: Card Integration Complete (CIC)                    │
 │  STEP 6   Box / Backplane Integration                          (L4)    │
 │             ↓ gate: Box Integration Complete (BIC)                     │
 │  STEP 7   Box Qualification under Environments                 (L4+)   │
 │             ↓ gate: Box Qualification Complete (BQC)                   │
 │  STEP 8   System Integration Lab / Integrated Test Bed         (L5)    │
 │             ↓ gate: System Integration Complete (SIC)                  │
 │  STEP 9   Formal Qualification Test                            (L6)    │
 │             ↓ gate: Formal Qualification Test Complete (FQT)           │
 │  STEP 10  Mission Configuration Freeze & MRR                   ——      │
 └────────────────────────────────────────────────────────────────────────┘
```

Each step below uses a fixed template:

> **Objective** — what this step is for
> **Entry Criteria** — must all be true to start; a gate chair verifies
> **Activities** — what gets done
> **Test Scope: Nominal** — required nominal verification
> **Test Scope: Off-Nominal** — required fault/stress verification
> **Exit Criteria** — must all be true to close
> **Artifacts Produced**
> **Gate Authority** — who signs

---

## 4. STEP 1 — CSU / CSC Development & Unit Verification (L0)

### Objective
Establish that each software unit and component behaves per its detailed design, in isolation, before it is combined with anything else.

### Entry Criteria

| # | Criterion | Evidence |
|---|---|---|
| 1.E1 | SRS/IRS requirements allocated to this CSC are baselined | Requirements baseline ID and rev |
| 1.E2 | Software Design Document (SDD) for the CSC reviewed and approved | SDD rev, review minutes |
| 1.E3 | Interfaces the CSC depends on are specified (internal API or ICD) | Interface spec rev |
| 1.E4 | Coding standard and static analysis ruleset defined and enforced in CI | CI config in repo |
| 1.E5 | Repository created under CM control with branch protection applied | Repo URL, protection policy |
| 1.E6 | Unit test framework and coverage instrumentation operational in CI | Green pipeline on skeleton |
| 1.E7 | Criticality classification assigned (drives coverage and review depth) | Classification record |

### Activities
- Implement CSUs against the SDD
- Author unit tests concurrently (test-first preferred for safety-critical CSCs)
- Peer review every change; safety-critical CSCs require two reviewers, one independent of the author
- Run static analysis on every commit; treat new violations as build failures
- Maintain bidirectional traceability as code is written, not retroactively
- Measure and record worst-case stack usage per task and WCET for hard real-time paths

### Test Scope: Nominal
- Every CSU function exercised across its documented input domain
- Every requirement allocated to the CSC has at least one passing test
- Structural coverage per criticality:

| Criticality | Statement | Branch | MC/DC |
|---|---|---|---|
| Class A (loss of system) | 100% | 100% | 100% |
| Class B (loss of mission) | 100% | 100% | Not required |
| Class C | 100% | ≥90% | Not required |
| Class D | ≥90% | — | — |

Any shortfall requires a written, approved justification per uncovered construct — not a blanket waiver.

### Test Scope: Off-Nominal
- **Boundary and out-of-range inputs** on every external entry point: min, min−1, max, max+1, zero, negative, NaN/Inf for floats, empty and oversized buffers
- **Null/invalid pointer and handle** rejection
- **Numeric hazards**: divide by zero, integer overflow/underflow, signed/unsigned conversion, float denormal, accumulator saturation
- **State machine illegal transitions**: every event injected in every state, including states where the event is undefined; verify defined rejection behavior, not undefined behavior
- **Resource exhaustion**: queue full, buffer pool empty, semaphore timeout, task stack near-limit
- **Error propagation**: every error return path from a called function forced at least once (fault injection at the stub boundary)
- **Re-entrancy and concurrency**: for shared-state CSUs, verify under simulated preemption at defined interleavings; verify all critical sections are bounded
- **Initialization order**: behavior when called before init and after shutdown

### Exit Criteria

| # | Criterion | Evidence |
|---|---|---|
| 1.X1 | 100% of allocated requirements have passing unit tests | RTM extract |
| 1.X2 | Coverage targets met or every gap individually justified and approved | Coverage report + justification doc |
| 1.X3 | Zero open static analysis violations; all deviations recorded and approved | Static analysis report, deviation record |
| 1.X4 | All code peer-reviewed; review records linked to commits | Review records |
| 1.X5 | Worst-case stack and WCET analysis complete, within budget with margin | Analysis reports |
| 1.X6 | No dynamic memory allocation after init (or approved deviation) | Analysis / inspection record |
| 1.X7 | CSC tagged in SCM with reproducible build | Tag, SHA, build record |
| 1.X8 | Zero open Severity 1 or 2 defects against the CSC | Defect report |

### Artifacts Produced
CSC source at tagged SHA · unit test suite and results · coverage report · static analysis report ·
stack/WCET analysis · updated RTM rows · peer review records

### Gate Authority
Software Lead (CSC owner) + Independent reviewer. SQA audits the record for a sampled subset.

---

## 5. STEP 2 — CSC → CSCI Software Integration (L1)

### Objective
Combine verified CSCs into the deployable CSCI image and demonstrate that the internal interfaces
between them behave as designed. This is still software-only — no target hardware.

### Entry Criteria

| # | Criterion | Evidence |
|---|---|---|
| 2.E1 | All constituent CSCs have passed Step 1 | Step 1 gate records for each CSC |
| 2.E2 | CSCI architecture and internal interface definitions baselined | SDD rev |
| 2.E3 | Task/thread model defined: priorities, periods, deadlines, and the schedulability argument | Rate-monotonic or equivalent analysis |
| 2.E4 | Memory map defined and budgeted: flash, RAM, stack per task, no-execute regions | Linker script + memory budget sheet |
| 2.E5 | Third-party components (RTOS, stacks, middleware) version-pinned with qualification evidence | Component list, qual records |
| 2.E6 | Simulated HAL / bus models available and version-controlled | Sim model repo + tag |
| 2.E7 | Build produces a loadable image reproducibly | Two-agent identical build record |

### Activities
- Integrate CSCs incrementally — add one CSC at a time to an already-passing baseline, never all at once. Big-bang integration destroys the ability to localize a defect.
- Bring up RTOS, BSP shim, and the simulated HAL
- Exercise every internal interface at its contract boundary
- Profile: task execution times, CPU utilization under synthetic worst-case load, jitter
- Verify memory map: no overlap, no overflow, guard regions intact

### Test Scope: Nominal
- End-to-end functional threads through the CSCI: input → processing → output
- Mode/state transitions across the full CSCI state machine
- Startup, initialization, steady-state, and controlled shutdown sequences
- Timing: all periodic tasks meet deadlines under nominal load
- Telemetry and logging output content and rate correctness

### Test Scope: Off-Nominal
- **CSC interface contract violations**: each CSC fed malformed/out-of-contract data from its peer
- **Task overrun**: force a task to exceed its budget; verify overrun detection and the defined response (skip, degrade, or reset — per design, not by accident)
- **Priority inversion**: construct the inversion scenario; verify the inheritance/ceiling protocol actually engages
- **Deadlock and livelock**: exercise all lock-acquisition orderings; verify lock ordering discipline
- **Starvation**: flood a low-priority path and confirm high-priority deadlines hold
- **Resource exhaustion at CSCI scope**: message queues, buffer pools, file handles, socket descriptors
- **Watchdog**: force a hang in each task; verify watchdog fires within its window and the recorded reset cause is correct
- **Init failure**: fail each initialization step in turn; verify the CSCI either safes or reports — never proceeds on a failed init
- **Fault injection into simulated HAL**: every simulated device returns error, timeout, and garbage in turn

### Exit Criteria

| # | Criterion | Evidence |
|---|---|---|
| 2.X1 | All CSCs integrated; CSCI builds clean with zero warnings at the project warning level | Build log |
| 2.X2 | Image fits within flash/RAM budget with the mandated margin (recommend ≥20% at this stage) | Memory map, budget sheet |
| 2.X3 | CPU utilization worst-case within budget with margin | Profiling report |
| 2.X4 | Schedulability argument holds with measured, not assumed, execution times | Updated analysis |
| 2.X5 | All internal interface tests pass, nominal and off-nominal | Integration test report |
| 2.X6 | Watchdog and reset-cause reporting verified for every task | Test report |
| 2.X7 | Tagged CSCI candidate build with full provenance | Tag, SHA, build record |
| 2.X8 | Zero open Sev 1/2 | Defect report |

### Artifacts Produced
CSCI candidate image · integration test report · profiling and schedulability report · memory map · updated RTM

### Gate Authority
Software Lead + Software Architect.

---

## 6. STEP 3 — CSCI Verification in SIL (L1 / L2)

### Objective
Formally verify the CSCI against its SRS/IRS in a controlled software environment, and confirm it executes correctly on the *actual target processor* before it touches mission-like hardware.

This step has two parts. Part A (L1) is host-based formal verification. Part B (L2) is
processor-in-the-loop. **Part B is mandatory** — it is where toolchain, endianness, alignment, cache, and real timing defects surface, and it is far cheaper to find them here than on a card.

### Entry Criteria

| # | Criterion | Evidence |
|---|---|---|
| 3.E1 | Step 2 exit criteria met and gated | Step 2 record |
| 3.E2 | Software Test Plan and Test Procedures reviewed and approved | STP/STD rev, review minutes |
| 3.E3 | Every SRS/IRS requirement has an assigned verification method and level | RTM completeness check |
| 3.E4 | SIL environment configuration-managed and its fidelity documented, including known deviations from real hardware | SIL config record, fidelity statement |
| 3.E5 | Bus models validated against the ICD (the model is itself verified) | Model V&V record |
| 3.E6 | Target eval board or mission-like processing element available with known configuration | HW config record, serial number |
| 3.E7 | Cross-compiled image loads and runs on target | PIL smoke test |
| 3.E8 | SQA test witness scheduled for formal runs | Schedule |

### Activities
- Execute the formal test procedures with SQA witness on record runs
- Run the same functional suite on host (Part A) and on target (Part B); compare results — any divergence is a defect until proven otherwise
- Measure real execution times on target and compare to the L1 estimates; update the schedulability argument with measured target numbers
- Long-duration soak: run at least 24 hours (recommend 72) continuous; monitor for leaks, counter rollover, drift, and fragmentation

### Test Scope: Nominal
- Every requirement with verification level ≤ L2 executed and passed
- Full operational scenario walkthroughs across every mission phase the CSCI participates in
- Interface message set: every message in the ICD transmitted and received at rate, with correct encoding, scaling, and units
- Time management: clock discipline, time source acquisition, monotonic time correctness

### Test Scope: Off-Nominal
- **Full ICD violation sweep** per §14 — malformed, out-of-sequence, out-of-range, wrong length, bad checksum, stale, duplicate, and absent messages for each interface
- **Timing violations**: input arriving early, late, jittery, bursty, and not at all
- **Counter and time rollover**: force every counter and timestamp to its rollover boundary (32-bit ms counters, frame counters, sequence numbers, GPS week rollover)
- **Numerical robustness**: feed sensor data that drives filters/estimators toward divergence; verify covariance and residual limits engage
- **Mode confusion**: command a mode transition that is illegal from the current state; verify rejection and correct reporting
- **Command rejection**: invalid, unauthorized, out-of-sequence, and replayed commands
- **Soak anomalies**: memory leak detection, heap/pool fragmentation, log wraparound, telemetry buffer wraparound over 72 hours
- **Reset and recovery**: power-on reset, watchdog reset, commanded reset, and brownout reset — each verified for correct state recovery and correct reset-cause reporting
- **Degraded mode entry and exit**: verify every degraded/safe mode is reachable, correct, and exitable per design

### Exit Criteria

| # | Criterion | Evidence |
|---|---|---|
| 3.X1 | 100% of requirements with verification level ≤L2 verified and passed, witnessed | Formal test report, SQA signature |
| 3.X2 | Host vs target results reconciled; every divergence explained or fixed | Comparison record |
| 3.X3 | Measured target timing within budget; schedulability re-argued with measured data | Timing report |
| 3.X4 | Soak test ≥24 h with no unexplained anomaly, no resource growth trend | Soak report with resource plots |
| 3.X5 | Off-nominal suite complete; every fault produces the specified response, no undefined behavior | Off-nominal test report |
| 3.X6 | RTM shows no orphan requirements and no orphan tests | RTM audit |
| 3.X7 | CSCI qualified version tagged and released to the artifact registry | Tag, registry digest |
| 3.X8 | VDD published | VDD |

### Artifacts Produced
Qualified CSCI image + DRP (per §2.4) · formal test reports · soak report · timing report · VDD · updated compatibility matrix rows (candidate status)

### Gate Authority
**CSCI Qualification Review (CQR)**: Software Lead (chair), Systems Engineering, SQA, CM, Safety.

---

## 7. STEP 4 — Hardware Integration Readiness Review (HIRR)

### Objective
This is the gate the rest of the document exists to protect. HIRR authorizes applying power to mission-like hardware with this software load. It is a **joint software + hardware + safety** gate.

The failure mode this gate prevents: a card is damaged, a schedule is lost, or a defect is masked because software arrived at the bench without a defined, reproducible, compatible configuration.

### 7.1 Entry Criteria — Software Side

| # | Criterion | Required evidence |
|---|---|---|
| 4.E1 | CSCI has passed CQR (Step 3) | CQR record |
| 4.E2 | **Repository URL and full commit SHA** recorded | DRP manifest `provenance` block |
| 4.E3 | **Signed annotated tag** exists and resolves to that SHA; signature verified | Tag verification output |
| 4.E4 | **Version** assigned per §2.2, no `-rc` if targeting L5+ later in the campaign | Manifest |
| 4.E5 | **Build reproducible** — two-agent byte-identical rebuild demonstrated | Build records from both agents |
| 4.E6 | **Toolchain pinned by digest** | Manifest `toolchain` block |
| 4.E7 | **Binary artifacts with SHA-256 and load addresses** published to the registry | Registry digests |
| 4.E8 | **Memory budget** within allocation with margin; linker map attached | Manifest + map file |
| 4.E9 | **SBOM** complete; all third-party components identified, versioned, licensed, and qualified | CycloneDX/SPDX file |
| 4.E10 | **ICD compliance declared**: for each interface, ICD ID + revision + message DB hash | Manifest `interfaces` block |
| 4.E11 | **Compatibility declared**: card PN and supported revisions, firmware version range, bootloader version range, backplane rev, harness rev, peer CSCI version ranges | Manifest `compatibility` block |
| 4.E12 | **Peer CSCI availability confirmed** at compatible versions, or a documented stub/simulator substitute with fidelity statement | Peer DRP manifests or sim config |
| 4.E13 | **Defects**: zero open Sev 1/2; every open Sev 3 has a written disposition and an impact statement for this integration | Defect report |
| 4.E14 | **Impact analysis and regression scope** for this delivery vs the last integrated version | Impact analysis doc |
| 4.E15 | **Load and recovery procedure** written and dry-run in SIL, including the path back to a known-good image if the load bricks the card | Approved procedure, dry-run record |
| 4.E16 | **Known-good fallback image** identified, available, and its load path verified | Fallback image digest + procedure |
| 4.E17 | **Debug/test hooks inventoried**: every hook, backdoor, test mode, and simulation flag listed, with its state in this build and its removal plan for mission | Test hook register |
| 4.E18 | **Safety-related behaviors identified**: which requirements implement or protect an inhibit, and how each will be verified at L3/L4 | Safety software list, hazard report refs |

### 7.2 Entry Criteria — Hardware Side

| # | Criterion | Required evidence |
|---|---|---|
| 4.E19 | Card available with recorded PN, revision, dash number, **serial number**, and as-built deltas | Card config record |
| 4.E20 | Firmware loaded, version recorded, and within the CSCI's declared supported range | Firmware version readback |
| 4.E21 | Bootloader version recorded and within supported range | Bootloader readback |
| 4.E22 | Card acceptance test (HW-only) passed: power, clocks, PHY, JTAG/debug access | Card ATP record |
| 4.E23 | **Safe-to-mate analysis complete**: continuity, isolation, pinout verification, no shorts, correct power sequencing, connector keying | Safe-to-mate record, signed |
| 4.E24 | Harness/breakout drawing and revision recorded; continuity and hipot as applicable | Harness config record |
| 4.E25 | Power budget confirmed; supply current-limited to a safe value for first power-on | Bench power configuration record |
| 4.E26 | Bus loads, terminations, and pull-ups correct for the configuration under test (especially 1553 terminations and I²C pull-ups) | Bench configuration record |
| 4.E27 | ESD-controlled work area, grounded, verified; personnel ESD-certified | Facility record |

### 7.3 Entry Criteria — Test Readiness

| # | Criterion | Required evidence |
|---|---|---|
| 4.E28 | Integration test procedure written, reviewed, and approved, with expected results and pass/fail criteria per step | Approved procedure |
| 4.E29 | Test equipment available and **in calibration**: bus analyzers, fault injectors, scopes, logic analyzers, power supplies, load banks | Calibration records |
| 4.E30 | Data recording plan: what is captured, at what rate, where it is stored, retention period | Data plan |
| 4.E31 | Test conductor and QA witness assigned; roles briefed | Assignment record |
| 4.E32 | Abort/safing criteria defined: conditions under which the test stops and power is removed | In procedure |
| 4.E33 | Anomaly reporting path defined and understood by all participants | Process reference |

### 7.4 Review Conduct

HIRR is a checklist-driven review. Every row in §7.1–7.3 is answered **Yes / No / Waived**.
A "No" blocks the gate. A "Waived" requires a written deviation approved by the gate authority with
a stated risk and a mitigation.

### Exit Criteria

| # | Criterion |
|---|---|
| 4.X1 | All entry criteria satisfied or formally waived |
| 4.X2 | HIRR checklist signed by all gate authorities |
| 4.X3 | Configuration of record frozen: a single, named, immutable configuration under test |
| 4.X4 | Compatibility matrix updated with the configuration marked "Under Test" |
| 4.X5 | Authorization to apply power documented with date, configuration ID, and authorizing signatures |

### Artifacts Produced
Signed HIRR checklist · frozen configuration record · deviation/waiver records · authorization to power

### Gate Authority
Chair: Integration & Test Lead. Signatories: Software Lead, Card/Hardware Lead, Systems Engineering, SQA, Safety.

---

## 8. STEP 5 — Card-Level HW/SW Integration (L3)

### Objective
Prove the CSCI runs correctly on the real card with real silicon, real PHYs, and real firmware; with the rest of the world simulated by bus analyzers and emulators.

### Entry Criteria
HIRR (Step 4) closed. Plus:

| # | Criterion |
|---|---|
| 5.E1 | Bus emulators configured to the ICD and their configuration recorded (1553 BC/RT sim, Ethernet traffic generator, I²C host/target emulator, serial terminal/protocol emulator, GPIO stimulus/monitor). PCIe is not emulated — see §18.8 |
| 5.E2 | Emulator message databases derived from the same ICD revision the CSCI declares, with matching hash |
| 5.E3 | First-power-on procedure staged: current-limited supply, monitored inrush, defined abort threshold |
| 5.E4 | Instrumentation attached: debug UART, JTAG, bus taps, current monitoring |
| 5.E5 | Bench fixture identified by drawing and revision per §18.4; terminations, pull-ups, and slot/presence strapping verified against the fixture drawing |
| 5.E6 | Test article under test identified per §18.1, at a recorded version; if it is not the release CSCI, the verification credit it can earn is stated up front |
| 5.E7 | Bench time base established and all instruments synchronized to it per §18.9 |
| 5.E8 | Checkout cell canary passed within the current shift, or the campaign is run manually with that stated in the record (§19.8) |

### Activities
**Sequence matters.** Bring up in this order; do not jump ahead:

1. **Power-on and boot** — current draw within predicted envelope, boot reaches bootloader, correct image selected, correct version reported
2. **Clocks and reset** — verify all clock domains present and within tolerance; reset sources behave as designed
3. **Memory** — RAM test, flash readback verification against the delivered SHA-256, EDAC/ECC functional
4. **Per-bus bring-up, one bus at a time** — GPIO → I²C → serial → 1553 → Ethernet → PCIe.
   PCIe is brought up for communication only; the link itself is COTS and unverified (§14.6).
   Simplest and most observable first. Each bus fully nominal before moving to the next.
5. **Multi-bus concurrent operation** — all buses active simultaneously at rate
6. **Full functional threads** at rate
7. **Off-nominal campaign** (§14)
8. **Soak** at rate with data recording

### Test Scope: Nominal
- Boot time within requirement, from every reset source
- Reported software/firmware/bootloader versions match the DRP exactly (version readback is a mandatory test, not a nicety)
- Every ICD message on every bus: transmitted at correct rate, correct content, correct encoding
- Timing measured with a scope or analyzer, not just self-reported: frame boundaries, output latency, sync alignment to the 1PPS or frame sync discrete
- Discrete I/O: every input read correctly at both levels; every output drives correct level with correct timing
- CPU utilization, memory usage, and temperature under full concurrent bus load
- Health and status telemetry accuracy against independently measured truth

### Test Scope: Off-Nominal
Execute the full per-bus fault matrix in §14 at card level, plus:
- **Power**: undervoltage ramp to brownout threshold, overvoltage to spec limit, slow ramp, fast ramp, power interruption at 10 ms to 1 s intervals, power cycling during a flash write
- **Reset during activity**: assert reset during each bus transaction type; verify no corruption and correct recovery
- **Firmware/software version mismatch**: deliberately load an out-of-range firmware version; verify the CSCI detects the incompatibility and safes rather than running
- **Corrupted image**: flip a bit in the application image; verify CRC/signature check catches it and the golden/fallback image loads
- **Thermal**: operate at cold start and hot soak limits; verify timing margins hold
- **EDAC/SEU emulation**: inject single- and double-bit errors in RAM via the EDAC test path; verify correction, logging, and uncorrectable-error response
- **Watchdog on real silicon**: verify the hardware watchdog (not just the software one) fires and resets within its window

### Exit Criteria

| # | Criterion | Evidence |
|---|---|---|
| 5.X1 | Boot from every reset source, 100% success over ≥50 consecutive cycles | Power-cycle test log |
| 5.X2 | Version readback matches DRP for CSCI, firmware, and bootloader | Readback record |
| 5.X3 | Every bus verified nominal at rate, individually and concurrently | Bus test reports + analyzer captures |
| 5.X4 | Timing requirements verified by external measurement | Scope/analyzer captures with annotations |
| 5.X5 | Full off-nominal matrix executed; every fault produces the specified response | Off-nominal report |
| 5.X6 | Resource margins confirmed on real hardware (CPU, RAM, flash, thermal) | Measurement report |
| 5.X7 | Soak ≥24 h at rate with no unexplained anomaly | Soak report |
| 5.X8 | All L3-allocated requirements verified | RTM extract |
| 5.X9 | Zero open Sev 1/2; open Sev 3 dispositioned for L4 entry | Defect report |
| 5.X10 | Compatibility matrix row updated to "Card Verified" with exact versions | Matrix rev |

### Artifacts Produced
Card integration test report · analyzer captures (retained as data, not screenshots) · off-nominal report · soak report · as-run procedure with redlines · updated compatibility matrix · updated RTM

### Gate Authority
**Card Integration Complete (CIC)**: Card Lead (chair), Software Lead, I&T Lead, SQA.

---

## 9. STEP 6 — Box / Backplane Integration (L4)

### Objective
Integrate all cards into the real chassis on the real backplane and verify that the box behaves as a system: shared buses arbitrate correctly, power sequences correctly, cards interoperate, and the box degrades gracefully when a card fails.

This is the first level where **backplane contention, cross-card timing, and intra-box EMI** are observable. Many defects live here and nowhere else.

### Entry Criteria

| # | Criterion |
|---|---|
| 6.E1 | Every card in the box has passed CIC (Step 5), or has an approved deviation with a stated risk |
| 6.E2 | Backplane HWCI identified by PN, revision, and serial; acceptance tested standalone |
| 6.E3 | Chassis power supply/PDU verified; total box power budget confirmed against the sum of measured per-card draws |
| 6.E4 | **Compatibility matrix confirms the exact card/firmware/CSCI/backplane combination is an allowed row** |
| 6.E5 | Card insertion order, slot assignment, and keying verified against the drawing |
| 6.E6 | Backplane bus topology confirmed: 1553 stub lengths and terminations, Ethernet switch config and VLANs, PCIe lane mapping and link width, I²C address map with no collisions, GPIO net list |
| 6.E7 | **I²C address map reviewed for collisions across all cards** — a frequent and expensive find at this level |
| 6.E8 | Power sequencing requirements between cards defined and implemented |
| 6.E9 | Box-level test procedure approved with per-step expected results |
| 6.E10 | Safe-to-mate performed at box level: backplane continuity, no shorts across slots, connector integrity |
| 6.E11 | Time synchronization source and distribution defined (1PPS, PTP, 1553 sync-with-data, or frame sync discrete) |

### Activities
1. Power on with cards progressively populated — never all slots at once on first power
2. Verify power sequencing and inrush at box level
3. Establish time sync across cards; measure actual inter-card time alignment
4. Bring up each shared bus with all participants active
5. Run integrated box functional threads
6. Execute off-nominal campaign including card-removal and card-failure cases
7. Thermal soak with all cards running at full duty

### Test Scope: Nominal
- Box power-on to operational within requirement; power sequencing correct and repeatable
- All cards enumerate/announce; box-level health and status reflects true card state
- **Backplane bus performance under full load**: aggregate 1553 bus loading %, Ethernet throughput and switch buffer occupancy, PCIe link width/speed and error counters at zero, I²C bus loading and clock-stretch behavior with all targets present
- Inter-card time alignment measured and within requirement
- Cross-card data flows: sensor card → processing card → effector card, end-to-end latency measured
- Box-level telemetry aggregation: completeness, rate, and correct source attribution
- Redundancy: if the box contains redundant elements, verify both are exercised and both report

### Test Scope: Off-Nominal
- **Card removal / non-presence**: for each slot, run with the card absent; verify the box detects, reports, and continues in the defined degraded mode
- **Card failure simulation**: hold a card in reset, power it down mid-operation, or have it assert a fault discrete; verify detection latency and box response
- **Bus contention and saturation**: drive each shared bus to and past its rated load; verify graceful degradation, not collapse; verify overload is detected and reported
- **Backplane bus fault injection** per §14, now with real cards as victims and aggressors
- **Babbling idiot**: have one card transmit continuously/erratically on a shared bus; verify containment — the rest of the box must survive. Test on Ethernet (flood), 1553 (unauthorized transmit), and I²C (SDA held low)
- **Time sync loss**: remove the sync source; verify each card's holdover behavior and the drift rate
- **Power sequencing violation**: bring cards up in the wrong order, and remove power to one card while others run; verify no latch-up, no back-powering, no damage
- **Simultaneous faults**: two faults within one detection window (a fault during recovery from another fault) — this is where FDIR designs typically break
- **Thermal**: operate at hot and cold limits; verify no timing violations appear only at temperature
- **Intra-box EMI**: operate all buses at maximum activity simultaneously and check for bit errors, CRC errors, and analog measurement degradation on neighboring cards

### Exit Criteria

| # | Criterion | Evidence |
|---|---|---|
| 6.X1 | Box powers on and reaches operational state reliably (≥50 cycles, 100% success) | Cycle log |
| 6.X2 | All cards report the exact expected versions; box-level configuration readback matches the CSLS records | Readback record |
| 6.X3 | All shared buses verified nominal under full concurrent load; zero unexplained error counts | Bus reports, error counter dumps |
| 6.X4 | Inter-card timing and time sync within requirement, measured externally | Timing report |
| 6.X5 | Every card-absent and card-failure case produces the specified degraded behavior | Off-nominal report |
| 6.X6 | Babbling-idiot containment demonstrated on every shared bus | Containment test report |
| 6.X7 | Box thermal performance acceptable; no temperature-dependent functional or timing failures | Thermal test report |
| 6.X8 | All L4-allocated requirements verified | RTM extract |
| 6.X9 | Zero open Sev 1/2 | Defect report |
| 6.X10 | Box configuration baselined: every card PN/rev/SN, firmware, bootloader, CSCI version | Box configuration record |

### Artifacts Produced
Box integration test report · bus load and error analyses · timing/sync report · degraded-mode report · thermal report · baselined box configuration · updated compatibility matrix

### Gate Authority
**Box Integration Complete (BIC)**: Box/LRU Lead (chair), Software Lead(s), Card Lead(s), I&T Lead, Systems Engineering, SQA.

---

## 10. STEP 7 — Box Qualification under Environments (L4+)

### Objective
Demonstrate that the integrated box — hardware and software together — meets requirements while exposed to the mission environment. Software is an active participant: it runs, it is monitored, and functional performance is verified *during* exposure, not only before and after.

### Entry Criteria

| # | Criterion |
|---|---|
| 7.E1 | BIC (Step 6) closed with the configuration baselined |
| 7.E2 | Environmental test plan approved with levels, durations, and tailoring rationale |
| 7.E3 | **Software configuration frozen for the qualification campaign** — a software change mid-campaign invalidates prior exposure unless a documented delta-qual is approved |
| 7.E4 | Functional test procedure defined for execution *during* exposure, with pass/fail criteria evaluated in real time |
| 7.E5 | Real-time monitoring and data recording from the article under test through the chamber/fixture feedthrough |
| 7.E6 | Abort criteria defined and briefed: what stops the test |
| 7.E7 | Pre-test functional baseline executed and recorded |

### Activities
- Vibration (random, sine, acoustic as applicable) with software running and monitored
- Mechanical shock
- Thermal cycling and thermal-vacuum, with functional tests at hot and cold plateaus
- EMI/EMC: radiated and conducted emissions and susceptibility, with software at full duty cycle
- Radiation as applicable: TID, SEE characterization on representative parts, with software FDIR monitored
- Post-exposure functional to the same baseline

### Test Scope: Nominal
- Continuous functional operation and health monitoring throughout each exposure
- Full functional test at each thermal plateau and after each exposure
- Bus error counters monitored continuously — a rise during vibration points at a connector or solder joint, and software is often the only instrument that sees it

### Test Scope: Off-Nominal
- **Intermittent connection**: vibration-induced bus dropouts — verify software detects, reports, and recovers rather than silently accepting corrupt data
- **EMI-induced bit errors**: verify CRC/parity detection actually triggers under EMC susceptibility exposure
- **SEU response**: verify EDAC scrubbing, error logging, and the uncorrectable-error safing path under beam or emulated injection
- **Thermal extremes**: re-run the timing-critical subset at both temperature limits; verify margins
- **Reset and recovery under environment**: power cycle at hot, at cold, and during vibration

### Exit Criteria

| # | Criterion | Evidence |
|---|---|---|
| 7.X1 | All environmental exposures completed at required levels and durations | Environmental test reports |
| 7.X2 | Functional performance maintained throughout; every in-test anomaly resolved or dispositioned | Monitoring data, anomaly records |
| 7.X3 | Pre- and post-exposure functional results match within tolerance | Comparison record |
| 7.X4 | No configuration change during the campaign, or delta-qual approved and executed | CM record |
| 7.X5 | All environment-allocated requirements verified | RTM extract |
| 7.X6 | Qualified box configuration published to the compatibility matrix as "Qualified" | Matrix rev |

### Artifacts Produced
Environmental qualification reports · in-test monitoring data · anomaly records and dispositions · qualified configuration baseline

### Gate Authority
**Box Qualification Complete (BQC)**: Box Lead (chair), Systems Engineering, Reliability, SQA, Customer/Program as required.

---

## 11. STEP 8 — System Integration Lab / Integrated Test Bed (L5)

### Objective
Integrate multiple boxes across strings with mission-representative harness, closed-loop system dynamics simulation, and the real ground segment. Verify mission behavior, redundancy management, and system-level FDIR.

### Entry Criteria

| # | Criterion |
|---|---|
| 8.E1 | All participating boxes have passed BIC; mission-path boxes have passed or are scheduled for BQC |
| 8.E2 | **No `-rc` builds** — all CSCIs at released versions |
| 8.E3 | System-level compatibility matrix confirms the complete cross-box configuration is an allowed set |
| 8.E4 | Mission-representative harness installed, revision recorded, continuity verified end to end |
| 8.E5 | HWIL simulation validated: system dynamics, sensor models, effector models, environment — each with a documented fidelity and validation record |
| 8.E6 | Mission timeline / mission sequence data files version-controlled and loaded at known versions |
| 8.E7 | Ground segment (EGSE, telemetry ground station, command path) integrated and version-recorded |
| 8.E8 | Test scenarios defined and reviewed, covering nominal mission and the fault scenario catalog |
| 8.E9 | Data recording sufficient to reconstruct any run: all buses, all telemetry, sim state, timestamps on a common time base |
| 8.E10 | Safety-critical and mission termination interfaces simulated with the correct inhibit logic — never live |

### Activities
- End-to-end nominal mission runs, closed loop
- Redundancy management exercises: string failover under every credible trigger
- Fault scenario campaign from the FMEA/hazard analysis
- Ground command and telemetry path verification, including latency and command authentication
- Day-in-the-life and mission rehearsal sequences, including hold/recycle
- Monte Carlo dispersions where the CSCI participates in a control or decision loop

### Test Scope: Nominal
- Full mission timeline executed closed-loop, repeatedly, with dispersions
- Cross-string data comparison and voting behavior where applicable
- Inter-box latency measured end to end against requirement
- Telemetry completeness: every defined measurand present, at rate, correctly scaled, correctly timestamped on the common time base
- Command path: every command in the command dictionary exercised end to end from the console
- Mode and phase transitions at the correct triggers, with correct timing

### Test Scope: Off-Nominal
- **Box loss**: power off each box mid-mission; verify detection, string failover, and mission continuation or safing per design
- **String failover**: trigger via every defined mechanism — health status, heartbeat loss, voting disagreement, manual command. Measure failover time against requirement.
- **Split-brain**: create conditions where two strings each believe they are prime; verify the arbitration mechanism resolves it deterministically
- **Inter-box bus faults**: cable open, short, intermittent, connector loosened; single-bus and redundant-bus loss
- **Sensor faults**: stuck, drifting, noisy, out-of-range, dropout, and disagreeing-redundant-sensor cases; verify voting/isolation
- **Effector faults**: commanded but not responding, responding late, responding wrong, hardover
- **Time sync loss at system scope**: verify holdover, drift, and recovery across boxes
- **Ground segment loss**: command path loss, telemetry loss, both; verify autonomous behavior
- **Fault during recovery**: inject a second fault during failover — the highest-yield off-nominal test in this step
- **Timeline anomalies**: hold, recycle, abort, and late commit; verify state consistency
- **Common-cause**: same software defect condition triggered in both strings simultaneously (verify the argument for dissimilarity or the acceptance of common-mode risk)

### Exit Criteria

| # | Criterion | Evidence |
|---|---|---|
| 8.X1 | Nominal mission executed successfully with required repeat count and dispersion coverage | Run log, Monte Carlo summary |
| 8.X2 | Every fault scenario in the catalog executed; system response matches the FMEA/hazard analysis prediction | Fault campaign report |
| 8.X3 | Redundancy management verified for every failover trigger; failover time within requirement | Failover report with measured times |
| 8.X4 | Every hazard control implemented in software verified at system level | Safety verification report |
| 8.X5 | End-to-end latencies and timing verified against requirement | Timing report |
| 8.X6 | All L5-allocated requirements verified | RTM extract |
| 8.X7 | Zero open Sev 1/2; all Sev 3 dispositioned with mission rationale | Defect report |
| 8.X8 | System configuration baselined and published as the mission candidate | Baseline record |

### Artifacts Produced
System integration test reports · fault campaign report · redundancy/failover report · safety verification report · Monte Carlo results · mission candidate configuration baseline

### Gate Authority
**System Integration Complete (SIC)**: System I&T Lead (chair), Chief Engineer, Software Lead(s), Systems Engineering, Safety, Mission Assurance, SQA.

---

## 12. STEP 9 — Formal Qualification Test (L6)

### Objective
Verify the software in the as-built system with mission hardware, mission harness, real sensors, and real effectors. This step finds as-built vs as-designed deltas that no lab can reproduce.

### Entry Criteria

| # | Criterion |
|---|---|
| 9.E1 | SIC closed; mission candidate configuration baselined |
| 9.E2 | All mission hardware installed with as-built configuration recorded to serial number |
| 9.E3 | **As-built vs as-tested delta analysis complete** — every difference between the L5 configuration and the system identified and assessed |
| 9.E4 | Mission software loaded via the mission load procedure with post-load verification (readback + checksum against DRP) |
| 9.E5 | Mission harness installed per drawing; continuity, isolation, and grounding verified; pin-level verification of safety-critical circuits |
| 9.E6 | Ordnance, propulsion, and FTS interfaces **inhibited and verified inhibited** before any software that commands them is powered |
| 9.E7 | Test procedures approved with explicit safing steps and stop conditions |
| 9.E8 | Range/facility safety approvals in place |
| 9.E9 | Personnel certified for the operations being performed |

### Activities
- Power-on and aliveness of the integrated system
- Interface verification test (IVT): every mission interface verified with the real end item
- Integrated systems test (IST): mission sequences with effectors inhibited or in a safe configuration
- Combined systems test / mission rehearsal
- Mission execution support, where software participates in sequencing and abort
- Post-test data review after every run

### Test Scope: Nominal
- System power-on, software boot, version readback from every box — verified against the mission configuration baseline
- Every mission interface exercised with the real sensor or effector: polarity, scaling, sense, range
- Mission sequence execution with all inhibits in place
- Telemetry through the real mission telemetry path to the real ground station
- Command path from the real console through the real uplink

### Test Scope: Off-Nominal
- **Inhibit verification**: attempt to command each inhibited function; verify the command is blocked by the intended inhibit and the block is reported. This is a safety verification and is witnessed.
- **Polarity and sense errors**: verify that a reversed sensor or effector would be detected — where it cannot be detected, the risk is documented and accepted explicitly
- **Real harness faults**: where safe, disconnect or open a non-critical circuit; verify detection and reporting
- **Abort cases**: trigger every automatic abort condition and every manual abort path; verify safing occurs within the required time
- **Hold and recycle**: enter hold at multiple points in the test; verify state consistency and successful recycle
- **Loss of link** during test; verify defined autonomous behavior
- **Power transfer**: system power to battery power transfer, and the failure of that transfer
- **Late-load and reload**: verify the procedure for loading software late in the flow, including verification that the correct version is actually running

### Exit Criteria

| # | Criterion | Evidence |
|---|---|---|
| 9.X1 | IVT complete: every mission interface verified against the real end item | IVT report |
| 9.X2 | IST / combined systems test complete and successful | IST report |
| 9.X3 | Mission rehearsal executed successfully including holds and recycles | Rehearsal report |
| 9.X4 | All inhibits verified, witnessed, and documented | Safety verification record |
| 9.X5 | Version readback from every box matches the mission configuration baseline exactly | Configuration verification record |
| 9.X6 | As-built vs as-tested deltas all closed or formally accepted | Delta closure record |
| 9.X7 | All L6-allocated requirements verified | RTM extract |
| 9.X8 | Zero open Sev 1/2; all open items dispositioned for mission | Defect report |

### Artifacts Produced
IVT and IST reports · mission rehearsal report · inhibit verification records · as-built configuration verification · delta closure record

### Gate Authority
**Formal Qualification Test Complete (FQT)**: System I&T Lead (chair), Chief Engineer, Safety, Mission Assurance, Quality as applicable.

---

## 13. STEP 10 — Mission Configuration Freeze and Mission Readiness

### Objective
Establish and protect the exact software configuration that will run the mission, and close out the verification argument.

### Entry Criteria

| # | Criterion |
|---|---|
| 10.E1 | FQT closed |
| 10.E2 | RTM complete: every requirement verified at its assigned level, no open items without a disposition |
| 10.E3 | All test hooks, backdoors, simulation flags, and debug modes accounted for — each either removed, or present and justified with a verification that it cannot be inadvertently activated |
| 10.E4 | Every deviation and waiver closed or accepted at the correct authority level |
| 10.E5 | Mission images in the registry with verified digests matching what is loaded on the system |
| 10.E6 | Mission software VDD published for every CSCI |
| 10.E7 | Contingency plan defined: conditions requiring a reload, the procedure, and the schedule impact |
| 10.E8 | On-console software support plan defined: who is present, what they can and cannot change |

### Activities
- Final configuration verification: readback from system vs registry digests
- Software Mission Readiness Review input package assembled
- Freeze declared; any change after freeze requires a documented change board action with a
  re-verification plan

### Exit Criteria

| # | Criterion |
|---|---|
| 10.X1 | Mission configuration frozen, published, and verified on the system by readback |
| 10.X2 | Verification argument complete and accepted |
| 10.X3 | Software MRR closed with no open constraints, or with constraints formally accepted |
| 10.X4 | Post-freeze change control in effect |

### Gate Authority
**Mission Readiness Review (MRR)**: Program-level. Software input signed by Chief Software Engineer,
Mission Assurance, Safety.

---

## 14. Off-Nominal Test Matrix by Bus

This is the required minimum fault set. It is not exhaustive — the FMEA and hazard analysis may add
cases, and any bus-specific ICD feature adds its own.

**Test ID convention:** `OFN-<BUS>-<NNN>` for the case; run instances are
`<TestID>-L<level>-<article>-<date>`.

**"Required behavior" rules that apply to every row:**
1. The fault is **detected** within a specified time.
2. The fault is **reported** in health/status telemetry with a distinguishable cause code — not a
   generic "error."
3. The response is **defined and bounded** — no undefined behavior, no silent data acceptance, no
   unbounded retry.
4. The software **recovers** or enters a defined degraded/safe state per design.
5. The event is **logged** in non-volatile storage where the design requires post-mission
   reconstruction.

### 14.1 GPIO / Discretes

| ID | Fault condition | Injection method | Required behavior | Min level |
|---|---|---|---|---|
| OFN-GPIO-001 | Input stuck high | Jumper to rail / force in breakout | Detect via plausibility or cross-check; report; do not act on implausible state | L3 |
| OFN-GPIO-002 | Input stuck low | Jumper to ground | Same as above | L3 |
| OFN-GPIO-003 | Input open / floating | Disconnect line | Detect open (pull bias or sense circuit); report; treat as invalid, not as a valid level | L3 |
| OFN-GPIO-004 | Contact bounce | Relay/switch with no debounce, or arbitrary function generator burst | Debounce per ICD; no spurious state change; no event storm | L3 |
| OFN-GPIO-005 | Level in the indeterminate band | Programmable supply between V_IL and V_IH | Defined behavior; no oscillation; report if outside valid bands | L3 |
| OFN-GPIO-006 | Mutually exclusive discretes asserted together | Force both | Detect the illegal combination; reject; report; do not pick one arbitrarily | L3 |
| OFN-GPIO-007 | Sync/1PPS pulse missing | Suppress pulse | Detect within N frames; enter holdover; report; verify drift rate | L3 |
| OFN-GPIO-008 | Sync pulse jittered / early / late | Delay generator | Verify tolerance window; reject out-of-window pulses; report | L3 |
| OFN-GPIO-009 | Sync pulse doubled / runt | Pulse generator | Reject; do not double-step the frame counter | L3 |
| OFN-GPIO-010 | Output loaded / shorted | Resistive load, short to rail | Detect via readback or current sense; report; protect the driver | L3 |
| OFN-GPIO-011 | Output readback mismatch | Force the net opposite to commanded | Detect mismatch; report; safe the affected function | L3 |
| OFN-GPIO-012 | Discrete transition during reset | Toggle during reset assertion | No spurious output glitch; defined power-on state on every output | L3 |
| OFN-GPIO-013 | Safety inhibit discrete removed unexpectedly | Open the inhibit line | Immediate safing; report; **witnessed test** | L4/L6 |

### 14.2 I²C (Housekeeping, EEPROM, Thermal, Power Monitors)

I²C is the most fragile bus in the list and deserves disproportionate attention. A hung I²C bus has
taken down more embedded systems than its bandwidth share would suggest.

| ID | Fault condition | Injection method | Required behavior | Min level |
|---|---|---|---|---|
| OFN-I2C-001 | Target NAKs address | Remove/disable target, or programmable target emulator | Bounded retry count; report; do not block the calling task | L3 |
| OFN-I2C-002 | Target NAKs mid-transfer | Target emulator | Abort transaction cleanly; release bus; report | L3 |
| OFN-I2C-003 | **SDA held low by target (bus hang)** | Target emulator forces SDA low | Detect hang within a bounded timeout; execute recovery (9 SCL pulses + STOP); verify bus released; report; escalate if recovery fails | L3 |
| OFN-I2C-004 | SCL held low | Force SCL low | Detect; report; recovery is not possible by the host — verify the design escalates (target power cycle or bus segment isolation) rather than hanging | L3 |
| OFN-I2C-005 | Excessive clock stretching | Target emulator stretches beyond spec | Enforce a maximum stretch timeout; abort; report | L3 |
| OFN-I2C-006 | Arbitration loss (multi-master) | Second master on the bus | Detect; back off; retry with bounded attempts | L3 |
| OFN-I2C-007 | Address collision across cards | Configure two targets to the same address | **Detect during box bring-up**; this is a design error and must be caught, not tolerated | L4 |
| OFN-I2C-008 | Corrupted data (bit flip) | Inject glitch on SDA during data phase | Detect via PEC/CRC or plausibility check; reject; report; do not use the value | L3 |
| OFN-I2C-009 | Glitch on SCL | Glitch injector | No false clock edge accepted; recover | L3 |
| OFN-I2C-010 | Target returns stale/frozen data | Target emulator returns a fixed value | Staleness detection (change detection, counter, or timestamp); report | L3 |
| OFN-I2C-011 | Target returns out-of-range value | Target emulator | Range check before use; reject; report; do not propagate to control | L3 |
| OFN-I2C-012 | EEPROM write interrupted by power loss | Cut power mid-write | Detect corruption on next read (CRC); fall back to redundant copy; report | L3 |
| OFN-I2C-013 | Bus capacitance / rise time marginal | Add capacitance to the bus | Characterize; verify error detection at the margin; document limits | L3/L4 |
| OFN-I2C-014 | Target absent at init | Remove target | Init does not hang; report missing device; continue in degraded mode per design | L3 |

### 14.3 Serial (RS-422 / RS-485 / UART)

| ID | Fault condition | Injection method | Required behavior | Min level |
|---|---|---|---|---|
| OFN-SER-001 | Baud rate mismatch | Configure peer emulator off-baud | Framing errors detected; report; do not accept garbage as data | L3 |
| OFN-SER-002 | Framing error | Corrupt stop bit via injector | Detect; discard frame; count; report | L3 |
| OFN-SER-003 | Parity error | Force wrong parity | Detect; discard; count; report | L3 |
| OFN-SER-004 | Break condition | Hold line in break | Detect; report; recover when cleared without a stuck state | L3 |
| OFN-SER-005 | Truncated message | Cut transmission mid-frame | Inter-character timeout; discard partial; resynchronize on the next valid frame | L3 |
| OFN-SER-006 | Extra/garbage bytes between frames | Inject noise bytes | Resynchronize on sync pattern; do not misalign the frame parser | L3 |
| OFN-SER-007 | Bad checksum/CRC | Corrupt the checksum field | Reject frame; count; report; never use the payload | L3 |
| OFN-SER-008 | Message rate too high (flood) | Traffic generator at 10× rate | RX buffer does not overflow silently; overrun detected and reported; higher-priority tasks still meet deadlines | L3 |
| OFN-SER-009 | Message rate too low / absent | Stop transmission | Timeout detection within the specified window; report; enter defined degraded behavior | L3 |
| OFN-SER-010 | RX overrun | Suspend the RX task while data arrives | Detect overrun; report; recover without losing frame sync permanently | L3 |
| OFN-SER-011 | RS-485 bus contention | Two nodes transmit simultaneously | Detect collision/garbling; back off per protocol; report | L3 |
| OFN-SER-012 | RS-485 transmitter stuck enabled | Hold DE asserted on an emulator node | Detect the bus is held; report; the victim node must not deadlock waiting for the bus | L3/L4 |
| OFN-SER-013 | Line open / short / reversed pair | Harness fault box | Detect no-communication; report distinguishable from a silent-peer case if the design allows | L3 |
| OFN-SER-014 | Sequence number gap or repeat | Emulator sends out-of-sequence | Detect; report; defined handling (reject vs reorder) per ICD | L3 |

### 14.4 MIL-STD-1553

Test both as BC and as RT, per the CSCI's declared role, and both channels A and B.

| ID | Fault condition | Injection method | Required behavior | Min level |
|---|---|---|---|---|
| OFN-1553-001 | RT no response to a command | Disable the RT on the emulator | BC detects no-response timeout; retries per ICD; retries on the alternate bus; reports; declares the RT failed after N attempts | L3 |
| OFN-1553-002 | RT responds late (outside the response-time window) | Bus fault injector delays the status word | Treat as no-response per standard; do not accept a late response | L3 |
| OFN-1553-003 | Status word Message Error bit set | Emulator sets ME | Defined handling; retry or fail per ICD; report | L3 |
| OFN-1553-004 | Status word Busy bit set | Emulator sets Busy | Defined retry policy with bounded attempts; report if persistent | L3 |
| OFN-1553-005 | Status word Subsystem Flag / Terminal Flag set | Emulator sets flags | Detect; report; the affected data is dispositioned per ICD | L3 |
| OFN-1553-006 | Status word Service Request | Emulator sets SRQ | Correct vector word handling and follow-up transfer | L3 |
| OFN-1553-007 | Word count mismatch | Injector sends wrong data word count | Detect; reject the message; report | L3 |
| OFN-1553-008 | Manchester encoding error | Bus fault injector | Detect; reject; count; report | L3 |
| OFN-1553-009 | Parity error on a word | Injector | Detect; reject; count | L3 |
| OFN-1553-010 | Sync pattern error | Injector | Detect; reject | L3 |
| OFN-1553-011 | Illegal command to RT (undefined subaddress/mode code) | BC emulator | RT sets Message Error and does not act; superseding illegalization behavior per ICD | L3 |
| OFN-1553-012 | Bus A failure → failover to bus B | Open/terminate channel A | Failover within the required time; report; verify no data loss beyond the specified allowance | L3 |
| OFN-1553-013 | Both buses degraded | Degrade both | Defined safing behavior; report | L3/L4 |
| OFN-1553-014 | Bus loading exceeds design | Traffic generator drives loading to and past the limit | Detect over-subscription; report; higher-priority messages still meet their rate | L4 |
| OFN-1553-015 | **Babbling RT** — unauthorized transmission | Emulated RT transmits without command | BC and other RTs survive; the offender is detected and reported; verify any hardware transmitter-timeout is effective | L4 |
| OFN-1553-016 | Mode code handling: Reset RT, Transmit BIT Word, Synchronize (with/without data) | BC emulator issues each mode code | Each handled per standard and ICD; Synchronize aligns the frame correctly | L3 |
| OFN-1553-017 | Data stale — same data repeated | Emulator freezes the payload | Staleness detection via counter or timestamp; report; do not feed stale data to control | L3 |
| OFN-1553-018 | Terminated/unterminated stub, marginal coupling | Modify termination on the bench | Characterize error rate; verify detection at the margin; document | L3/L4 |
| OFN-1553-019 | BC loss (RT perspective) | Stop the BC | RT detects loss of bus activity; enters defined mode; reports; does not latch in a waiting state | L3 |
| OFN-1553-020 | Command during an in-progress transfer | Injector | Correct per-standard abort and resynchronization | L3 |

### 14.5 Ethernet (Telemetry / Bulk / Control Plane)

| ID | Fault condition | Injection method | Required behavior | Min level |
|---|---|---|---|---|
| OFN-ETH-001 | Link down | Unplug / disable the port | Detect link state change; report; failover to the redundant path if designed; recover on relink | L3 |
| OFN-ETH-002 | Link flap | Automated relay cycling the link | Bounded reconnection attempts; no reconnect storm; no resource leak per attempt | L3 |
| OFN-ETH-003 | Link speed/duplex mismatch | Force the peer to a different config | Detect; report; do not operate silently degraded | L3 |
| OFN-ETH-004 | Packet loss (1%, 10%, 50%) | Network impairment device | Detect via sequence gaps; report loss rate; application behavior defined at each level | L3 |
| OFN-ETH-005 | Packet reordering | Impairment device | Detect and reorder or reject per ICD; never process out of order silently | L3 |
| OFN-ETH-006 | Packet duplication | Impairment device | Detect and discard duplicates; no double-processing of commands | L3 |
| OFN-ETH-007 | Latency and jitter injection | Impairment device | Timing requirements evaluated at the specified bounds; degraded behavior defined beyond them | L3 |
| OFN-ETH-008 | Bad CRC frames | Frame injector | Dropped at the MAC; counted; reported | L3 |
| OFN-ETH-009 | Malformed frames: bad length field, truncated, oversized (jumbo when not supported), runt | Raw packet crafting | Rejected without a crash, hang, or buffer overrun. **This is also a security test.** | L3 |
| OFN-ETH-010 | Fragmented IP datagrams, overlapping fragments | Packet crafting | Defined handling; no reassembly buffer exhaustion | L3 |
| OFN-ETH-011 | Broadcast/multicast storm | Traffic generator at line rate | CPU survives; high-priority processing still meets deadlines; storm detected and reported; verify switch storm control if present | L3/L4 |
| OFN-ETH-012 | Unicast flood at line rate to this node | Traffic generator | RX path does not exhaust buffers; controlled drop; report | L3/L4 |
| OFN-ETH-013 | ARP conflict / duplicate IP | Second node with the same IP | Detect; report; defined behavior — not silent misrouting | L4 |
| OFN-ETH-014 | Unexpected source address | Packet crafting | Reject if the ICD specifies expected sources; report | L3 |
| OFN-ETH-015 | Time sync loss (PTP/1588 or equivalent) | Stop the grandmaster | Holdover; measured drift rate; report; recovery on restoration without a time step that breaks the application | L4 |
| OFN-ETH-016 | Time sync step / false master | Inject a rogue master with a wrong time | Reject per BMCA/authentication design; never accept a backward time step in mission software | L4 |
| OFN-ETH-017 | Switch/VLAN misconfiguration | Misconfigure the backplane switch | Detect unreachable peers; report; do not hang at init | L4 |
| OFN-ETH-018 | Full-rate traffic on all ports simultaneously | Traffic generators on every port | Backplane switch buffer behavior characterized; no head-of-line blocking of critical traffic | L4 |

### 14.6 PCIe — COTS Link, Software Response Only

> **Scope.** PCIe cards, the PCIe stack, and the device driver are COTS and vendor-supplied. Link
> training, LTSSM behavior, TLP formation, replay, and AER generation are the vendor's
> implementation and are **not verified by this process**. What is verified here is (a) this
> system's use of the link and (b) this software's response when the vendor stack reports a fault.
> See §18.8 for the bench implications and §2.7 for the configuration control this scoping requires.

**Design rule — end-to-end integrity above the COTS stack.** PCIe carries mission-critical data on
a stack this program cannot inspect and cannot fix. Payloads crossing PCIe therefore carry their own
application-layer sequence number, length, CRC, and timestamp, checked by the receiving CSCI before
the data is used. Without this, a silent corruption, truncation, duplication, or reorder inside a
layer nobody on this program owns is undetectable — and the consequence lands on a mission-critical
path. The integrity fields are not optional and their checking is verified at L2 by deliberately
corrupting payloads at the application boundary.

**Nominal cases (must pass before off-nominal):**

| ID | Case | Required behavior | Min level |
|---|---|---|---|
| NOM-PCIE-001 | Every ICD message across PCIe, transmitted and received at rate | Correct content, encoding, scaling, and units; no loss at operational rate | L3 |
| NOM-PCIE-002 | Sustained throughput and latency at operational load | Measured externally against requirement, with margin | L3 |
| NOM-PCIE-003 | Enumeration reports the expected device list, vendor revision, link width, and link speed | All four recorded in the test record; compared to the expected values from §2.7 | L3 |
| NOM-PCIE-004 | End-to-end integrity fields generated and checked on every payload | Sequence, length, CRC, and timestamp present and validated | L2 |

**Off-nominal cases.** Injection is at the driver API boundary, in configuration space, or by
physically disturbing the device — not at the protocol layer. No protocol exerciser is required.

| ID | Fault condition | Injection method | Required behavior | Min level |
|---|---|---|---|---|
| OFN-PCIE-001 | Device absent or fails to enumerate | Depopulate the slot, or hold the endpoint in reset through boot | Detect; report; **no boot hang**; continue in the defined degraded mode | L3 |
| OFN-PCIE-002 | Link trains to a degraded width or speed | Clamp width/speed in configuration space, or vendor tooling | Detect actual vs expected at startup; report; re-evaluate the bandwidth budget; **never run silently degraded** | L3 |
| OFN-PCIE-003 | Surprise link down during traffic | Power down the endpoint mid-transfer | Detect; abort outstanding transactions; no hang; no partial payload consumed; report | L3 |
| OFN-PCIE-004 | Hot removal and re-add | sysfs remove / rescan, or equivalent BSP facility | Clean teardown and re-initialization; no resource leak over ≥1000 cycles | L3 |
| OFN-PCIE-005 | Secondary bus reset or function-level reset during operation | sysfs or configuration space | Clean recovery; state consistency verified; no stale buffer consumed after reset | L3 |
| OFN-PCIE-006 | Completion timeout | Hold the endpoint in reset mid-transaction | Detect within the configured timeout; abort; the requesting task is not blocked indefinitely | L3 |
| OFN-PCIE-007 | Driver API returns an error on every call path | Fault injection at the driver API boundary — each error return forced at least once | Every error path exercised; defined handling; no undefined behavior | L2 |
| OFN-PCIE-008 | Driver call blocks indefinitely | Delay or stall injected at the API boundary | Bounded timeout in the calling software. The watchdog is a backstop, not the primary defense. | L2 |
| OFN-PCIE-009 | AER correctable errors accumulating | Kernel AER injection where the BSP supports it; otherwise read and trend the counters under stress | Counted, thresholded, and reported; a link degrading over time is visible in telemetry before it fails | L3 |
| OFN-PCIE-010 | AER uncorrectable, non-fatal and fatal | Kernel or platform error injection | Defined recovery or safing; reset cause correctly attributed; logged | L3 |
| OFN-PCIE-011 | Payload corrupted in transit | Corrupt the payload at the application boundary | End-to-end CRC catches it; data rejected, not consumed; reported | L2 |
| OFN-PCIE-012 | Payload duplicated, reordered, or stale | Replay or delay at the application boundary | Sequence and timestamp checks catch it; defined handling per ICD | L2 |
| OFN-PCIE-013 | Unexpected device, wrong device, or unexpected vendor revision at enumeration | Populate a different card, or alter the expected device list | Configuration mismatch detected against the expected device and revision list (§2.7); operations inhibited or clearly flagged; **not proceeded with silently** | L4 |
| OFN-PCIE-014 | Vendor driver or stack version outside the qualified range | Install an out-of-range driver version | Detected at startup; reported; never runs silently on an unqualified stack | L3 |
| OFN-PCIE-015 | Reference clock loss or jitter | Clock fault injection at the fixture | Detect; report; defined safing | L3/L4 |
| OFN-PCIE-016 | Sustained maximum throughput with concurrent CPU load | Traffic plus synthetic load | No deadline misses; thermal and power within budget. This is a software budget question, not a PCIe question, and it remains in scope. | L4 |

**Cases deliberately not executed.** Poisoned TLP, malformed TLP, endpoint-initiated DMA to an
invalid address, and DMA overrun beyond a descriptor all require protocol-layer injection and all
verify the vendor's implementation rather than this system's. They are out of scope.

Where the consequence of such a fault would be memory corruption, the mitigation is **platform
configuration, verified by inspection rather than by injection**: IOMMU or address-window
enforcement enabled and correctly scoped, verified as a configuration item at L3 and recorded in
the box configuration baseline. Record this as an analysis-based closure with the rationale, not as
an untested gap.

### 14.7 Power, Clock, Memory, and Processing Element

| ID | Fault condition | Injection method | Required behavior | Min level |
|---|---|---|---|---|
| OFN-PWR-001 | Undervoltage / brownout ramp to the POR threshold | Programmable supply ramp | Clean reset; no partial execution; correct reset cause reported | L3 |
| OFN-PWR-002 | Overvoltage to the spec limit | Programmable supply | Continued correct operation to the limit; protection above it | L3 |
| OFN-PWR-003 | Power interruption 1 ms – 1 s, swept | Programmable load switch | Correct recovery at every duration; no hung state in any window | L3 |
| OFN-PWR-004 | Power loss during a flash/EEPROM write | Cut power mid-write, repeated ≥100× | No corruption of the active image; fallback available; detected on next boot | L3 |
| OFN-PWR-005 | Slow power ramp (below the POR spec rate) | Programmable supply | Defined behavior; no metastable boot | L3 |
| OFN-PWR-006 | Inrush exceeding the predicted envelope | Measure at first power-on | Within the box budget; no supply foldback | L3/L4 |
| OFN-CLK-001 | Primary oscillator loss | Disable the oscillator | Detect; switch to backup or safe; report | L3 |
| OFN-CLK-002 | Clock frequency drift beyond tolerance | Programmable clock source | Detect; report; timing behavior characterized | L3 |
| OFN-MEM-001 | Single-bit RAM error | EDAC injection register | Corrected; counted; reported; scrubbed | L3 |
| OFN-MEM-002 | Double-bit / uncorrectable RAM error | EDAC injection | Detected; defined safing or reset; reset cause correct; logged | L3 |
| OFN-MEM-003 | Scrubber disabled or failed | Disable the scrub task | Detected; reported | L3 |
| OFN-MEM-004 | Flash bit rot / corrupt application image | Deliberately corrupt a byte in the image | CRC/signature check at boot rejects it; golden image loads; reported | L3 |
| OFN-MEM-005 | Corrupt golden image *and* application image | Corrupt both | Defined last-resort behavior — typically bootloader-only safe mode with a reload path | L3 |
| OFN-WDG-001 | Task hang in each task | Suspend each task in turn | Watchdog fires within its window; reset cause identifies the offending task if the design supports it | L2/L3 |
| OFN-WDG-002 | Watchdog service from a hung system (false kick) | Construct a scenario where a low-priority task kicks while the main loop is hung | Verify the design prevents this — a windowed or multi-stage watchdog, not a single unconditional kick | L2/L3 |
| OFN-SEE-001 | SEU in a register/state machine | Radiation beam or fault injection in firmware | Detected by EDAC/TMR/plausibility; recovered or safed; reported | L4+ |
| OFN-SEE-002 | Single-event functional interrupt | Beam or injection | Detected; reset/recovery path exercised; time-to-recover measured | L4+ |
| OFN-SEE-003 | Single-event latch-up | Beam, with current monitoring | Current limit trips; power cycle recovery; reported | L4+ |

### 14.8 Cross-Cutting Off-Nominal

| ID | Fault condition | Required behavior | Min level |
|---|---|---|---|
| OFN-X-001 | Two simultaneous faults on different buses | Both detected and reported independently; no masking of one by the other | L4 |
| OFN-X-002 | Second fault injected during recovery from a first | Defined, bounded behavior; no recovery loop, no state corruption | L4/L5 |
| OFN-X-003 | Fault storm: many faults in one detection window | Reporting is rate-limited but never drops the highest-severity cause; no telemetry buffer overflow that loses the root cause | L4 |
| OFN-X-004 | Fault during mode transition | Transition completes or aborts to a defined state — never lands in a partial state | L4/L5 |
| OFN-X-005 | Fault during software load / reload | Load fails safe; previous image remains bootable | L3/L4 |
| OFN-X-006 | Clock/time discontinuity during a fault | No negative time deltas; no divide-by-zero from a zero delta; no scheduler corruption | L3 |
| OFN-X-007 | Telemetry path saturated while faults are being reported | Fault reporting is prioritized over routine telemetry | L4/L5 |
| OFN-X-008 | Recovery leaves resources leaked | Repeat any recovery path ≥1000×; verify no monotonic resource growth | L2/L3 |

---

## 15. Change Control and Regression After First Integration

Once a CSCI has entered hardware integration, every subsequent delivery is governed by impact
analysis. The question is never "did we test it" but "what verification credit did this change
invalidate."

### 15.1 Impact Analysis (required with every post-HIRR delivery)

For each change in the delivery, record:

| Field | Content |
|---|---|
| Change ID | DR/CR number |
| Type | Defect correction / capability / obsolescence / tool or environment |
| Modules touched | CSU/CSC list with diff stat |
| Requirements affected | Requirement IDs, added/changed/removed |
| Interfaces affected | ICD IDs and revisions; whether the change is interface-breaking |
| Resources affected | Flash, RAM, stack, CPU deltas |
| Timing affected | Yes/no with analysis |
| Verification invalidated | Test case IDs whose credit is void |
| Regression scope | Test case IDs to re-run, with the level for each |
| Peers affected | Peer CSCIs, firmware, or hardware requiring re-verification |

### 15.2 Regression Tiers

| Tier | Content | Duration target | When |
|---|---|---|---|
| **T0 — Smoke** | Boot, version readback, one message per bus, health status. Implemented as the automated C0–C4 ladder (§19.6) | < 30 min | Every load, every level, every time |
| **T1 — Targeted** | T0 + the invalidated test cases from the impact analysis | < 4 h | PATCH deliveries |
| **T2 — Functional** | T1 + full nominal functional suite at the current level | < 24 h | MINOR deliveries |
| **T3 — Full** | T2 + full off-nominal matrix + soak | ≥ 1 week | MAJOR deliveries, interface changes, or return from a long gap |

Default to the higher tier when the analysis is uncertain. The cost asymmetry is extreme.

### 15.3 Change Classes and Authority

| Class | Definition | Approval required |
|---|---|---|
| **I** | Interface-breaking, or affects a safety function, an inhibit, or a hazard control | Change Control Board + Safety; full re-verification plan |
| **II** | Functional change with no interface break | Software CCB; T2 regression minimum |
| **III** | Defect correction, no interface or functional change | Software Lead + SQA; T1 regression minimum |
| **IV** | Non-code (comments, docs, build scripts with no artifact change) | Software Lead; T0 plus reproducible-build verification proving the binary is unchanged |

**Post-freeze (after Step 10):** all changes are Class I regardless of technical content. The
question is no longer engineering risk alone but program risk.

### 15.4 Environment and Tool Changes

A change to the toolchain, the RTOS, the test environment, a simulator, or a bus model is a
configuration change and is subject to the same analysis. A recompile with a different compiler
version produces a different artifact and requires, at minimum, T2. Document why the change is
necessary — "the build agent was upgraded" is not a plan.

---

## 16. Defect and Anomaly Management

### 16.1 Severity

| Severity | Definition | Gate effect |
|---|---|---|
| **Sev 1** | Could cause loss of system, loss of life, or violation of a safety inhibit | Blocks every gate. Stop-work on the affected path. |
| **Sev 2** | Could cause loss of mission or loss of a primary function; no workaround | Blocks every gate |
| **Sev 3** | Degrades a function; a workaround exists | Blocks L5+ gates unless dispositioned with rationale |
| **Sev 4** | Minor; cosmetic; documentation | Tracked; does not block |
| **Sev 5** | Enhancement request | Backlog |

### 16.2 Anomaly Workflow During Integration

1. **Stop and preserve.** On an unexpected result, do not re-run and do not power-cycle before the
   state is captured. Bus captures, memory dumps, logs, and register state are perishable.
2. **Record the configuration.** Exact versions of everything, plus environmental conditions and
   the step in the procedure where it occurred.
3. **Assess safety.** Determine whether it is safe to continue. When in doubt, safe the article.
4. **Write the anomaly report** before troubleshooting begins, so the observation is recorded
   separately from the hypothesis.
5. **Classify**: software, hardware, firmware, harness, test equipment, procedure, or operator. Note
   that "test equipment" and "procedure" are common and frequently misdiagnosed as software.
6. **Reproduce.** An anomaly that cannot be reproduced is not closed — it is deferred with a written
   rationale and a monitoring plan. Unreproducible anomalies on mission hardware are a serious
   finding, not a nuisance.
7. **Root cause**, not symptom. Identify why it was not caught at a lower level — that gap is itself
   a process defect to correct.
8. **Correct, verify, and regress** per §15.
9. **Close** with the gate authority that owns the affected level.

### 16.3 Escape Analysis

For every defect found at L4 or above, answer: *at what level should this have been found, and why
wasn't it?* Feed the answer back into the lower-level test suites. A process that does not do this
will keep paying full price for the same class of defect.

---

## 17. Roles and Review Boards

| Role | Responsibility |
|---|---|
| **Software Lead (per CSCI)** | Owns the CSCI, its DRP, and its verification argument |
| **Software Architect** | Owns the CSCI decomposition, task model, and internal interfaces |
| **Card Lead** | Owns the card HWCI and the Card Software Load Set composition |
| **Box / LRU Lead** | Owns the integrated box configuration and backplane behavior |
| **Integration & Test Lead** | Owns the integration plan, procedures, lab configuration, and gate execution |
| **Systems Engineering** | Owns requirements, ICDs, and the allocation of verification to levels |
| **Safety** | Owns hazard analysis, inhibit verification, and Class I change concurrence |
| **SQA** | Witnesses formal tests, audits records, verifies process compliance |
| **CM** | Owns baselines, tags, the compatibility matrix, and configuration verification |
| **Mission Assurance** | Owns the overall verification argument at system and mission level |
| **Test Conductor** | Runs the procedure; the sole authority to deviate in real time, within defined bounds |

**Gate summary:**

| Gate | Chair | Authorizes |
|---|---|---|
| CSC Verification Complete | Software Lead | CSC entry into CSCI integration |
| CSCI Build Integration Complete | Software Lead | Entry into formal CSCI verification |
| CSCI Qualification Review (CQR) | Software Lead | CSCI release; entry to HIRR |
| **HIRR** | I&T Lead | **Applying power to mission-like hardware** |
| Card Integration Complete (CIC) | Card Lead | Entry to box integration |
| Box Integration Complete (BIC) | Box Lead | Entry to qualification / system integration |
| Box Qualification Complete (BQC) | Box Lead | Mission-path use of the box |
| System Integration Complete (SIC) | System I&T Lead | Entry to formal qualification test |
| Formal Qualification Test Complete (FQT) | System I&T Lead | Entry to mission readiness |
| Mission Readiness Review (MRR) | Program | Mission |

---

## 18. Lab Architecture and Tooling

This section defines how a card is exercised on the bench at L2 and L3, before a chassis exists. It
is written for a lab being built from nothing; §18.11 gives the build-out order and the long-lead
items.

### 18.1 Test Articles That Run on a Card

A CSU does not run on a card. A CSU is a function or module verified on the host with stubs at L0 —
there is nothing to load. What runs on a standalone card is one of four articles. They are named
separately here because they carry different configuration status and earn different credit.

| # | Article | Contents | CM status | Verification credit |
|---|---|---|---|---|
| 1 | **Bring-up image** | Boot, debug UART, memory test, one bus initialized at a time. No application. | Tagged, but not a CSCI | None — diagnostic aid |
| 2 | **CSC-on-target harness** | One CSC (e.g. the 1553 driver) plus a test `main()`, driving real silicon | Tagged; harness is its own repo item | Supporting evidence only |
| 3 | **Instrumented CSCI** | The release image plus extra telemetry, trace, and test hooks | Tagged; hooks listed in the test hook register (4.E17) | Supporting evidence only |
| 4 | **Release CSCI** | The delivered image, hooks in their mission state | Full DRP (§2.4) | **All L3 credit** |

**Rules:**
- Every article above is version-controlled and reproducibly built. "It worked on the bring-up
  image" is a claim someone will make at 2 a.m., and it must be resolvable to a SHA.
- Articles 1–3 exist to make debugging tractable, not to substitute for verification. A defect found
  with article 2 is fixed and then re-verified with article 4.
- Article 2 is the highest-value one during bring-up. A driver harness that exercises one bus with
  nothing else running localizes defects that are nearly undiagnosable inside a full CSCI.
- The delta between article 3 and article 4 is itself a risk. Record it, and run the L3 exit suite
  on article 4.

### 18.2 What the Backplane Supplies Beyond Buses

A card pulled from its chassis loses more than its data interfaces. Before any bus work is possible,
the fixture must supply:

| Category | Typical content | Failure if omitted |
|---|---|---|
| **Power** | Each rail at correct voltage, sequence, and ramp rate; correct current limits | Card does not boot, or boots into an undefined state |
| **Slot identity** | Geographic address / slot ID pins, often strapped per slot | Card boots but claims the wrong identity; bus addresses wrong |
| **Presence and enable** | Backplane-present, card-enable, inhibit discretes | Card holds in reset with no diagnostic |
| **Reset** | Chassis reset distribution, power-on reset timing | Indeterminate reset behavior; intermittent boot |
| **Clocks** | Backplane reference clock, PCIe REFCLK, sync distribution | PCIe never trains; timing tests meaningless |
| **Termination and bias** | 1553 bus termination, I²C pull-ups, differential biasing | Marginal signaling that passes on the bench and fails in the box |
| **Ground return** | Correct ground topology, not a single wire | Noise-induced defects that look like software |

The single most common standalone bring-up failure is a card that will not boot because a slot
address pin floats or a presence discrete is deasserted. **The fixture's pin-level intent belongs on
a drawing, not in a technician's memory.**

### 18.3 Fixture Options

| Option | What it is | Strengths | Limits | Use for |
|---|---|---|---|---|
| **Extender card** | UUT plugs into an extender; extender plugs into a chassis slot | Best electrical fidelity; live probe access with the card in its real environment | Lengthens every trace. Fine for discretes and I²C, marginal for 1553 stub length | Probing and scope work, not primary test |
| **Single-slot test backplane** | Custom fixture replicating one slot's pinout, routing each bus to external connectors and supplying §18.2 | Full control of terminations, pull-ups, and strapping; each bus reaches proper instrumentation | Not the real backplane: no contention, no cross-card coupling | **Primary L3 fixture** |
| **Bench chassis with emulator cards** | Real chassis, UUT in its real slot, peer slots populated with emulator cards | Highest fidelity; the only honest way to exercise backplane arbitration and cross-card coupling | Expensive; blurs the L3/L4 boundary | Late L3 / early L4. Not required for PCIe — see §18.8. |
| **Breakout panel** | Patch panel terminating every net to labeled test points | Fast reconfiguration; readable by technicians | Adds stubs; keep it downstream of the test backplane | Routing flexibility during a campaign |

**Recommended:** single-slot test backplane as the workhorse, an extender on the shelf for probing,
and a bench chassis only if backplane arbitration work at late L3 justifies it.

Connector guidance for the test backplane: twinax for 1553, RJ45 or SFP for Ethernet, D-sub for
serial and discretes, SMA for clocks and sync, and separate banana or lug connections per power
rail with individual current monitoring. **PCIe is not broken out** — see §18.8.

### 18.4 The Fixture Is a Configuration Item

Test backplanes, extenders, breakout panels, harnesses, and the termination and pull-up values
installed on them are configuration items with part numbers and revisions.

| Requirement | Rationale |
|---|---|
| Fixture drawing with revision, including every strap, termination, and pull-up value | A bench with 4.7 kΩ I²C pull-ups where the box has 2.2 kΩ produces timing results that do not transfer |
| Fixture serial number recorded in every test record | Two fixtures built to the same drawing are not identical until proven so |
| Continuity and isolation verified before first use and after any rework | Fixture rework between campaigns is a silent source of irreproducible results |
| Deviations from the backplane drawing documented explicitly | These become the fidelity statement in §18.10 |

A test result that cannot name the fixture revision it was obtained on is not traceable evidence.

### 18.5 Three-Layer Bench Architecture

Split the bench by layer, not by bus. Each layer has different requirements, different tooling, and
different owners.

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  SEQUENCER LAYER                                                 │
  │  Owns the procedure. Drives both layers below over a network     │
  │  API. Emits machine-readable pass/fail. Timestamps everything    │
  │  to a common time base. Runs in CI.                              │
  └───────────────┬──────────────────────────────┬───────────────────┘
                  │                              │
  ┌───────────────▼──────────────┐  ┌────────────▼───────────────────┐
  │  MESSAGE LAYER               │  │  INSTRUMENTATION LAYER         │
  │  Linux servers.              │  │  PXI / bench instruments.      │
  │  Protocol and message-level  │  │  Physical and electrical       │
  │  simulation: 1553 BC/RT      │  │  stimulus: discretes with      │
  │  behavior, Ethernet peers,   │  │  level translation, power      │
  │  serial protocol peers,      │  │  ramps and interrupts, clocks, │
  │  I²C target behavior.        │  │  sync pulses, measurement.     │
  │  Generated from the ICD.     │  │  Instrument orchestration.     │
  │  Text source, code-reviewed, │  │  Technician-facing panel.      │
  │  in version control, in CI.  │  │                                │
  └──────────────┬───────────────┘  └────────────┬───────────────────┘
                 │                               │
                 └───────────┬───────────────────┘
                             ▼
              ┌──────────────────────────────┐
              │   TEST BACKPLANE / FIXTURE   │
              │        (§18.3, §18.4)        │
              └──────────────┬───────────────┘
                             ▼
                     ┌───────────────┐
                     │  CARD UNDER   │
                     │     TEST      │
                     └───────────────┘
```

**Why the split matters.** Graphical instrumentation environments are strong at tightly-timed
digital and analog I/O, power sequencing, and giving a technician a usable panel at the bench. They
are weak at message-level simulation with real state machines, at text diffs and code review, and at
running unattended in CI. A 1553 RT simulator that lives in a graphical VI will not get under
automated regression and will not be maintained by the software team.

Conversely, a Linux box cannot drive a 28 V discrete with a 50 µs edge requirement or ramp a supply
through a brownout threshold on a scripted profile.

Put each where it is strong. The sequencer is the only thing that talks to both.

**Two rules that outrank the tool choice:**

1. **Retarget the L1 models; do not rewrite them.** The bus models written for SIL at L1 should be
   retargeted to drive real hardware at L2/L3, not reimplemented. Two implementations of the same
   ICD will diverge, and the divergence surfaces at L4 where it is expensive.
2. **The sequencer runs the same test cases at L1, L2, and L3.** The transport underneath changes;
   the case definitions do not. This is what makes the regression tiers in §15.2 achievable, and it
   is the foundation the automated checkout cell in §19.3 is built on.

### 18.6 The ICD as Single Source of Truth

A machine-readable ICD should be the generation source for every representation of every message:

```
                    ┌──────────────────────────┐
                    │   MACHINE-READABLE ICD   │
                    │   (schema / message DB)  │
                    │   versioned, hashed      │
                    └────────────┬─────────────┘
                                 │  generate
        ┌────────────────┬───────┴────────┬──────────────────┐
        ▼                ▼                ▼                  ▼
  Onboard message   Bench simulator   Analyzer message   Test case
  structures &      structures &      database           scaffolding &
  encode/decode     peer behavior     (1553, Ethernet)   range/limit
                                                         checks
```

This closes the transcription defect class: a hand-written bench simulator that interprets the ICD
slightly differently from the onboard code will pass its own tests and fail at integration.

**Requirements:**
- The ICD source is versioned and hashed; the hash appears in the DRP (`interfaces.message_db_sha256`)
- Every generated artifact records the ICD version and hash it came from
- The bench refuses to run if its generated artifacts disagree with the CSCI's declared ICD revision
- Generators are themselves configuration-managed and their output is reproducible

### 18.7 Instrumentation by Interface

| Interface | Stimulus / peer | Fault injection capability needed | Buy or build |
|---|---|---|---|
| **Discretes / GPIO** | DAQ or PXI DIO with level translation to the card's discrete standard (3.3 V, 5 V, 28 V). Must both drive inputs and *sense* outputs with edge timing. | Stuck high/low, open, indeterminate band, bounce, runt and jittered sync pulses | Buy DIO; build the level-translation and 28 V driver interface |
| **I²C** | Microcontroller acting as a programmable target, emulating sensors, EEPROMs, and power monitors | Hold SDA low, stretch clock past spec, NAK on command, address collision, glitch injection | **Build.** A USB adapter handles nominal traffic but cannot do any of the fault cases in §14.2 |
| **Serial (RS-422/485)** | Linux host with proper differential transceivers; protocol peer in software | Framing, parity, break, truncation, bus contention, transmitter stuck enabled | Buy transceivers; build the protocol peer |
| **1553** | Commercial interface card with BC, RT, and bus monitor modes | **Error injection option — Manchester, parity, sync, word count, timing.** License it at purchase. | **Buy.** No substitute exists. Long lead. |
| **Ethernet** | Linux server with a capable NIC; managed switch for VLAN and storm behavior; raw socket / crafted frame tooling | Loss, latency, jitter, reorder, duplication, bad CRC, malformed frames, flood | Buy the impairment device and switch; build the peer and crafting tooling |
| **PCIe** | The real COTS card in a real slot or on a qualified cable; configuration space access tooling | Driver API error and stall injection; endpoint power/reset switching; kernel AER injection where the BSP supports it. No protocol-layer injection — see §18.8 | **Build** the driver API fault harness; no instrument purchase |
| **Power** | Programmable supplies with scriptable ramp, interrupt, and current-limit profiles; per-rail current monitoring | Brownout ramps, interrupts swept 1 ms–1 s, slow ramp, power loss during flash write | Buy |
| **Clocks / sync** | Programmable clock source; pulse and delay generator | Missing, early, late, jittered, doubled, and runt sync pulses; clock loss and drift | Buy |
| **Harness faults** | Fault box with per-net open, short-to-rail, short-to-ground, and pin-swap | Applied at the fixture boundary | Build |
| **Measurement** | Oscilloscope with adequate bandwidth and deep memory; current probes; logic analyzer with protocol decode | — | Buy |

### 18.8 PCIe — COTS Scope on the Bench

Because the PCIe cards, stack, and driver are COTS and are not verified by this process (§14.6), the
bench requirement collapses dramatically. This is the single largest cost reduction available to the
lab build-out.

| Not required | Why |
|---|---|
| PCIe protocol analyzer / exerciser | Verifies the vendor's link implementation, which is out of scope |
| PCIe endpoint emulator | Same |
| PIPE-level or lane-masking injector | Degraded-width detection is tested by clamping configuration space instead |
| Bench chassis procured *on PCIe's account* | A chassis may still be wanted for backplane arbitration at late L3, but PCIe no longer forces the purchase |

| Required | Purpose |
|---|---|
| The real COTS card in a real slot, or on a qualified cabled adapter | Communication testing needs a working link, not an instrumented one |
| Configuration space access tooling | Read enumeration, vendor revision, link width and speed; clamp width/speed for OFN-PCIE-002 |
| Driver API fault injection harness | OFN-PCIE-007 and -008 — the largest and most valuable part of the set |
| Switchable endpoint power and reset | OFN-PCIE-003, -004, -005, -006 |
| Kernel / platform error injection, where the BSP supports it | OFN-PCIE-009 and -010. Confirm BSP support early; if absent, these close by counter trending and analysis. |
| Application-layer throughput and latency measurement | NOM-PCIE-002 and OFN-PCIE-016 |

**Bench topology consequence — do not break out PCIe.** The signal integrity problem that makes
backplane PCIe unsuitable for a connectorized fixture disappears once verification of the link is
out of scope. Seat the card in a real slot or use a qualified PCIe cable, and break out only the
buses you *are* verifying — GPIO, I²C, serial, 1553, Ethernet. This removes the hardest constraint
from the test backplane design (§18.3) and should be reflected in the fixture drawing.

**The vendor stack is a third-party component in a mission-critical path.** This program owns none
of it and can fix none of it, which changes what the process owes:

- The driver and stack appear in the DRP `third_party` block with an exact version and a
  qualification record, per §2.7
- A **known-good driver/stack version is pinned**, and the version is asserted at startup
  (OFN-PCIE-014)
- A vendor escalation path exists and is named before integration begins, not after the first
  anomaly
- Where a vendor defect is found and cannot be fixed, the response is a software-side workaround
  with its own verification, plus a documented risk acceptance — not a schedule assumption that the
  vendor will patch

**What this does not remove.** Everything above the stack is still verified: message content and
rates, throughput and latency budgets, the end-to-end integrity checks required by §14.6, and this
software's response to every error the vendor stack can report. That last item is the largest single
block of PCIe test effort remaining, and it runs at L2 on a driver stub before the card exists.

### 18.9 Observability, Time Base, and Data Capture

**Minimum observability on a standalone card:**

| Access | Purpose |
|---|---|
| Debug UART, always populated | First signal of life; boot progress; panic output. Non-negotiable. |
| JTAG / SWD | Halt-mode debug, image load, memory inspection, recovery from a bad load |
| Dedicated test telemetry port | Structured health and trace output independent of the mission interfaces |
| Per-rail current monitoring | Detects latch-up, unexpected activity, and inrush deviations |
| Bus taps on every interface | Capture is passive and always on during a run, not enabled after a failure |

**Common time base.** Every instrument, simulator, and capture device timestamps to one time base,
distributed by a shared trigger line or PTP. Without it, correlating a 1553 error at the analyzer
with a UART message and a current spike is guesswork. This is an early build-out item, not a later
refinement — retrofitting it means re-running campaigns.

**Data capture:**
- Everything recorded by default, for the whole run, not started when something looks interesting
- Analyzer captures retained as **data files**, never screenshots
- Each run's record carries: test case ID, article version and SHA, fixture drawing and revision,
  instrument models, serials, and calibration dates, and the ICD version in use
- Automated post-run analysis produces machine-readable pass/fail; manual review is for
  investigation, not for adjudication

### 18.10 Fidelity Limits of a Standalone Card Bench

A card on a test backplane is not a card in a box. These deltas must be written down, because each
is a reason a test passes at L3 and fails at L4:

| Delta | Consequence |
|---|---|
| Stub lengths and termination differ | 1553 and high-speed signaling margins differ; marginal signaling may pass here and fail there |
| No neighboring cards | No backplane contention, no arbitration, no cross-card coupling or EMI |
| Ground return topology differs | Noise behavior differs; some defects appear only in the chassis |
| Power source impedance and sequencing differ | Inrush, brownout, and sequencing behavior are not representative |
| Thermal environment differs | Timing margins measured at bench ambient, not at box hot soak |
| Simulated peers are not real peers | A simulator implements the ICD; a real card implements its own reading of it |

**Rule:** an L3 pass is credit for the requirements assigned to L3, and evidence only for anything
above it (§20.2). Requirements whose verification depends on any delta in the table above are
assigned to L4 or higher from the start, not migrated down to recover schedule.

### 18.11 Build-Out Sequence for a New Lab

Ordered by dependency and lead time. The items in **bold** are long lead and should be on purchase
orders before the first card arrives.

| Phase | Trigger | Acquire / build |
|---|---|---|
| **0 — Before first card** | Program start | **Test backplane fabrication** (custom PCB, expect 8–12 weeks including respin); programmable power supplies; DAQ/DIO with level translation; oscilloscope; JTAG probes; Linux build and sequencer servers; ICD generators (§18.6); sequencer skeleton running L1 cases |
| **1 — First card bring-up** | Card delivery | **1553 interface card with error injection option**; managed switch; Ethernet peer server; serial transceivers; I²C MCU target (build); extender card |
| **2 — Full off-nominal campaign** | Entering §14 matrix | Ethernet impairment device; programmable clock source; pulse/delay generator; harness fault box (build); I²C glitch injection (extend the MCU target); current probes; logic analyzer |
| **3 — Box integration** | L4 | Bench chassis and emulator cards *if* backplane arbitration work justifies them; switchable endpoint power/reset for the PCIe error set; thermal chamber access |
| **4 — System integration** | L5 | HWIL simulation host; real-time dynamics models; multi-box harness; expanded recording capacity |

**Buy vs build.** Buy the 1553 interface, the impairment device, the supplies,
and the measurement instruments — these are commodity capability where in-house effort buys nothing.
Build the test backplane, the I²C programmable target, the harness fault box, the discrete level
translation, the message-layer simulators, and the sequencer — these are all specific to your ICD
and your fixture, and no vendor will maintain them for you.

**The first thing to build is the sequencer**, before any hardware exists, running L0 and L1 cases
against simulated interfaces. If it arrives after the hardware, the bench will be driven manually,
the regression tiers in §15.2 will not be met, and the campaign will be paced by technician time.
See §19.12 for the full automation build order, of which the sequencer is item 1.

### 18.12 Capability Summary by Level

| Category | Capability required | Used at |
|---|---|---|
| **Bus analysis** | 1553 bus analyzer/emulator with BC, RT, and monitor modes **and error injection**; Ethernet capture at line rate with hardware timestamping; I²C/serial analyzers; logic analyzer with protocol decode | L3–L5 |
| **Fault injection** | 1553 error injector (Manchester, parity, sync, timing); Ethernet impairment device (loss, latency, jitter, reorder, duplication); driver-API fault injection harness for COTS interfaces; I²C glitch and bus-hold injector; harness fault box (open, short, swap) | L3–L5 |
| **Stimulus** | Programmable power supplies with ramp/interrupt scripting; programmable clock sources; discrete stimulus with programmable levels and timing; delay generators | L3–L4 |
| **Measurement** | Oscilloscope with sufficient bandwidth and deep memory; current probes; thermal imaging or instrumented thermal monitoring | L3–L4 |
| **Fixtures** | Single-slot test backplane, extender card, breakout panel, bench chassis — each a configuration item per §18.4 | L2–L4 |
| **Emulation** | Processor emulator or eval board matching mission silicon; simulated HAL; validated bus models retargeted from L1 | L1–L2 |
| **Environment** | Thermal chamber, thermal-vacuum, vibration table, shock, EMI/EMC chamber, radiation beam access | L4+ |
| **HWIL** | Real-time system dynamics simulation with validated sensor and effector models, interfaced to the mission buses at mission rates | L5 |
| **Data** | Centralized time-synchronized recording across all buses, telemetry, and sim state; retention and retrieval policy; automated post-run analysis | L3–L6 |
| **Automation** | Scripted execution of procedures with machine-readable pass/fail results, so the regression suites in §15 are actually runnable at their stated durations | L1–L5 |

### 18.13 Equipment Configuration Control

**Calibration and configuration control:** test equipment is a configuration item. An
out-of-calibration analyzer or an unrecorded impairment-device setting invalidates the result. Record
equipment model, serial number, calibration date, and configuration in every test record.

**Test equipment as a defect source:** at L3–L4, a meaningful fraction of apparent software anomalies
are test equipment, fixture, or procedure defects. Design the setup so that the test equipment can be
independently verified — a loopback, a known-good reference article, or a second independent
measurement. Budget for this explicitly; a bench with no way to prove itself correct will burn
software engineering time on hardware problems.

## 19. Automation and Hardware-in-the-Loop CI

### 19.1 Objective and the Automation Contract

**Target state:** a card sits in a fixture, permanently harnessed and powered under program control. A job is submitted naming a CSCI by its DRP manifest and a checkout suite. With no human touching anything, the system loads the image, verifies the load, runs the suite, produces a machine-readable verdict with full evidence, and leaves the card in a known safe state — whether it passed or failed.

Everything in this section serves that contract. State it precisely, because it is what the tooling is measured against:

```
  INPUT                             OUTPUT
  ─────                             ──────
  DRP manifest URL + SHA-256   →    verdict: PASS | FAIL | ABORT | CELL_FAULT
  suite ID                     →    per-case results, machine-readable
  fixture constraints          →    all captures, logs, and measurements
                               →    the exact configuration under test
                               →    card left in a defined safe state
```

Note the fourth verdict. **`CELL_FAULT` is not a failure of the software under test** — it means the cell could not produce a trustworthy result. Conflating that with `FAIL` is the most common way automated hardware testing loses the team's trust (§19.8).

### 19.2 Automation Coverage by Level

Full automation is achievable at some levels and dishonest to claim at others. Plan against real numbers.

| Level | Realistic automation | Automated | Necessarily manual |
|---|---|---|---|
| **L0** | 100% | Unit tests, static analysis, coverage, stack and WCET analysis, build reproducibility check | — |
| **L1** | 100% | Full CSCI integration suite against simulated HAL; profiling; schedulability; soak | — |
| **L2** | ~95% | Everything at L1 re-run on target silicon in a cell; timing measurement; comparison of host vs target results | First bring-up on new silicon; toolchain investigation |
| **L3** | ~85% | Load, checkout ladder, per-bus nominal and off-nominal, timing measurement, soak, power cycling | First power-on of a new card *or fixture* revision; physical fixture reconfiguration; scope-probe investigation |
| **L4** | ~60% | Functional and bus suites, degraded-mode cases, thermal-plateau functional runs, data collection | Card population changes; chamber setup and teardown; some backplane fault injection; anomaly investigation |
| **L5** | ~50% | Scenario execution, Monte Carlo dispersions, fault scenario campaign, data collection and post-run analysis | Configuration changes between campaigns; anomaly investigation; scenario authoring |
| **L6** | ~30% | Data collection, automated limit checking, post-run analysis and report generation | Execution is witnessed; adjudication is human; safety-critical inhibit verification is witnessed by requirement |

**The lever is L2 and L3.** That is where the volume of regression lives, where every delivery must re-run, and where manual execution silently caps how often you can afford to integrate. Automating L3 to 85% is worth more than pushing L5 from 50% to 60%.

### 19.3 The Card Checkout Cell

A cell is one card, one fixture, and everything needed to drive it unattended.

```
   ┌─────────────────────────────────────────────────────────────────┐
   │  CELL CONTROLLER  (one per cell)                                │
   │  • accepts jobs, owns the run, enforces one-job-at-a-time       │
   │  • drives load, suite execution, safing, artifact capture       │
   │  • publishes verdict + evidence; never adjudicates by hand      │
   └───┬───────────────┬──────────────┬──────────────┬───────────────┘
       │               │              │              │
  ┌────▼─────┐  ┌──────▼──────┐  ┌────▼──────┐  ┌────▼────────────┐
  │ POWER    │  │ LOAD PATH   │  │ MESSAGE   │  │ INSTRUMENTATION │
  │ control  │  │ • operational│ │ LAYER     │  │ LAYER           │
  │ • prog.  │  │   loader     │ │ (§18.5)   │  │ (§18.5)         │
  │   supply │  │ • JTAG as    │ │ bus peers │  │ discretes,      │
  │ • per-   │  │   RECOVERY   │ │ from ICD  │  │ clocks, sync,   │
  │   rail I │  │   only       │ │ (§18.6)   │  │ measurement     │
  │ • abort  │  │ • boot-slot  │ │           │  │                 │
  │   limits │  │   strap ctrl │ │           │  │                 │
  └────┬─────┘  └──────┬───────┘ └────┬──────┘  └────┬────────────┘
       └───────────────┴──────────────┴──────────────┘
                              │
                    ┌─────────▼──────────┐
                    │  TEST BACKPLANE    │
                    │  (§18.3, §18.4)    │
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │  CARD UNDER TEST   │
                    │  permanently       │
                    │  harnessed         │
                    └────────────────────┘
```

**What makes a cell unattended-capable** — each of these is a hard requirement, not a nicety:

| Capability | Why |
|---|---|
| Programmable power with scripted sequencing, current limit, and hard abort threshold | The cell must be able to cut power on its own judgment without a human present |
| Fixture-controlled boot-slot strap | Lets the controller force a boot from the golden image after a bad load |
| JTAG permanently attached | The only recovery path that does not depend on the card being alive |
| Always-on passive capture on every bus | A failure that requires a re-run to capture is a failure you may not reproduce |
| Per-rail current monitoring with trend limits | Detects latch-up and abnormal activity before damage |
| Thermal monitoring with an independent cutoff | Unattended running with no thermal interlock is how you lose a card overnight |
| Card permanently harnessed | Reconnection between runs reintroduces the human and the variability |

### 19.4 Automated Load and Load Verification

**Use the operational load path, not JTAG.** The load mechanism that must work in the field is the one under test. Loading via JTAG for convenience means the bootloader and loader software are never exercised, and their defects surface at L4 or later. JTAG is the recovery path only.

**Always write the inactive slot.** Load into slot B while A runs, verify, then switch the boot selection. Never overwrite the running image, and never write the golden slot as part of routine automation — that is a separately authorized operation with its own procedure.

**Three independent confirmations before a load is declared good.** The programmer reporting success is not one of them:

| # | Check | Compared against |
|---|---|---|
| 1 | Flash readback, hashed | `artifacts[].sha256` in the DRP manifest |
| 2 | Version reported by the running image | `csci.version` and `provenance.commit_sha` in the manifest |
| 3 | Card's own boot-time image CRC or signature check | Card's self-report, which must agree with 1 and 2 |

If any of the three disagrees, the run aborts and safes. It does not proceed and report a warning.

**The load itself is a test case.** Load time, retry count, and success rate across ≥ 50 consecutive cycles are recorded results feeding 5.X1, not setup steps.

### 19.5 Recovery and Safe States

Unattended automation must be able to dig itself out. Define the ladder and implement all of it:

| Failure | Automated recovery |
|---|---|
| Load fails verification | Abort, do not switch boot selection, card still boots the prior image |
| Card boots but hangs | Power cycle; retry once; on second failure escalate |
| Card does not boot the new image | Assert the golden-boot strap, power cycle, boot golden, report |
| Card does not boot golden either | JTAG recovery: halt, re-flash golden, verify, power cycle |
| JTAG recovery fails | **Stop. Safe the card. Mark the cell `CELL_FAULT` and quarantine it.** Do not retry, do not run further jobs. |
| Current exceeds the abort threshold at any point | Immediate power removal, regardless of test state; record the trace leading up to it |
| Thermal cutoff | Immediate power removal; cell quarantined pending inspection |

**Terminal safe state** is defined and is what the cell leaves behind on every path: power removed or at a defined standby, all stimulus outputs in their safe state, all captures flushed and stored, and the fixture's state recorded. A cell that exits a failed run with the card powered and the discretes in an arbitrary state is not safe to leave unattended.

### 19.6 The Checkout Ladder

Checkout runs as a graded ladder. Each tier gates the next; a failure stops the ladder and safes. Running functional tests on a card that has not passed power and boot checks produces noise, not data.

| Tier | Name | Content | Typical duration | On failure |
|---|---|---|---|---|
| **C0** | Pre-power | Fixture continuity and isolation check, strap verification, instrument presence and calibration check | < 1 min | Abort — `CELL_FAULT` |
| **C1** | Power-on | Current envelope vs predicted, rail sequencing, inrush | < 1 min | Abort — safe immediately |
| **C2** | Boot and identity | Boot completes; version readback for CSCI, firmware, bootloader; compared to the manifest and the compatibility matrix | < 2 min | Abort — recovery per §19.5 |
| **C3** | Memory and image | RAM test, flash readback hash, EDAC functional | < 5 min | Abort |
| **C4** | Bus smoke | One nominal exchange per bus, in the §22 bring-up order | < 5 min | Stop ladder, report |
| **C5** | Functional | Full nominal functional suite at rate, all buses concurrent, timing measured | 1–4 h | Stop ladder, report |
| **C6** | Off-nominal | The §14 matrix at the scope assigned to this level | 4–24 h | Continue and report; individual case failures do not stop the ladder |
| **C7** | Soak | Sustained operation at rate with resource trending | 24–72 h | Report |

**C0 through C4 is the smoke suite** referenced in §15.2 tier T0, and it is the thing to automate first. A 15-minute automated gate that runs on every load catches the majority of integration breakage and is what makes frequent integration affordable.

**C6 behaves differently by design.** Off-nominal cases are independent; one failing does not invalidate the rest, and stopping the ladder would waste a long unattended window. Nominal tiers gate; off-nominal tiers accumulate.

### 19.7 Cell API and CI Integration

The cell is a service. Everything else — the CI system, the sequencer, a developer at a terminal — is a client. This is what makes the same suite runnable from a commit hook and from a formal witnessed run.

```
POST /runs
{
  "manifest_url":    "registry://drp/msnproc/2.4.1/drp-manifest.yaml",
  "manifest_sha256": "...",
  "suite":           "C0-C4",
  "fixture":         { "card_pn": "PN-40012", "card_rev": "Rev D", "any_of": ["cell-03","cell-07"] },
  "purpose":         "regression",        // regression | formal | investigation
  "requested_by":    "<ci-job-id>"
}
→ 202 { "run_id": "...", "queued_behind": 2 }

GET /runs/{run_id}
→ {
    "state":   "complete",
    "verdict": "PASS",                    // PASS | FAIL | ABORT | CELL_FAULT
    "cases":   [ { "id":"IT-L3-1553-007", "result":"PASS", "measured":{...} }, ... ],
    "config_under_test": {
      "csci": {"name":"msnproc","version":"2.4.1","commit_sha":"..."},
      "firmware": "3.3.0", "bootloader": "1.9.0",
      "card": {"pn":"PN-40012","rev":"Rev D","sn":"SN0114"},
      "fixture": {"drawing":"FIX-2210","rev":"C","sn":"F007"},
      "icd_version": "...", "instruments": [ {...calibration dates...} ]
    },
    "artifacts": [ "captures/...", "logs/...", "measurements/..." ],
    "canary": { "ran_at":"...", "verdict":"PASS" }
  }
```

**Rules:**
- One job per cell at a time, enforced by the controller, not by convention
- Jobs are queued and idempotent; a resubmitted job with the same inputs returns the same run
- `purpose: formal` runs additionally require a witness identity and refuse to start without one
- The CI system submits `C0-C4` on every merge to `main` and `C0-C6` nightly; `C0-C7` runs on release candidates
- **Verdicts are never edited by hand.** A disputed verdict is investigated with a new run, not overwritten.

### 19.8 Proving the Cell: Canary Runs and Seeded Defects

This subsection is what separates automation people trust from automation people route around.

**Canary runs.** Before any job whose result will be believed — at minimum at the start of every shift and after any fixture touch — the cell loads a **pinned known-good CSCI** and runs C0–C4. If the canary fails, the cell is marked `CELL_FAULT`, jobs are quarantined, and **no verdicts are published** until it is resolved.

Without this, the first bad fixture connection produces a stream of red results that the software team spends two days chasing. It happens on every program that skips this step.

**Seeded-defect verification.** A checkout suite that has never failed on a bad image has not been shown to work. On a schedule — weekly, and after any change to the suite — run deliberately broken images and confirm the suite catches each:

| Seeded defect | Suite must catch at |
|---|---|
| Wrong version string in the image | C2 |
| Corrupted image (single bit flipped) | Load verification, §19.4 |
| Image built for the wrong card revision | C2 via compatibility check |
| One bus initialization disabled | C4 |
| A timing budget deliberately exceeded | C5 |
| An off-nominal response deliberately removed | C6, the specific case |

A miss is a defect against the checkout suite with the same severity as a defect against the CSCI it was supposed to catch.

**The suite and the cell controller are configuration items.** Version-controlled, peer-reviewed, reproducibly built, and released with their own version recorded in every run record. Test software that adjudicates mission software can be wrong, and when it is wrong it is wrong silently.

### 19.9 Flake Policy

Hardware tests are non-deterministic in ways host tests are not. Handle it explicitly, or the team will handle it implicitly by ignoring red.

1. **No silent retry.** A retry is permitted where the procedure defines one, and it is recorded as a retry in the run record. Automation that quietly re-runs until green is manufacturing false confidence.
2. **Track per-case flake rate** across all runs. It is a first-class metric, published alongside pass rate.
3. **A case flaking above the defined threshold is a defect** — against the test, the fixture, or the software, in that order of likelihood. It is triaged, not tolerated.
4. **A flaky case cannot close a requirement.** §20.2 credit requires a repeatable result; a case that passes sixty percent of the time has not verified anything.
5. **Intermittent hardware failures are findings, not noise.** An intermittent that is dismissed at L3 becomes an unreproducible anomaly at L6, which is a far more serious finding (§16.2).

### 19.10 Human-in-the-Loop Exceptions

Automation does not extend to these, by intent:

| Exception | Rationale |
|---|---|
| **First power-on of a new card revision or a new fixture revision** | 5.E3 and 4.E23 exist because a wiring error can destroy hardware. A human with a current probe and an abort switch does the first one. Automate from the second article onward. |
| Physical card insertion, removal, or harness reconfiguration | Mechanical risk, ESD, and the fixture configuration must be re-verified afterward |
| Environmental chamber setup and teardown | Article handling and instrumentation routing |
| Any abort requiring judgment about hardware damage | The cell's thresholds handle the clear cases; ambiguous ones stop and wait |
| **Witnessed safety tests** — inhibit verification in particular | Execution may be automated; the witness requirement is a process requirement and stays |
| Adjudication of formal qualification results | Data collection and limit checking are automated; acceptance is human and recorded |
| Anomaly investigation | By definition unscripted |

Everything not in this table is a candidate for automation, and the default answer is yes.

### 19.11 Run Records and Traceability Feed

Every run emits a record that is complete enough to reconstruct the run and is consumed directly by the RTM — no transcription step.

Required in every run record:

- Run ID, timestamp, purpose, requester, and witness identity where `purpose: formal`
- Complete configuration under test per the `config_under_test` block in §19.7
- Checkout suite version and cell controller version
- Per-case result with measured values, not just pass/fail — a pass at 61.4% CPU and a pass at 69.8% against a 70% budget are different facts and the trend matters
- All captures as data files (§18.9), referenced by hash
- Canary result for the cell at the time of the run
- Retry count per case

**The RTM is fed by run records, not by humans.** A test case ID in the run record maps to the requirement it verifies; the RTM is regenerated from the record set. This removes the transcription defect class from verification evidence and makes the pre-gate audits in §20.1 a query rather than a review.

### 19.12 Build Order

Sequenced by dependency and by return on effort.

| Order | Build | Available before | Why first |
|---|---|---|---|
| 1 | **Sequencer with machine-readable case definitions**, running L0/L1 against simulated interfaces | Any hardware exists | Case definitions built here are reused unchanged at L2–L5. Built later, they never get retrofitted. |
| 2 | **ICD generators** (§18.6) | First card | Everything downstream consumes generated artifacts |
| 3 | **Automated load + the three-way verification** (§19.4) | First card bring-up | Removes the highest-frequency manual step immediately |
| 4 | **C0–C4 smoke ladder + recovery** (§19.5, §19.6) | First card bring-up | The 15-minute gate. Largest single return in the whole section. |
| 5 | **Canary + `CELL_FAULT` handling** (§19.8) | Before anyone relies on a red result | Cheap, and it protects the team's trust in everything above |
| 6 | **Cell API + CI integration** (§19.7) | Regular integration cadence | Turns the cell from a bench into infrastructure |
| 7 | **C5–C7** functional, off-nominal, soak | Step 5 campaign | The bulk of the runtime, but only useful once 1–5 are solid |
| 8 | **Seeded-defect verification** (§19.8) | Before the suite gates a formal gate | Proves the suite; required before C-tier results carry verification credit |
| 9 | Second and subsequent cells | Queue depth becomes the constraint | Scale only after one cell is trustworthy |

**Do not build the second cell before the first one is trustworthy.** Two unreliable cells produce twice the false failures and half the confidence.

---

## 20. Traceability and Verification Credit

### 20.1 Trace Chain

```
Requirement (SRS/IRS)
   ↔ Design element (SDD)
      ↔ Code (CSU/CSC, by file and function)
         ↔ Test case (by ID)
            ↔ Test result (by run instance: article SN, config ID, date)
```

Bidirectional and complete. Two audits are run before every gate:
- **Orphan requirements** — requirements with no test case
- **Orphan tests** — test cases tracing to no requirement (these are not wrong, but they must be   labeled as robustness/exploratory rather than counted as verification credit)

### 20.2 Verification Credit Rules

1. Credit is earned at the requirement's **assigned** level. Lower-level passes are supporting evidence.
2. Credit is bound to a **configuration**. A pass on configuration X does not transfer to configuration Y without an analysis.
3. Credit is bound to an **article class**. A pass on an EDU card is evidence, not closure, for a mission card, unless the delta analysis shows equivalence.
4. Credit from a **simulation or emulator** requires that the model itself be validated, and the validation record is part of the evidence.
5. Credit expires when the impact analysis in §15.1 says it does.

### 20.3 Test Case Identification

```
<TYPE>-<LEVEL>-<AREA>-<NNN>

TYPE:   UT (unit) | IT (integration) | OFN (off-nominal) | QT (qualification) | ST (system)
LEVEL:  L0..L6
AREA:   CSC name, bus name, or subsystem
NNN:    sequential

Examples:
  UT-L0-KALMAN-014
  IT-L3-1553-007
  OFN-L4-ETH-011
  ST-L5-FAILOVER-003

Run instance:
  <TestID>/<ArticleSN>/<ConfigID>/<YYYYMMDD>-<seq>
  OFN-L4-ETH-011/BOX-A-SN004/CFG-2026-0912-B/20260912-01
```

---

## 21. Appendix A — HIRR Checklist (one-page form)

```
HARDWARE INTEGRATION READINESS REVIEW
Configuration ID: ________________   Date: __________   Chair: ________________

CSCI: ____________  Version: ________  Tag: ______________________
Repo: ___________________________________________________________
Commit SHA (full): ______________________________________________
Signature verified: [ ] Y  [ ] N        Reproducible build verified: [ ] Y [ ] N

Card PN: __________ Rev: ______ SN: __________
Firmware: __________ Ver: ______   Bootloader: __________ Ver: ______
Backplane Rev: ______   Harness Dwg/Rev: _______________
Combination present in compatibility matrix: [ ] Y  [ ] N  [ ] New row proposed

SOFTWARE READINESS                                    Y / N / WAIVED   Ref
 4.E1  CQR passed                                     [ ] [ ] [ ]     ______
 4.E2  Repo + full SHA recorded                       [ ] [ ] [ ]     ______
 4.E3  Signed tag verified                            [ ] [ ] [ ]     ______
 4.E4  Version assigned per policy                    [ ] [ ] [ ]     ______
 4.E5  Reproducible build (2 agents, identical)       [ ] [ ] [ ]     ______
 4.E6  Toolchain pinned by digest                     [ ] [ ] [ ]     ______
 4.E7  Artifacts + SHA-256 in registry                [ ] [ ] [ ]     ______
 4.E8  Memory budget within allocation + margin       [ ] [ ] [ ]     ______
 4.E9  SBOM complete, licenses cleared                [ ] [ ] [ ]     ______
 4.E10 ICD revs declared + message DB hash            [ ] [ ] [ ]     ______
 4.E11 Compatibility declared (HW/GW/BL/peers)        [ ] [ ] [ ]     ______
 4.E12 Peer CSCIs available at compatible versions    [ ] [ ] [ ]     ______
 4.E13 Zero Sev 1/2; Sev 3 dispositioned              [ ] [ ] [ ]     ______
 4.E14 Impact analysis + regression scope             [ ] [ ] [ ]     ______
 4.E15 Load procedure written and dry-run             [ ] [ ] [ ]     ______
 4.E16 Known-good fallback image + path verified      [ ] [ ] [ ]     ______
 4.E17 Test hooks inventoried with states             [ ] [ ] [ ]     ______
 4.E18 Safety-related behaviors identified            [ ] [ ] [ ]     ______

HARDWARE READINESS
 4.E19 Card config recorded to SN                     [ ] [ ] [ ]     ______
 4.E20 Firmware in supported range                    [ ] [ ] [ ]     ______
 4.E21 Bootloader in supported range                  [ ] [ ] [ ]     ______
 4.E22 Card HW acceptance passed                      [ ] [ ] [ ]     ______
 4.E23 Safe-to-mate complete and signed               [ ] [ ] [ ]     ______
 4.E24 Harness config recorded and verified           [ ] [ ] [ ]     ______
 4.E25 Power budget + current limit set               [ ] [ ] [ ]     ______
 4.E26 Terminations / pull-ups correct                [ ] [ ] [ ]     ______
 4.E27 ESD controls in place                          [ ] [ ] [ ]     ______

TEST READINESS
 4.E28 Procedure approved with expected results       [ ] [ ] [ ]     ______
 4.E29 Equipment available and in calibration         [ ] [ ] [ ]     ______
 4.E30 Data recording plan                            [ ] [ ] [ ]     ______
 4.E31 Test conductor + QA witness assigned           [ ] [ ] [ ]     ______
 4.E32 Abort / safing criteria defined                [ ] [ ] [ ]     ______
 4.E33 Anomaly reporting path briefed                 [ ] [ ] [ ]     ______

WAIVERS (each requires risk statement and mitigation):
 ________________________________________________________________
 ________________________________________________________________

AUTHORIZATION TO APPLY POWER
 I&T Lead        ______________________  Date ________
 Software Lead   ______________________  Date ________
 Card/HW Lead    ______________________  Date ________
 Systems Eng     ______________________  Date ________
 SQA             ______________________  Date ________
 Safety          ______________________  Date ________
```

---

## 22. Appendix B — Bring-Up Order Rationale

The bus bring-up order in Step 5 (GPIO → I²C → serial → 1553 → Ethernet → PCIe) is deliberate:

1. **GPIO first** — a single net, directly observable on a scope, no protocol. If a discrete does    not work, nothing above it will be diagnosable.
2. **I²C second** — simple, slow, easy to observe, and it typically carries the board's own identity    and thermal/power telemetry. Working I²C gives observability for everything that follows.
3. **Serial third** — point-to-point, no arbitration, easy to observe with a terminal or analyzer.
4. **1553 fourth** — deterministic, well-instrumented, and command/control critical. Enough complexity to need the lower layers working first.
5. **Ethernet fifth** — high rate, stack-heavy, many layers that can each fail.
6. **PCIe last** — highest complexity, deepest hardware coupling, most difficult to observe, and most dependent on correct clocks, power, and reset sequencing already proven by the steps above. It is also the only bus here that is COTS and unverified (§14.6), so bring-up is about establishing communication and exercising the error paths, not about proving the link.

Debugging a PCIe enumeration failure is dramatically easier when the card's own health telemetry over I²C already works.

---

## 23. Open Tailoring Items

These need program decisions before this document is baselined:

1. **Software classification framework.** NPR 7150.2 vs DO-178C vs a company standard. Drives coverage targets, independence requirements, and the depth of the verification argument.
2. **Partitioned vs non-partitioned OS.** If partitioned (ARINC 653-style or a separation kernel), add partition-level isolation testing: time and space partitioning violation tests, health monitor behavior, and partition restart semantics.
3. **Firmware treatment.** Whether FPGA logic is treated as a CSCI, an HWCI, or its own category — this affects the gates it passes through and the coverage evidence it owes.
4. **Redundancy architecture specifics.** Dual vs triple; voting location; cross-strapping topology. §11 fault scenarios depend heavily on this.
5. **Safety and termination system software involvement.** If any software participates in the termination path, its requirements are far more stringent and likely externally imposed.
6. **Radiation environment.** Mission duration and operating environment determine whether SEE testing is required and at what fidelity.
7. **Reuse and heritage.** Whether heritage software carries verification credit forward, and under what analysis.
8. **Automation investment.** The regression tier durations in §15.2 assume the automation described in §19, and the coverage percentages in §19.2 assume it is funded as infrastructure rather than as spare-time effort. Without it, T2 and T3 are not achievable at the stated cadence and the integration rate is capped by technician availability.
9. **Number of EDU vs mission articles.** Drives how much verification can be done on non-mission hardware and what the delta analysis must cover.
10. **Ground software scope.** Whether EGSE and ground command/telemetry software follow this same process or a lighter one.
11. **COTS qualification depth.** PCIe cards and their vendor stack are COTS in a mission-critical data path. How much vendor evidence, screening, or acceptance testing is required, and who accepts the residual risk, is a program decision that §2.7 assumes but does not answer.

---

*End of document.*

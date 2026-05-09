# Zonal Architecture Demo — Requirements Specification

**Document ID:** ZAD-REQ-001
**Version:** 0.1 (Draft)
**Author:** [Your Name]
**Date:** [Fill in]
**Status:** Draft — for internal review

---

## 1. Purpose and Scope

This document specifies the requirements for the Zonal Architecture Demonstrator (hereafter "the system"), a bench-top prototype illustrating the interaction between a Zone Electronic Control Unit (Zone ECU), a central vehicle compute node, an in-zone smart sensor/actuator node, and a Bluetooth Low Energy (BLE) "phone-as-key" device.

The system is intended as an educational and portfolio artifact and is **not** intended for use in or on a production vehicle. Where a requirement deliberately diverges from a comparable production-grade requirement, the divergence is documented in Section 8 (Deviations from Production Practice).

### 1.1 In Scope

- A single Zone ECU implemented on an STM32 Nucleo-H563ZI development board.
- A Smart Sensor/Actuator Node (SSAN) communicating with the Zone ECU over a Classical CAN bus (500 kbit/s).
- A Central Compute Node (CCN) implemented on a host laptop running Linux, exposing a SOME/IP-style service over Ethernet.
- A BLE Phone Key implemented on an ESP32 development board.
- End-to-end use case: authenticated phone presence triggers a simulated door unlock.

### 1.2 Out of Scope

- Production-grade functional safety (ISO 26262) compliance.
- Production-grade cybersecurity (ISO 21434) compliance, secure boot, or hardware security modules.
- Time-Sensitive Networking (TSN), Automotive Ethernet PHYs (100BASE-T1/1000BASE-T1), or PTP synchronization.
- Ultra-Wideband (UWB) ranging.
- Power management, sleep/wake, or partial-network operation.

### 1.3 Document Conventions

- The word **shall** denotes a binding requirement.
- The word **should** denotes a recommendation.
- The word **may** denotes an option.
- Each requirement carries a unique identifier and a verification method tag:
  - **T** — Test
  - **A** — Analysis
  - **I** — Inspection
  - **D** — Demonstration

---

## 2. Definitions and Acronyms

| Term | Definition |
|------|------------|
| Zone ECU | Electronic Control Unit responsible for sensors and actuators within one physical vehicle zone. |
| CCN | Central Compute Node — high-performance node running service-oriented applications. |
| SSAN | Smart Sensor/Actuator Node — simple in-zone node attached to the Zone ECU via CAN. |
| SOME/IP | Scalable service-Oriented MiddlewarE over IP. |
| PEPS | Passive Entry Passive Start. |
| RSSI | Received Signal Strength Indicator. |
| BLE | Bluetooth Low Energy. |
| FDCAN | Flexible Data-rate Controller Area Network (used here in Classical CAN mode). |

---

## 3. System Overview

The system comprises four nodes:

1. **CCN (laptop)** — runs the SOME/IP service `DoorLockService` and a BLE scanner that authenticates the phone key.
2. **Zone ECU (Nucleo-H563ZI)** — connects to the CCN over Ethernet and to the SSAN over CAN. Hosts local user I/O: 16x2 LCD, 4x4 keypad, status LED, audible buzzer/speaker.
3. **SSAN (ESP32 / second MCU)** — drives the door-lock servo and reports lock state on CAN.
4. **Phone Key (ESP32)** — BLE peripheral advertising a custom GATT service.

A complete architecture diagram is maintained in `02_architecture.md`.

---

## 4. Customer Requirements (CR)

These are the high-level behaviors a hypothetical end-user or product owner would care about. They are deliberately written in non-technical language.

| ID | Requirement | Verification |
|----|-------------|--------------|
| **CR-01** | The vehicle shall unlock automatically when the authenticated user's phone key is brought within close proximity (~2 m) of the vehicle. | D |
| **CR-02** | The vehicle shall re-lock automatically when the authenticated phone key has been absent for more than 10 seconds. | D |
| **CR-03** | The user shall be able to unlock the vehicle by entering a 4-digit PIN on the keypad as a backup if no phone key is present. | D |
| **CR-04** | The system shall provide visible and audible feedback to the user upon every lock or unlock event. | D |
| **CR-05** | An unauthorized phone (one without valid credentials) shall not be able to unlock the vehicle. | T |
| **CR-06** | The system shall reject more than 5 consecutive incorrect PIN entries within 60 seconds and enter a lockout state for at least 30 seconds. | T |

---

## 5. System Requirements (SYS-REQ)

Decomposed from the customer requirements. Each system requirement traces back to one or more customer requirements.

### 5.1 Functional

| ID | Requirement | Traces to | Verification |
|----|-------------|-----------|--------------|
| **SYS-REQ-01** | The system shall detect a paired BLE peripheral with RSSI ≥ −75 dBm within 1000 ms of the peripheral entering range. | CR-01 | T |
| **SYS-REQ-02** | The system shall complete a challenge-response authentication with a detected BLE peripheral within 500 ms of detection. | CR-01, CR-05 | T |
| **SYS-REQ-03** | The total latency from successful BLE authentication to door-lock actuator movement shall not exceed 800 ms (95th percentile, 50 trials). | CR-01 | T |
| **SYS-REQ-04** | The system shall transition to the LOCKED state when the authenticated phone key has not been observed for ≥ 10 s. | CR-02 | T |
| **SYS-REQ-05** | The system shall accept PIN input via the 4x4 keypad and unlock when a 4-digit PIN matching the stored value is entered. | CR-03 | D |
| **SYS-REQ-06** | The system shall display the current lock state ("LOCKED" or "UNLOCKED") on the LCD at all times. | CR-04 | D |
| **SYS-REQ-07** | The system shall emit a distinct audible tone (≥ 300 ms) on each lock and unlock event. | CR-04 | D |
| **SYS-REQ-08** | The system shall reject a BLE peripheral whose challenge-response signature does not validate against the stored shared key. | CR-05 | T |
| **SYS-REQ-09** | The system shall enter a 30 s lockout state after 5 consecutive incorrect PIN entries within a 60 s window. | CR-06 | T |

### 5.2 Communication

| ID | Requirement | Traces to | Verification |
|----|-------------|-----------|--------------|
| **SYS-REQ-10** | The CCN shall expose a SOME/IP-style service `DoorLockService` (Service ID `0x1234`) reachable via UDP on port 30509. | CR-01, CR-03 | I |
| **SYS-REQ-11** | The Zone ECU shall communicate with the SSAN over a Classical CAN bus operating at 500 kbit/s with 60 Ω termination at each end. | CR-01 | I |
| **SYS-REQ-12** | The CAN frame `LOCK_CMD` (ID `0x100`) shall carry a 1-byte payload: `0x00` = LOCK, `0x01` = UNLOCK. All other values are reserved. | — | I |
| **SYS-REQ-13** | The SSAN shall broadcast a heartbeat CAN frame `LOCK_STATUS` (ID `0x101`) at 100 ms ± 10 ms intervals containing the current lock state. | — | T |
| **SYS-REQ-14** | The Zone ECU shall log a fault and display "SSAN TIMEOUT" on the LCD if no `LOCK_STATUS` frame is received for ≥ 500 ms. | CR-04 | T |

### 5.3 Non-Functional

| ID | Requirement | Verification |
|----|-------------|--------------|
| **SYS-REQ-15** | The complete system shall be powered from a single 9 V supply (battery or wall adapter) with on-board 5 V regulation. | I |
| **SYS-REQ-16** | The full source code, schematics, requirements, and test results shall be maintained in a public Git repository. | I |
| **SYS-REQ-17** | All source code shall be formatted using `clang-format` (LLVM style) and pass a Cppcheck static analysis run with no warnings of severity `error` or `warning`. | A |

---

## 6. Component Requirements

Component requirements decompose system requirements onto specific nodes. They are the contract between the system architect and the implementer of each node.

### 6.1 Zone ECU (ZC-REQ)

| ID | Requirement | Traces to | Verification |
|----|-------------|-----------|--------------|
| **ZC-REQ-01** | The Zone ECU shall be implemented on an STM32 Nucleo-H563ZI development board. | — | I |
| **ZC-REQ-02** | The Zone ECU shall expose an Ethernet interface configured with a static IPv4 address `192.168.10.20/24`. | SYS-REQ-10 | T |
| **ZC-REQ-03** | The Zone ECU shall act as a SOME/IP-style client and subscribe to the `DoorLockService` event group on startup. | SYS-REQ-10 | T |
| **ZC-REQ-04** | The Zone ECU shall transmit the `LOCK_CMD` CAN frame within 50 ms (95th percentile) of receiving an `UnlockRequest` from the CCN. | SYS-REQ-03 | T |
| **ZC-REQ-05** | The Zone ECU shall scan the 4x4 keypad at ≥ 50 Hz and debounce each key for ≥ 20 ms. | SYS-REQ-05 | T |
| **ZC-REQ-06** | The Zone ECU firmware shall use FreeRTOS with separate tasks for: (a) network, (b) CAN, (c) user I/O. | — | I |

### 6.2 Smart Sensor/Actuator Node (SS-REQ)

| ID | Requirement | Traces to | Verification |
|----|-------------|-----------|--------------|
| **SS-REQ-01** | The SSAN shall actuate the door-lock servo to the UNLOCKED position (0°) within 100 ms of receiving a `LOCK_CMD = 0x01` frame. | SYS-REQ-03 | T |
| **SS-REQ-02** | The SSAN shall actuate the door-lock servo to the LOCKED position (90°) within 100 ms of receiving a `LOCK_CMD = 0x00` frame. | SYS-REQ-03 | T |
| **SS-REQ-03** | The SSAN shall emit `LOCK_STATUS` heartbeat frames as defined in SYS-REQ-13. | SYS-REQ-13 | T |

### 6.3 Central Compute Node (CC-REQ)

| ID | Requirement | Traces to | Verification |
|----|-------------|-----------|--------------|
| **CC-REQ-01** | The CCN shall scan for BLE advertisements at least once every 200 ms. | SYS-REQ-01 | T |
| **CC-REQ-02** | The CCN shall maintain a whitelist of authorized BLE device UUIDs in a configuration file (`config/keys.toml`). | SYS-REQ-08 | I |
| **CC-REQ-03** | The CCN shall perform challenge-response authentication using HMAC-SHA256 with a 256-bit pre-shared key. | SYS-REQ-02, SYS-REQ-08 | I |
| **CC-REQ-04** | The CCN shall publish a SOME/IP `UnlockRequest` event upon successful authentication, and a `LockRequest` event upon SYS-REQ-04 timeout. | SYS-REQ-04, SYS-REQ-10 | T |

### 6.4 Phone Key (PK-REQ)

| ID | Requirement | Traces to | Verification |
|----|-------------|-----------|--------------|
| **PK-REQ-01** | The Phone Key shall advertise a custom GATT service (UUID `[to-be-assigned]`) with a writable challenge characteristic and a notify response characteristic. | SYS-REQ-01, SYS-REQ-02 | I |
| **PK-REQ-02** | The Phone Key shall respond to a 16-byte challenge with the HMAC-SHA256 of the challenge using the pre-shared key, within 200 ms. | SYS-REQ-02 | T |
| **PK-REQ-03** | The Phone Key shall transmit BLE advertisements at 0 dBm output power. | SYS-REQ-01 | I |

---

## 7. Traceability Matrix

A standalone traceability matrix is maintained in `04_traceability.csv`. The following table is a high-level summary; CSV is the source of truth.

| Customer Req | System Reqs | Component Reqs | Test Cases |
|--------------|-------------|----------------|------------|
| CR-01 | SYS-REQ-01, 02, 03, 10, 11 | ZC-REQ-02..04, SS-REQ-01, CC-REQ-01..04, PK-REQ-01..03 | TC-001..005 |
| CR-02 | SYS-REQ-04 | CC-REQ-04, SS-REQ-02 | TC-006 |
| CR-03 | SYS-REQ-05, 10 | ZC-REQ-05 | TC-007 |
| CR-04 | SYS-REQ-06, 07, 14 | — | TC-008, 009 |
| CR-05 | SYS-REQ-08 | CC-REQ-02, 03 | TC-010 |
| CR-06 | SYS-REQ-09 | — | TC-011 |

---

## 8. Deviations from Production Practice

Documenting these is part of the engineering exercise. Each deviation lists what a production-grade implementation would do differently and why this prototype does not.

| ID | Deviation | Production Practice | Rationale for Deviation |
|----|-----------|---------------------|-------------------------|
| **DEV-01** | Pre-shared HMAC key, baked into firmware. | Per-vehicle key provisioning at end-of-line; key stored in HSM/SE; rotation supported. | Out of scope for an 8-week prototype; key management is a project of its own. |
| **DEV-02** | RSSI used as a coarse proximity estimate. | UWB ranging (CCC Digital Key 3.0) for centimeter-level distance, immune to relay attacks. | UWB hardware (e.g., Qorvo DWM3000) adds cost and complexity beyond scope. |
| **DEV-03** | Standard 100BASE-TX Ethernet between CCN and Zone ECU. | 100BASE-T1 / 1000BASE-T1 single-pair Automotive Ethernet. | Automotive PHYs require expensive media converters and offer no functional difference for a bench demo. |
| **DEV-04** | No secure boot, no signed firmware. | ECUs use chain-of-trust boot rooted in immutable hardware. | The H5 supports this (RoT, OBK) but configuring it correctly is a multi-week task on its own. |
| **DEV-05** | SOME/IP implementation is partial — service discovery and event subscription only, no full COVESA vsomeip compliance on the MCU side. | Full vsomeip or comparable stack on every node. | Porting vsomeip to a Cortex-M is impractical; a documented subset is the honest engineering choice. |

---

## 9. Open Items

Items that need to be resolved before this document moves from Draft to Released.

- [ ] Assign concrete UUIDs for the BLE GATT service and characteristics.
- [ ] Confirm 9 V battery life under continuous operation; consider USB-C power instead.
- [ ] Decide whether the SSAN is a second STM32 or an ESP32 (impacts SS-REQ-01..03 timing budgets).
- [ ] Threat model document (`03_threat_model.md`) to be drafted in Week 6.

---

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | [Today] | [Your Name] | Initial draft. |

# Zonal Architecture Demo — System Architecture

**Document ID:** ZAD-ARC-001
**Version:** 0.1 (Draft)
**Traces to:** ZAD-REQ-001
**Author:** [Your Name]
**Date:** [Fill in]

---

## 1. Purpose

This document describes the architecture of the Zonal Architecture Demonstrator. It is the bridge between the requirements (`01_requirements.md`) and the implementation. A reader who has skimmed the requirements should be able to read this document and understand *how* the system is structured to satisfy them, without yet reading any source code.

The architecture is presented in three views, in order of increasing detail:

1. **Logical view** — what the nodes are and what they do.
2. **Communication view** — how the nodes talk to each other, at every protocol layer.
3. **Physical view** — what the actual hardware looks like, what the wiring is, and where each piece of code runs.

A fourth section covers the **runtime behavior** for the principal use case (phone-as-key unlock), as a sequence diagram in prose.

---

## 2. Logical View

The system has four logical nodes. Each node has a single primary responsibility, deliberately mirroring the separation of concerns in a production zonal E/E architecture.

### 2.1 Central Compute Node (CCN)

The CCN runs on a host laptop under Linux. It is the only node that knows about *users* and *credentials*. Its responsibilities are:

- BLE scanning and challenge-response authentication of phone keys.
- Maintaining the high-level lock state machine (LOCKED, UNLOCKED, LOCKED_OUT).
- Hosting the SOME/IP-style service `DoorLockService` that downstream nodes consume.
- Logging.

The CCN does **not** know how the lock is physically actuated, what kind of bus the actuator sits on, or what the actuator's electrical interface looks like. That separation is the entire point of a service-oriented architecture: the "what" lives upstream, the "how" lives downstream.

### 2.2 Zone Electronic Control Unit (Zone ECU)

The Zone ECU runs on an STM32 Nucleo-H563ZI. In the production analogy, it is one of (typically) four zone controllers in a vehicle, responsible for the sensors and actuators in its physical zone. In this prototype it is the only zone — the "front-left" zone of an imaginary vehicle. Its responsibilities are:

- Acting as a SOME/IP client to the CCN's `DoorLockService`.
- Translating service-level events into in-zone bus traffic (CAN frames).
- Owning the in-zone user I/O: the 16x2 LCD (status), the 4x4 keypad (PIN entry), the status LED, and the audible buzzer.
- Monitoring the health of in-zone nodes via heartbeat.

The Zone ECU is a *gateway* in the classical sense: it speaks Ethernet on one side and CAN on the other, and translates between them. This is exactly the role of a real Bosch zone controller.

### 2.3 Smart Sensor/Actuator Node (SSAN)

The SSAN is a small microcontroller (a second STM32 or an ESP32 in CAN mode — see §6.4 for the open decision) that lives at the end of the CAN bus and drives the door-lock servo. Its responsibilities are:

- Receiving `LOCK_CMD` frames and actuating the servo.
- Emitting `LOCK_STATUS` heartbeat frames at 10 Hz so the Zone ECU knows it's alive.

The SSAN deliberately has *no knowledge* of authentication, users, or service contracts. It is a dumb actuator that does what it's told. This is intentional — in production, smart sensors and actuators are built by suppliers who don't know the vehicle's high-level architecture, and the protocol on the wire is the entire interface.

### 2.4 Phone Key

An ESP32 acting as a BLE peripheral. It advertises a custom GATT service and responds to challenges with HMAC-SHA256 signatures. Its only job is to prove possession of the shared key.

In production this would be a smartphone running the OEM's app, using the Car Connectivity Consortium's Digital Key spec. The ESP32 is a stand-in that lets us implement the protocol without writing an iOS or Android app.

---

## 3. Communication View

This is where the protocol stack matters. Every link in the system is shown below with its full layering.

### 3.1 CCN ↔ Zone ECU (Ethernet)

| Layer | Protocol | Notes |
|-------|----------|-------|
| Application | `DoorLockService` (SOME/IP-style events) | Two events: `UnlockRequest`, `LockRequest`. Service ID `0x1234`. |
| Middleware | SOME/IP subset (service discovery + event group subscription) | Full vsomeip on CCN; minimal hand-rolled subset on Zone ECU (see DEV-05). |
| Transport | UDP, port 30509 | UDP rather than TCP; SOME/IP events are stateless and a dropped packet is recovered by the next event. |
| Network | IPv4, static `192.168.10.0/24` | CCN at `.10`, Zone ECU at `.20`. |
| Link / Physical | 100BASE-TX | Standard Ethernet rather than 100BASE-T1 (DEV-03). |

### 3.2 Zone ECU ↔ SSAN (CAN)

| Layer | Protocol | Notes |
|-------|----------|-------|
| Application | Two messages: `LOCK_CMD` (ID `0x100`, 1 byte) and `LOCK_STATUS` (ID `0x101`, 1 byte). | See SYS-REQ-12, -13. |
| Data Link | Classical CAN 2.0A, 500 kbit/s | FDCAN peripheral on the H5, configured for Classical mode. |
| Physical | ISO 11898-2 high-speed CAN, 60 Ω termination at each end | SN65HVD230 transceivers, 3.3V-compatible with the H5's I/O. |

### 3.3 CCN ↔ Phone Key (BLE)

| Layer | Protocol | Notes |
|-------|----------|-------|
| Application | Challenge-response over a custom GATT service | One writable characteristic (challenge), one notify characteristic (response). |
| Authentication | HMAC-SHA256, 256-bit pre-shared key | DEV-01: real systems would use ECDH key agreement and per-session keys. |
| Bluetooth | BLE 5.0, peripheral role on Phone Key, central role on CCN | Connection interval not tuned for power; this is a wall-powered prototype. |

---

## 4. Physical View

### 4.1 Bill of Materials (Build Configuration)

| Node | Hardware | Where it runs |
|------|----------|---------------|
| CCN | Host laptop (Linux or WSL2) | `ccn/` directory of repo |
| Zone ECU | STM32 Nucleo-H563ZI | `zone_ecu/` (STM32CubeIDE project) |
| SSAN | [TBD: see §6.4] | `ssan/` |
| Phone Key | ESP32-WROOM-32 DevKitC | `phone_key/` (ESP-IDF project) |
| In-zone I/O | 16x2 character LCD (Newhaven NHD-0216BZ-RN-YBW), 4x4 membrane keypad, LED, 7805 regulator, 9V battery, tactile switch, 0.5W speaker, 10K trim pot | Wired to Zone ECU |
| CAN PHY | SN65HVD230 transceiver breakouts (×2) | One on Zone ECU, one on SSAN |
| Actuator | SG90 micro servo | Driven by SSAN PWM output |
| Bus | 60 Ω termination at each end of the CAN bus | Two 120 Ω resistors in parallel at each end, OR one 120 Ω resistor at each physical end |

### 4.2 Power Distribution

- 9 V alkaline battery feeds a 7805 linear regulator producing 5 V.
- 5 V powers the LCD, the SG90 servo (briefly — peak current is ~600 mA so a bulk capacitor on the rail is required), and the tactile/keypad pull-ups.
- Each MCU is powered separately via USB during development; the 9 V rail is reserved for the simulated "vehicle 12 V" sub-system. In a future revision, the Zone ECU should be powered from the 5 V rail to better mirror a real ECU.
- A 10 µF tantalum capacitor sits at the regulator output; another 0.1 µF ceramic at each IC's Vcc pin.

### 4.3 Wiring Summary

A complete schematic lives in `hardware/schematic.pdf`. The minimum-viable wiring is:

- **Zone ECU FDCAN1**: PD0 (RX), PD1 (TX) → SN65HVD230 RXD/TXD pins. Transceiver CAN_H/CAN_L to the bus.
- **Zone ECU LCD**: I2C backpack on PB8 (SCL), PB9 (SDA), 5V, GND. (If using parallel: 6 GPIOs plus power.)
- **Zone ECU keypad**: 8 GPIOs (4 rows + 4 columns) on the Arduino Uno header pins.
- **Zone ECU LED**: PB0 (built-in green LED) plus an external diffused red LED on PB7 with a 330 Ω series resistor.
- **Zone ECU speaker**: PA5 → MOSFET gate → speaker → 5V (or directly via series resistor for low-volume operation).
- **Zone ECU Ethernet**: built-in RJ45 jack → patch cable → laptop or switch.
- **SSAN servo**: one GPIO (PWM-capable) → SG90 signal pin. Servo Vcc on 5V rail with bulk cap.

---

## 5. Runtime Behavior — Principal Use Case

This narrative covers the happy path: an authorized phone key approaches the vehicle and the door unlocks.

1. **Steady state.** The Zone ECU is booted, has subscribed to `DoorLockService`, and is displaying "LOCKED" on the LCD. The SSAN is emitting `LOCK_STATUS = 0x00` (LOCKED) every 100 ms. The CCN is scanning for BLE advertisements every 200 ms.
2. **Detection.** The user walks within ~2 m of the laptop with the powered Phone Key in their pocket. The CCN sees an advertisement matching a whitelisted UUID with RSSI ≥ −75 dBm. (T0)
3. **Authentication.** The CCN connects to the Phone Key, writes a 16-byte random challenge to the challenge characteristic, and waits for the notify response. The Phone Key computes HMAC-SHA256(key, challenge) and notifies the response. The CCN verifies the signature. (T0 + ~300 ms)
4. **Service event.** The CCN updates its internal state to UNLOCKED and publishes a SOME/IP `UnlockRequest` event. The Zone ECU, subscribed to this event, receives it within one network round-trip. (T0 + ~310 ms)
5. **Bus translation.** The Zone ECU's network task wakes on the received event and posts a message to the CAN task. The CAN task transmits a `LOCK_CMD = 0x01` frame on the bus. (T0 + ~360 ms)
6. **Actuation.** The SSAN's CAN receive interrupt fires, the main loop reads the new command, and the PWM duty cycle is updated to drive the servo to 0° (UNLOCKED). The servo physically rotates over ~80 ms. (T0 + ~440 ms)
7. **Feedback.** In parallel with step 6, the Zone ECU updates the LCD to "UNLOCKED", lights the red LED, and emits a 300 ms unlock tone on the speaker.
8. **State broadcast.** The SSAN's next `LOCK_STATUS` heartbeat (within 100 ms) carries the new state `0x01`. The Zone ECU forwards a SOME/IP `LockStateChanged` event back to the CCN for logging.

End-to-end target: T0 → servo movement complete in ≤ 800 ms (SYS-REQ-03).

---

## 6. Architectural Decisions and Open Items

Decisions worth recording — both for the reader and for your own future self when you forget why you did something three months later.

### 6.1 Why a single Zone ECU?

In a real vehicle there would be 3-5 zone controllers. A single zone is enough to demonstrate the architectural pattern: gateway role, in-zone bus, smart actuator at the end of the bus. Adding a second zone would be more wiring without much new learning.

### 6.2 Why UDP, not TCP, for SOME/IP?

SOME/IP events are designed around UDP because they are stateless notifications. A dropped event is recovered by the next event. TCP's head-of-line blocking would actively hurt the latency budget in SYS-REQ-03.

### 6.3 Why HMAC-SHA256 for the BLE challenge?

It is the simplest construction that resists replay (because the challenge is fresh) and forgery (because the attacker doesn't have the key). It does **not** resist *relay* attacks — an attacker between the phone and the car can replay the BLE link end-to-end. UWB ranging is the production answer (DEV-02).

### 6.4 OPEN — SSAN hardware

The SSAN has not been fixed. The trade is:

- **Second STM32** (e.g. an F4 Nucleo): consistent toolchain with the Zone ECU, native CAN. Higher BOM cost.
- **ESP32 + MCP2515**: cheaper, but two transceivers in two ecosystems. The ESP32's Arduino-style ecosystem is faster to bring up.

Decision is owed by end of Week 2.

### 6.5 OPEN — Where does the Phone Key whitelist live?

Currently planned as a TOML file on the CCN (CC-REQ-02). In a real vehicle this would be in a secure element with a key-rotation protocol. For the prototype, file-based is fine but should be flagged in the threat model.

---

## 7. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | [Today] | [Your Name] | Initial draft. |

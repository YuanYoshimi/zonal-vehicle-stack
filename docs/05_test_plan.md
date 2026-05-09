# Zonal Architecture Demo — Test Plan

**Document ID:** ZAD-TP-001
**Version:** 0.1 (Draft)
**Traces to:** ZAD-REQ-001

---

## 1. Purpose

This document specifies the test cases used to verify the requirements in ZAD-REQ-001. Each test case lists the requirement(s) it covers, the equipment needed, the procedure, the pass criteria, and a results section to be filled in during execution.

## 2. Test Environment

- **Bench setup**: Zone ECU, SSAN, CCN (laptop), Phone Key (ESP32) all powered and connected per the architecture diagram.
- **Measurement tools**: USB logic analyzer (Saleae clone, Sigrok/PulseView), CAN sniffer (`candump` on CCN via CANable), Wireshark on CCN Ethernet interface, oscilloscope (optional for RF/signal-integrity checks).
- **Software**: Test scripts in `tests/` directory of the project repo. Latency measurements use the GPIO-toggle-and-logic-analyzer technique unless otherwise stated.

## 3. Pass / Fail Convention

A test case **passes** only if every numbered step's expected result is observed. Any deviation requires an entry in the Results table with the observed value and a follow-up issue filed in the project tracker.

---

## 4. Test Cases

### TC-001 — BLE proximity detection latency

**Verifies:** SYS-REQ-01, PK-REQ-03
**Setup:** Phone Key powered off and placed exactly 2.0 m ± 0.1 m from the CCN antenna. Stopwatch script `tests/tc001_proximity.py` running on CCN.

**Procedure:**
1. Power on the Phone Key. Script captures the timestamp of power-on (T0) via a GPIO line tied to the ESP32 boot pin.
2. Script logs the timestamp (T1) of the first BLE advertisement seen with RSSI ≥ −75 dBm matching the whitelisted UUID.
3. Repeat 50 times with at least 5 s between trials.

**Pass criteria:** (T1 − T0) ≤ 1000 ms in ≥ 95% of trials. Mean and 95th percentile recorded.

**Results:**

| Trial set | Mean (ms) | 95th pct (ms) | Pass count | Notes |
|-----------|-----------|---------------|------------|-------|
| [date]    |           |               |            |       |

---

### TC-002 — Challenge-response authentication latency

**Verifies:** SYS-REQ-02, CC-REQ-03, PK-REQ-02
**Setup:** Phone Key paired and within range. Wireshark capture filter on the BLE adapter interface.

**Procedure:**
1. CCN script issues a 16-byte random challenge over the GATT challenge characteristic (T0 = write completion).
2. CCN logs the timestamp of the notify response (T1).
3. CCN verifies the response is HMAC-SHA256(key, challenge).
4. Repeat 50 times.

**Pass criteria:** (T1 − T0) ≤ 500 ms in ≥ 95% of trials AND every response signature validates.

**Results:** _to be filled in_

---

### TC-003 — End-to-end unlock latency

**Verifies:** SYS-REQ-03, ZC-REQ-04, SS-REQ-01
**Setup:** Logic analyzer on (a) CCN GPIO marking SOME/IP `UnlockRequest` send, (b) Zone ECU GPIO marking CAN `LOCK_CMD` transmission, (c) SSAN servo PWM line.

**Procedure:**
1. Trigger an authenticated proximity event.
2. Logic analyzer captures (a), (b), and PWM duty change on (c).
3. Compute T_total = T(c) − T(a). Compute Zone ECU latency = T(b) − T(a). Compute SSAN latency = T(c) − T(b).
4. Repeat 50 times.

**Pass criteria:** T_total ≤ 800 ms (95th percentile) AND Zone ECU latency ≤ 50 ms (95th percentile) AND SSAN latency ≤ 100 ms (95th percentile).

**Results:** _to be filled in_

---

### TC-004 — SOME/IP service discovery and event delivery

**Verifies:** SYS-REQ-10, ZC-REQ-02, ZC-REQ-03, CC-REQ-04
**Setup:** Wireshark capturing on CCN Ethernet interface with SOME/IP dissector enabled.

**Procedure:**
1. Power-cycle the Zone ECU. Capture all SOME/IP traffic for 30 s.
2. Verify the service-discovery `OfferService` from CCN is observed.
3. Verify the Zone ECU sends `SubscribeEventgroup` for `DoorLockService`.
4. Trigger an unlock and verify the `UnlockRequest` event is received by the Zone ECU.

**Pass criteria:** All four observations present; no malformed SOME/IP frames.

**Results:** _to be filled in_

---

### TC-005 — CAN bus electrical and protocol integrity

**Verifies:** SYS-REQ-11
**Setup:** Oscilloscope on CAN_H and CAN_L. CAN sniffer on the bus.

**Procedure:**
1. Measure differential voltage during dominant and recessive bits. Verify dominant ≈ 2 V differential, recessive ≈ 0 V.
2. Measure bit time at 500 kbit/s; expect 2 µs ± 0.5%.
3. Send 1000 frames; verify zero errors reported by the CAN controller.

**Pass criteria:** All electrical measurements within tolerance; zero protocol errors.

**Results:** _to be filled in_

---

### TC-006 — Auto-lock after key absence

**Verifies:** SYS-REQ-04, CC-REQ-04, SS-REQ-02
**Setup:** Phone Key in range, system unlocked.

**Procedure:**
1. Power off the Phone Key. Mark T0 as the time of last received advertisement.
2. Observe LCD and servo. Mark T1 as the time the servo reaches LOCKED position.
3. Repeat 10 times.

**Pass criteria:** (T1 − T0) is between 10 s and 11 s in every trial.

**Results:** _to be filled in_

---

### TC-007 — PIN-based unlock

**Verifies:** SYS-REQ-05, ZC-REQ-05
**Setup:** Phone Key absent. Stored PIN `1234`.

**Procedure:**
1. Enter `1234` on the keypad. Verify unlock and audible feedback.
2. Enter `0000` on the keypad. Verify no unlock and an error tone.
3. Repeat 10 trials of each.

**Pass criteria:** All correct PINs unlock; all incorrect PINs do not unlock; appropriate feedback for each.

**Results:** _to be filled in_

---

### TC-008 — User feedback (visual + audible)

**Verifies:** SYS-REQ-06, SYS-REQ-07
**Setup:** None special.

**Procedure:**
1. Trigger lock and unlock events via PIN entry (10 of each).
2. Observe LCD shows correct state at all times.
3. Confirm an audible tone of perceptible duration accompanies each transition.

**Pass criteria:** LCD state matches actuator state in 100% of observations; audible tone present for every transition.

**Results:** _to be filled in_

---

### TC-009 — SSAN heartbeat timeout fault

**Verifies:** SYS-REQ-14
**Setup:** System operating normally.

**Procedure:**
1. Disconnect the SSAN's CAN_H wire. Mark T0.
2. Observe LCD. Mark T1 when "SSAN TIMEOUT" is displayed.

**Pass criteria:** (T1 − T0) ≤ 600 ms; fault state logged over UART.

**Results:** _to be filled in_

---

### TC-010 — Unauthorized phone rejection

**Verifies:** SYS-REQ-08, CC-REQ-02, CC-REQ-03
**Setup:** A second ESP32 flashed with the same GATT service but a different shared key, advertising at 0 dBm in range of CCN.

**Procedure:**
1. Power on the unauthorized device.
2. Observe CCN logs and lock state for 30 s.
3. Repeat with a third device whose UUID is not on the whitelist.

**Pass criteria:** Lock state never transitions to UNLOCKED. CCN logs an authentication-failure event for every challenge attempt.

**Results:** _to be filled in_

---

### TC-011 — PIN brute-force lockout

**Verifies:** SYS-REQ-09
**Setup:** Phone Key absent.

**Procedure:**
1. Enter 5 incorrect PINs in succession (each within ≤ 60 s of the first).
2. Mark T0 = time of 5th incorrect entry.
3. Attempt entry of the correct PIN repeatedly. Mark T1 = time of first successful unlock.

**Pass criteria:** (T1 − T0) ≥ 30 s. LCD shows "LOCKED OUT" during the lockout window.

**Results:** _to be filled in_

---

## 5. Results Summary

To be completed after a full test pass.

| Test ID | Date Run | Pass / Fail | Notes |
|---------|----------|-------------|-------|
| TC-001  |          |             |       |
| TC-002  |          |             |       |
| TC-003  |          |             |       |
| TC-004  |          |             |       |
| TC-005  |          |             |       |
| TC-006  |          |             |       |
| TC-007  |          |             |       |
| TC-008  |          |             |       |
| TC-009  |          |             |       |
| TC-010  |          |             |       |
| TC-011  |          |             |       |

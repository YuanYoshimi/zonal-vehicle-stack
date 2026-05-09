# Zonal Architecture Demonstrator

> A bench-top prototype of a zonal vehicle E/E architecture: a phone-as-key BLE device, a Linux central compute node running a SOME/IP-style service, an STM32-based zone controller, and a CAN-attached smart actuator. Built as a personal project ahead of a 2026 Bosch Cross-Domain Computing internship.

![status: in development](https://img.shields.io/badge/status-in%20development-yellow)
![license](https://img.shields.io/badge/license-MIT-blue)

---

## What this is

Modern vehicle E/E architectures are moving from ~100 distributed ECUs toward a small number of **zone controllers** (responsible for the sensors and actuators in one physical region of the car) coordinated by one or two **central compute nodes** running service-oriented software stacks. Bosch, Aptiv, ZF, and the major OEMs are all building toward this pattern.

This project is a small, honest implementation of that architecture on hobbyist hardware. It exercises:

- **Service-oriented communication** — a SOME/IP-style `DoorLockService` between the central compute node and the zone controller, over standard Ethernet.
- **In-zone bus traffic** — Classical CAN at 500 kbit/s between the zone controller and a smart actuator, with heartbeat monitoring and fault detection.
- **RF authentication** — a BLE challenge-response between the central compute node and a "phone key" (an ESP32 standing in for a smartphone), modeled loosely on production passive-entry systems.
- **Systems engineering rigor** — full requirements specification, traceability matrix, test plan with measured results, and a documented threat model.

## The pitch in one diagram

```
┌──────────────────────────┐                    ┌──────────────────────────┐
│   Central Compute Node   │   BLE (HMAC chal)  │       Phone Key          │
│   (Laptop, Linux)        │ ◄──────────────────│       (ESP32)            │
│   • vsomeip service      │                    │   GATT peripheral        │
│   • BLE scanner + auth   │                    └──────────────────────────┘
└────────────┬─────────────┘
             │ Ethernet (SOME/IP / UDP)
             ▼
┌──────────────────────────┐                    ┌──────────────────────────┐
│       Zone ECU           │   CAN @ 500kbit/s  │  Smart Sensor / Actuator │
│   (STM32 Nucleo-H563ZI)  │ ◄─────────────────►│   (TBD MCU + servo)      │
│   • SOME/IP client       │                    │   • Drives door lock     │
│   • Local I/O (LCD,      │                    │   • 10 Hz heartbeat      │
│     keypad, buzzer)      │                    └──────────────────────────┘
│   • CAN gateway          │
└──────────────────────────┘
```

End-to-end behavior: the phone key approaches the car, the central compute node detects the BLE advertisement, runs a challenge-response, publishes a SOME/IP `UnlockRequest` event, the zone controller translates that into a CAN `LOCK_CMD` frame, and the smart actuator rotates the servo. Target end-to-end latency: ≤ 800 ms.

## Why this project

I'm interning on Bosch's Vehicle Computer Engineering team in summer 2026, supporting RF development for zone-ECU platforms. This project covers the full surface area of the role — RF validation, zone-ECU concepts, automotive Ethernet, service-oriented architecture, and structured systems engineering — in a form I can actually point to and discuss in technical depth.

It is **not** intended to be a production design. Where it deliberately diverges from production practice (no UWB ranging, no secure boot, no real automotive PHYs, pre-shared symmetric keys), the deviations are documented and defended in [`docs/01_requirements.md`](docs/01_requirements.md), §8.

## Documentation

The engineering documentation is the primary deliverable; the hardware demo is its proof.

| Document | Purpose |
|----------|---------|
| [`docs/01_requirements.md`](docs/01_requirements.md) | Customer, system, and component requirements with verification methods. |
| [`docs/02_architecture.md`](docs/02_architecture.md) | Logical, communication, and physical architecture views. |
| [`docs/03_threat_model.md`](docs/03_threat_model.md) | STRIDE-style threat model for the BLE and Ethernet links. *(Week 6)* |
| [`docs/04_traceability.csv`](docs/04_traceability.csv) | Requirement → component → test-case traceability matrix. |
| [`docs/05_test_plan.md`](docs/05_test_plan.md) | 11 test cases with measurement procedures and results tables. |

## Repository layout

```
zonal-demo/
├── README.md                  ← you are here
├── docs/                      ← engineering documentation
├── ccn/                       ← central compute node (Python + C++ + vsomeip)
│   ├── ble_scanner.py         ← BLE scan + challenge-response
│   ├── lockservice/           ← SOME/IP service implementation
│   └── config/keys.toml       ← whitelisted phone keys
├── zone_ecu/                  ← STM32CubeIDE project for the H5
│   ├── Core/                  ← FreeRTOS tasks: net, can, io
│   └── Drivers/
├── ssan/                      ← smart sensor/actuator firmware
├── phone_key/                 ← ESP-IDF project for the phone key
├── hardware/                  ← schematic, BOM, photos
└── tests/                     ← test scripts referenced in the test plan
```

## Build status

This is a work-in-progress. Tracked against an 8-week build schedule:

- [x] Week 1 — Toolchain setup, requirements + architecture drafted, BOM finalized
- [ ] Week 2 — Local I/O on the Zone ECU (LCD, keypad, LED, speaker)
- [ ] Week 3 — CAN link between Zone ECU and SSAN
- [ ] Week 4 — Ethernet + lwIP on the Zone ECU
- [ ] Week 5 — SOME/IP service (CCN side full vsomeip; Zone ECU side hand-rolled subset)
- [ ] Week 6 — BLE phone key + threat model document
- [ ] Week 7 — End-to-end integration + latency measurements
- [ ] Week 8 — Documentation polish, video demo, retrospective

Test results are checked in to `docs/05_test_plan.md` as they become available.

## Hardware

| Item | Part | Source |
|------|------|--------|
| Zone ECU | STM32 Nucleo-H563ZI | Mouser 511-NUCLEO-H563ZI |
| BLE phone key | ESP32-WROOM-32 DevKitC | Espressif / Amazon |
| CAN transceivers (×2) | SN65HVD230 breakout | Amazon |
| USB-CAN adapter | CANable 2.0 | canable.io |
| LCD | Newhaven NHD-0216BZ-RN-YBW (16x2) | Mouser 763-0216BZ-RN-YBW |
| Keypad | Parallax 4x4 membrane | Mouser 619-27899 |
| Servo | SG90 micro servo | Amazon |
| Misc. | 7805 regulator, tantalum + ceramic caps, 9V battery, LEDs, resistors, breadboard, jumpers | Mouser |

Total cost: ~$170. Full BOM in [`hardware/bom.md`](hardware/bom.md).

## How to run it

*(To be filled in as nodes come online — quick-start instructions per node, how to launch the CCN service, how to flash each MCU, how to run the test scripts.)*

## License

MIT. See [`LICENSE`](LICENSE).

## Acknowledgments

Inspired by Bosch's published work on cross-domain computing and zone ECU architectures, COVESA's vsomeip project, and the Car Connectivity Consortium's Digital Key specification — none of which I am affiliated with. All trademarks belong to their respective owners.

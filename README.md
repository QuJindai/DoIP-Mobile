# DoIP-Mobile

Open-source Android DoIP diagnostic client for direct **USB-C Ethernet -> vehicle DoIP** communication.

## V1 scope

- Android / Kotlin / Jetpack Compose
- Ethernet network discovery and explicit socket binding
- ISO 13400 DoIP Vehicle Identification
- TCP/13400 + Routing Activation
- UDS over DoIP
- First end-to-end target: `22 F1 90` VIN read
- Raw TX/RX diagnostic console and local logs

No CAN, CAN FD, ISO-TP, ELM327, Bluetooth VCI, Python runtime or proprietary VCI is required.

The initial milestone is intentionally read-only. State-changing services such as DTC clear, ECU reset, SecurityAccess, RoutineControl and flashing are deferred until the transport stack is proven on simulator and real vehicle.

Design: [`docs/superpowers/specs/2026-09-16-doip-mobile-design.md`](docs/superpowers/specs/2026-09-16-doip-mobile-design.md)

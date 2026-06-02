# Umka Kiosk Standard

An open protocol for museum multimedia kiosk systems.

## What It Defines

- Operating modes (Loop, Browse, Custom) and their configurations
- Real-time control protocol (MQTT)
- Content synchronization (REST API)
- Local-first content architecture
- Guide tablet integration
- IoT device integration (capability-based MQTT protocol — status + commands)
- Event→action bindings (kiosk state drives kiosk/IoT commands)
- Fleet orchestration (operator-initiated power scenarios)
- Control plane service (power management, watchdog, system health)

## Version

Current: **v1.26.5.1** (May 2026)

Versioning scheme: `1.{year}.{patch}` — e.g. `1.26.0` for the 2026 release.

## Reference Implementation

[Umka Player](https://github.com/Maugry/Player) is the MIT-licensed reference implementation of this standard.

## License

This specification is an **open standard**. Any implementation following this specification is considered Umka-compatible.

Developed by [Multimedia Solutions Lab Ltd.](https://maugry.ru)

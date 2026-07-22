# Umka Kiosk Standard

An open protocol for museum multimedia kiosk systems.

## What It Defines

- Operating modes (Loop, Browse, Custom) and their configurations
- Real-time control protocol (MQTT)
- Command correlation and acknowledgement (every command traceable end to end)
- Content synchronization (REST API)
- Local-first content architecture
- Guide tablet integration
- IoT device integration (open, capability-based MQTT protocol — status + commands)
- Kiosk hardware self-description (touch, screens, app variant)
- Software update dispatch and progress reporting
- Log retrieval
- Event→action bindings (kiosk state drives kiosk/IoT commands)
- Named-action vocabulary for coordinated multi-entity behaviour
- Fleet orchestration (operator-initiated power scenarios)
- Control plane service (power management, watchdog, system health)

## Version

Current: **v2.26.0** (July 2026)

Versioning scheme: `{major}.{year}.{patch}` — `major` is the product generation, `year` the year that generation was declared. v2.26.0 opens the 2.26 generation; the preceding line was `1.26.x`.

**v2.26.0 is a breaking release.** Every command now travels in a correlation envelope and every executor acknowledges it; bare command payloads from 1.26.x are no longer valid. See [RELEASE-2.26.0.md](RELEASE-2.26.0.md) for what changed, why, and how to migrate.

## Machine-Readable Schemas

`@umka-management/protocol` publishes the wire and REST contracts as zod schemas with inferred TypeScript types, MQTT topic builders/parsers, and generated JSON Schema documents. Where the package and this specification disagree, **the specification governs**.

## Reference Implementation

[Umka Player](https://github.com/Maugry/Player) is the MIT-licensed reference implementation of this standard.

## License

This specification is an **open standard**. Any implementation following this specification is considered Umka-compatible.

Developed by [Multimedia Solutions Lab Ltd.](https://maugry.ru)

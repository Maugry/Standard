# Umka Kiosk Standard v2.26.0

**Released:** July 2026
**Status:** Breaking. Not backward compatible with 1.26.x.
**Previous:** v1.26.5.1 (May 2026)

---

## In one paragraph

Every command on the wire now carries a correlation id, and every executor must answer for it. That single change makes the difference between a system that reports *"command sent"* and one that reports *"the kiosk applied it"*, *"the bridge refused it"*, or *"nobody answered"*. It is also incompatible with everything that came before: a 1.26.x publisher's payloads are rejected by a 2.26.0 consumer, and vice versa. Alongside it, the IoT model opens up so new device types no longer require a spec release, kiosks now describe their own hardware, and five topics that shipped in implementations over the past year are finally written down.

---

## Why this is a major release

Two independent reasons landed together, which is why the version jumps rather than increments.

**The wire genuinely broke.** The command envelope is not an additive field. A command published without one is rejected, so a 1.26.x server cannot drive a 2.26.0 kiosk and a 2.26.0 server cannot drive a 1.26.x kiosk. Mixed fleets do not degrade gracefully; they stop responding. A protocol change of that shape has to be visible in the version number, and only a major bump makes it so.

**The product moved to a new generation.** The 2.26 line was declared across the product in July 2026. The Standard is the public, third-party-visible artefact of the system — leaving it on `1.26.x` while everything it specifies shipped as `2.26.x` would misrepresent what the specification describes.

These two reasons are worth separating, because a major bump justified *only* by the second would be a problem. A version number's job is to describe compatibility; renaming a spec to match a marketing generation makes the number lie about it. Here it doesn't — the break is real, and the epoch and the incompatibility coincide. Had they not, the honest move would have been to leave the number alone.

The version format also drops from four segments to three: `1.26.5.1` → `2.26.0`, now `{major}.{year}.{patch}`. The fourth segment existed to distinguish editorial revisions from substantive ones, a distinction the version history already carries in prose.

---

## What breaks

### Every command payload is now wrapped

Commands on `commands/*`, `wol`, and IoT `command` topics travel inside `{commandId, issuedAt, value}`.

Before (v1.26.x), volume on `umka/kiosks/main-hall/commands/volume`:

```json
80
```

After (v2.26.0):

```json
{
  "commandId": "3f2b8c1e-9a4d-4f7e-b2c1-8e5a0d6f3b91",
  "issuedAt": "2026-07-19T10:30:00.000Z",
  "value": 80
}
```

Every command is affected, including scalar ones — power (`"off"`), locale (`"ru"`), loop (`true`), WoL (a MAC address).

**There is no dual-format transition period, and this is deliberate.** Accepting both shapes would mean accepting untracked commands indefinitely, which defeats the guarantee the envelope exists to provide — and a bare payload is indistinguishable from a truncated or malformed one, so tolerating it means executing corrupted input. Deployments upgrade servers and kiosks together.

### Executors must acknowledge

New topics: `umka/kiosks/{slug}/ack` and `museum/{museumId}/iot/{deviceId}/ack`.

An executor publishes `{commandId, source, result, detail?, timestamp}` after acting, where `result` is `received`, `applied`, `failed`, or `unsupported`. Issuers conclude `timeout` themselves when no terminal ack arrives within 30 s.

**Silence is no longer conformant.** An executor that receives an action it doesn't implement must answer `unsupported`. Previously the guidance was to log and ignore, which left an unimplemented action indistinguishable from an offline device — the caller waits either way and learns nothing.

### Executors must reject stale commands

A command whose `issuedAt` is older than 120 s must be discarded. This is what stops a `power: off` that queued at the broker overnight from firing the moment a kiosk reconnects in the morning.

The check **fails open**: an unparseable or future-dated timestamp counts as *not stale*. A stale-check that fails closed converts a few seconds of clock drift into a kiosk that silently ignores every command it receives — a far worse failure than the one it prevents.

### Ids may be numbers

`mediaId`, `currentContent.id`, and action-entry targets arrive as **either a JSON string or a JSON number**. Consumers must accept both.

This one bites quietly. Content systems typically assign integer keys to top-level records and generated string ids to embedded rows, so the same logical id arrives as either type depending only on how the server stored it. An implementation that types these as string-only doesn't error — validation fails before dispatch and the command is discarded with no visible trace. It was already true on the wire before this release; v2.26.0 makes it explicit.

---

## Migration checklist

1. **Grant the new topics in your broker ACL** — `ack`, `iot/+/ack`, `commands/lock`, `commands/update`, `system/update-status`, `system/logs`. A missing grant is silent: the broker drops the publish and the client sees no error, so the feature simply does nothing. Verify by observing traffic, not by reading the ACL file.
2. **Update the server to issue envelopes** — generate a UUID and an ISO timestamp per command.
3. **Update every executor together** — Player, Supervisor, Bridge, WoL relay. Partial upgrades produce a fleet that accepts no commands.
4. **Subscribe to the ack topics** and reconcile by `commandId`; record `timeout` when nothing terminal arrives.
5. **Widen id handling** to accept numbers, normalising to string.
6. **Stop stripping unknown IoT readings** anywhere in the path — bridge, cache, REST projection, UI.
7. **Verify against a real device before rollout.** The failure mode of an incomplete migration is a kiosk that looks healthy — heartbeats, status, everything green — and ignores every command.

---

## What's new (non-breaking)

**The IoT capability model is open.** Devices may publish scalar readings the Standard doesn't name, and receive actions it doesn't name. Previously, integrating a device with one unnamed reading — a CO₂ sensor, a gateway register, a leak detector — meant a spec revision, a package release, and a coordinated redeploy, while the device was already sending the data and a strict schema was discarding it. Now it's a configuration change.

The trade is stated explicitly in the spec: unknown readings carry **no agreed semantics**. Two deployments may both publish `co2` and mean different units. Display them, store them, alert a human — but don't build automated behaviour on a key nobody has agreed the meaning of. Promoting a reading into the well-known set is exactly the act of granting it meaning, and remains a spec change. `humidity` is promoted in this release.

Unknown keys must hold **scalar** values; a non-scalar unknown key is dropped individually rather than rejecting the whole status. That keeps one malformed reading from blinding an operator to a device's `state`, while stopping a compromised device from pushing arbitrary structures into consumer storage.

**Kiosks describe their own hardware.** The heartbeat now carries `hardware: {touch, screens, appVariant}` — the kiosk-side counterpart of the device capability model. A server can detect that a kiosk configured for Browse has no touch input, or that a two-screen machine is configured for one output.

`touch` is deliberately tri-state: absent means *unknown*, not `false`. Treating unknown as `false` would block Browse mode on every kiosk running an older Player. Drift between reported hardware and configured intent is a **configuration warning, not a fault** — the kiosk is working as built.

**Renderer liveness.** `liveness.renderTickAgeMs` on the heartbeat exposes a failure that was previously invisible: a Player process alive, connected, and heartbeating normally while its renderer is frozen and the visitor sees a static screen.

It is **alert-only, and the spec says so normatively.** A frozen renderer and a legitimately idle one are not reliably distinguishable from this number, so an automatic restart keyed on it will eventually restart healthy kiosks in front of visitors.

**Topics that shipped but were never written down:**

| Topic | Purpose |
|---|---|
| `commands/lock` | Suppress visitor input while continuing to render and report |
| `commands/update` + `system/update-status` | Update dispatch and progress |
| `system/logs` | Chunked, credential-redacted log retrieval |
| `ack` / `iot/…/ack` | Command acknowledgement |

Two normative points on updates: kiosks **must not poll** an update feed (a fleet that self-updates on discovery is a fleet whose version distribution nobody controls), and `rollback` must be permitted to downgrade (otherwise recovering a bad release requires physical access to every machine).

**Other additions:** `showcase_item` playback command; `status.locked`; heartbeat `instanceId` for duplicate-slug detection and `content` for sync state; supervisor `circuitBreaker` state, persisted across restarts and reported on a retained topic so a quarantined kiosk isn't mistaken for a healthy one; `send-logs`; the optional *Named-action vocabulary* for orchestration definitions, in which `freeze`/`unfreeze` are composed over the `lock` primitive rather than existing on the wire.

---

## What did not change

Operating modes, local-first architecture, sync semantics, two-tier liveness derivation, heartbeat cadences and staleness windows, the trigger pipeline, fleet orchestration, and the REST surface are all unchanged. If your implementation is conformant to v1.26.5.1 in those areas, it remains conformant.

The content-package format is now explicitly marked **advisory** rather than pinned. It is the least settled part of the Standard — consumed as a type rather than schema-validated, and legitimately extended by authoring systems. Implementations should ignore unrecognised members rather than reject a package. Saying so is more honest than implying a precision the format doesn't have.

---

## Machine-readable schemas

The wire and REST contracts are published as `@umka-management/protocol` — zod schemas with inferred TypeScript types, MQTT topic builders and parsers, validating parse helpers, and generated JSON Schema documents. Implementations in other languages are equally conformant; the JSON Schema artefacts exist so they need not re-derive the shapes by hand.

**Where the package and the specification disagree, the specification governs.** The package is an implementation of the spec, not a second source of truth.

---

## Full specification

[STANDARD.md](STANDARD.md) — see *Version history* for the complete itemised change list.

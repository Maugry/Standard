# Umka Kiosk Standard v1.26.5.1

**Status:** Active
**Date:** May 2026
**License:** Open Standard (Implementation: MIT)

---

## Overview

The Umka Kiosk Standard defines a universal protocol for museum multimedia kiosk systems. This standard covers:

- Operating modes and their configurations
- Real-time control protocol (MQTT)
- Content synchronization (REST API)
- Local-first content architecture
- Integration with guide tablets, IoT devices, and physical triggers
- Two-tier liveness (player + supervisor) and graceful-offline semantics
- Optional event→action bindings (kiosk state drives kiosk/IoT commands)
- Optional fleet orchestration for museum/hall-wide power

### Design principles

1. **Local-first** — Kiosks always operate from locally stored content. Server connectivity is used for synchronization and remote control, not for primary operation.
2. **CMS-agnostic** — The standard defines the wire contract between kiosk, supervisor, and server. The server-side CMS implementation is not prescribed.
3. **Hardware-agnostic** — The player runs on a Windows PC and supports various output devices: monitors, TVs, touchscreens, LED panels, projectors, and audio systems.
4. **Standards-based** — Any implementation following this spec is Umka-compatible. Implementations will share protocol code by design.
5. **Two-tier liveness** — Player health and hardware health are reported on independent MQTT topics. Consumers MUST combine them into a derived state (see *Two-tier liveness*).
6. **Payload-shape disambiguation on shared topics** — Some topics are subscribed by two independent consumers (e.g. supervisor + player on `commands/app`). They disambiguate by payload shape; the standard pins both shapes.

### Terminology

| Term | Definition |
|------|-----------|
| **Umka** | Museum kiosk management system: server (CMS), admin interface, and guide tablet app. |
| **Player** | The kiosk-resident application that renders content (also "kiosk app"). |
| **Supervisor** | The control-plane service running on the same machine as the Player, responsible for lifecycle, OS power commands, and hardware-tier reporting. Reference implementation: Sentinel. |
| **Loop Mode** | Player operating mode with automatic cyclic playback of a media playlist. |
| **Browse Mode** | Interactive Player operating mode with a menu/catalog for visitor self-service content selection. |
| **Custom Mode** | Player operating mode with non-standardised functionality (e.g. interactive game). |
| **Screensaver** | Visual representation of the idle state. |
| **Content Package** | Collection of content (media files, settings) assigned to a specific kiosk. |
| **IoT Device** | Physical device connected via MQTT that implements a subset of the standard IoT capabilities (on/off, dimming, colour, temperature, scenario stepping). Lights, dimmers, ACs, buttons, sensors, and scenario controllers are common examples — the "type" is descriptive, not a wire concept (see *Device capability model*). |
| **Physical Trigger** | Hardware button (an IoT device of type `trigger`) that, when pressed, causes a kiosk to play associated media. |
| **Guide Tablet** | Tablet/phone web application for museum guides to monitor and control kiosks during tours. |
| **Bridge** | Service that translates between MQTT and a physical fieldbus (Modbus RTU-over-TCP in the reference implementation). |

---

## Operating modes

Umka defines three base operating modes. Each mode may be specialised through configuration profiles documented below.

### Loop Mode

Automatic cyclic playback of a media playlist (video, images, audio). Playlist order and image display duration are configured in the CMS.

**Common behaviour:**
- Plays content from an ordered playlist.
- Loops back to the beginning after the last item.
- Remote control via guide tablet (MQTT).
- Content synchronised from server, played from local storage.

**Configuration profiles:**

| Profile | Use case | Notes |
|---------|----------|-------|
| **Continuous** | LED panels, video walls, non-interactive displays | Automatic playback in an infinite loop. No user interaction. |
| **Interactive** | Touchscreen kiosks with video playback | Screensaver in idle; "Start" button begins playback; "Back" returns to screensaver; controls auto-hide. |
| **Triggered** | Exhibits with physical buttons near displays | Idle until an MQTT trigger arrives; plays once and returns to idle. No on-screen "Start". |
| **Audio-only** | Background music in museum halls | No video output. Plays audio playlist. Each hall is a separate audio kiosk. |
| **Projector** | Short-throw projectors, projection mapping | Completely passive — no on-screen controls. Triggered by MQTT or IoT button. |

These profiles are **configuration patterns**, not wire modes. The kiosk reports the base mode `loop` on the `mode` field regardless of profile; the profile is realised through content-package configuration (presence of a screensaver, trigger bindings, audio-only playlist, etc.), not through a distinct `mode` value. See *Operating modes on the wire* under the MQTT protocol.

### Browse Mode

Interactive mode with a menu/catalog for visitor self-service content selection.

**Common behaviour:**
- Hierarchical menu navigation with unlimited nesting depth.
- Multiple content types (video, articles, image galleries, showcase grids).
- Touch-optimised interface.
- Idle timeout returns to screensaver.
- Remote control via guide tablet (MQTT).

**Configuration profiles:**

| Profile | Use case |
|---------|----------|
| **Catalog** | Default — grid of cards with images and titles; tap to view detail. |
| **Showcase** | Carousel/grid of still images with captions; lighter than full Browse. Realised through `showcaseItems` content, not a distinct `mode` value — the kiosk reports `mode: "browse"`. |

Browse Mode requires:
- A screensaver during idle (carousel of images/video; configurable title, subtitle, "Start" button).
- A back stack — visitors MUST be able to return to the previous menu.
- Full-screen media viewing for video and images.

### Custom Mode

Non-standardised functionality. Custom Mode applications implement their own UI and logic but integrate with the Umka system for monitoring and power management.

**Use case:** Interactive games, custom interactive experiences.

**Required Umka integration:**
- Heartbeat publishing every 10 seconds.
- Power management commands (shutdown, reboot) via the Supervisor.
- Status reporting (online/offline; the player-level heartbeat is the marker).

**Optional integration:**
- Content synchronisation.
- Guide tablet control.
- Idle timeout and auto-reset.

Custom Mode implementations MAY report a more specific mode string in their status (e.g. `mode: "game"`); consumers that don't recognise the string MUST treat it as Custom.

---

## Functional requirements

### General (all modes)

- Display of museum logo (configured in CMS).
- Configurable inactivity timeout before returning to idle state.
- Automatic reset to default state on user inactivity.
- Smooth transitions between playlist items (Loop mode).
- Adaptive layout for different screen resolutions and orientations.

### Guide Mode

The kiosk in any base mode (Loop, Browse) supports remote control from a guide tablet via MQTT. **Guide controls are NOT displayed on the kiosk itself.**

**Guide capabilities:**
- Select and launch media content on a kiosk.
- Access content marked guide-only (hidden from visitors).
- Playback control: pause, play, seek, loop toggle.
- Volume control.
- Locale switching.
- Exit guide mode (return kiosk to default content).

### Guide-only content

Content marked `guideOnly: true`:
- **Hidden** from visitor-facing displays in all modes.
- **Accessible** only via guide tablet control.
- **Filtered** automatically from playlists and menus before render.
- **Stored** locally alongside regular content for offline guide access.

### Idle timeout and auto-reset

1. The Player tracks user activity (screen touches), including when video is paused.
2. On reaching the inactivity timeout, a warning appears with countdown (e.g. "Are you still here?"). The warning MUST fire some seconds before the timeout (reference implementation: 30 s).
3. The warning includes a "Stay" button — pressing it dismisses the warning and resets the timer.
4. If the user does not respond before countdown expires, the kiosk returns to default state.
5. On reset: volume and video position revert to defaults.

Inactivity timeout and warning lead time are configurable per kiosk.

### Physical triggers

A physical trigger is an IoT device of type `trigger` (typically a button wired to a Modbus discrete input). When pressed, the Bridge publishes a status update on the IoT topic (`state: "on"`, or numeric `1`). The CMS observes this transition and dispatches a `trigger_play` to the configured kiosk.

The full lifecycle is defined in *Trigger pipeline* below.

### Multi-language support

- Dynamic locale switching via MQTT command on `commands/locale`.
- Localised content delivery from CMS.
- Persistence of locale is implementation-specific — consumers MUST NOT assume the kiosk retains locale across reboots unless the kiosk's status reports its current locale on startup.

**Standard locales:** `ru` (Russian), `en` (English). Additional locales can be added per museum.

> **Implementation note.** The reference CMS does not currently expose a REST action that maps to `commands/locale` — implementations needing operator-driven locale switching must either add a CMS action that publishes to this topic, or have the locale set as part of the kiosk's content-package configuration. Direct MQTT publish is possible from any backend service but bypasses the "writes through CMS" architecture documented below.

---

## Local-first architecture

Umka kiosks follow a **local-first** architecture. The kiosk always operates from locally stored content. Server connectivity is used for synchronisation, not for primary operation.

```
┌─────────────────────────────────────────────────┐
│                  KIOSK                            │
│                                                   │
│  Local Storage ──→ Playback Engine                │
│       ↑                                           │
│       │ (background sync when server available)   │
│       ↓                                           │
│  Sync Service ──→ Server (CMS)                    │
│                                                   │
└─────────────────────────────────────────────────┘
```

- Content is **always played from local storage**, regardless of server connectivity.
- Sync runs in the background and does not interrupt playback.
- Server going offline has no immediate effect on kiosk operation.
- New content becomes available only after successful sync + local cache.

### Synchronisation

Sync is **trigger-based**, not polling. The kiosk downloads content when one of these triggers fires:

1. **Boot.** The kiosk SHOULD attempt sync once on startup. This is a best-effort call; failure is silent and the kiosk falls back to cached content.
2. **MQTT command.** The CMS publishes `{ "action": "sync" }` on `umka/kiosks/{slug}/commands/app`. This happens automatically when a content package is saved or when an operator clicks "Sync" in the admin UI.

There is **no automatic periodic sync**. The kiosk MUST NOT poll the CMS on a timer.

**What hits the server:**
1. Package metadata — always fetched from CMS REST API as a single document with media references resolved (`depth=3` or equivalent). Always a server call.
2. Media files — each file is checked against the local cache:
   - File exists locally (by media ID) AND size matches → **skip (no server call)**.
   - File missing or size differs → **download from server**.

**What stays local:**
- Playback always reads from local storage. The Player **never streams from the server** during normal operation.
- If the server goes down mid-playback, nothing changes — content is already local.

**Sync behaviour:**
- Background download does not interrupt current playback.
- The Player switches to new content after all files are cached, in a single atomic transition.
- Failed downloads are skipped (other files still sync); errors are logged.
- Single-flight: a `sync` command received while a sync is in flight MUST be ignored or coalesced into the in-flight call.

**Sync confirmation is not required.** Earlier versions of this standard defined `POST /api/kiosks/{slug}/sync-complete`. That endpoint has been removed; sync completion is observed via the next `status` message from the kiosk.

### Connectivity loss

When server connection is lost:

1. Content continues playing from local storage (no interruption).
2. Heartbeat publishing continues; the broker buffers if unreachable. The CMS marks the kiosk offline after its staleness window elapses.
3. MQTT remote control is unavailable until reconnection.
4. Auto-reconnect attempts continue in the background.
5. On reconnection: heartbeats resume and the kiosk awaits a sync command.

**Offline-mode supervisor opt-out.** A Supervisor implementation that has never observed a Player heartbeat in the current session MUST NOT restart the Player on heartbeat-timeout, because the absence of heartbeats is an expected consequence of an offline kiosk. The Player itself remains running and continues to serve cached content. See *Supervisor*.

### Cache management

Implementations SHOULD provide a cache-eviction mechanism for media files that no longer appear in any active content package. Unbounded growth is a known operational hazard on long-running kiosks.

---

## MQTT protocol

### Topic structure

**Format:** `umka/kiosks/{kioskSlug}/{category}/{type}`

Where `{kioskSlug}` is the unique text identifier for each kiosk.

**Categories:**
- `commands/*` — Server/Guide → Player or Supervisor (control inbound to the kiosk).
- `system/*` — Supervisor → Server (publishes from the control plane).
- `status` — Player → Server (state reporting).
- `heartbeat` — Player → Server (app-tier health).
- `wol` — Server → Wake-on-LAN relay (power-on of an off kiosk).

> **Breaking change from v1.26.3.2.** Earlier versions documented system-level commands on `system/power` and `system/app` with JSON payloads of shape `{ "command": "..." }`. The wire reality is `commands/power` and `commands/app` with **bare-string payloads**. The `system/*` prefix is reserved for *publishes from the Supervisor*. The standard now matches the implementation; implementations conforming to v1.26.3.2 are not compatible with v1.26.5.0.

### Command topics (Server/Guide → Player or Supervisor)

#### Content sync and mode

**Topic:** `umka/kiosks/{kioskSlug}/commands/app`
**Subscribers:** Player (acts on JSON `{action, value}` payloads), Supervisor (acts on bare-string payloads).

| Payload (JSON) | Acted on by | Description |
|---|---|---|
| `{ "action": "sync" }` | Player | Trigger content resync from CMS. |
| `{ "action": "mode", "value": "<mode>" }` | Player | Change operating mode without resyncing content. |
| `{ "action": "quit" }` | Player | Graceful quit, used by Supervisor on Windows shutdown. |

| Payload (bare string) | Acted on by | Description |
|---|---|---|
| `"start"` | Supervisor | Launch the Player application. |
| `"stop"` | Supervisor | Stop the Player application; the Supervisor MUST NOT auto-restart after an intentional stop. |
| `"restart"` | Supervisor | Stop and relaunch the Player application; the Supervisor MUST clear any restart counters and bypass the circuit breaker. |

The two payload shapes are how the topic is shared between two consumers. A Player implementation MUST ignore payloads it cannot parse as `{action, value}` JSON; a Supervisor implementation MUST ignore payloads that do parse as JSON of that shape.

#### Operating modes on the wire

**Valid `mode` values:** `loop`, `browse`, `custom`. There is no separate `profile` value on the wire — Loop/Browse configuration profiles (Continuous, Interactive, Triggered, Audio-only, Projector, Catalog, Showcase) are realised through content-package configuration, not a `mode` string.

A Custom kiosk MAY report a more specific specialisation string (e.g. `game`) instead of the bare `custom`; consumers that don't recognise the string MUST treat it as `custom`. The reference CMS labels its custom-mode kiosks `game` — this is exactly such a specialisation and carries no CMS-driven content flow.

#### Playback control

**Topic:** `umka/kiosks/{kioskSlug}/commands/playback`
**Subscriber:** Player.
**Payload:** JSON `{action, ...}`.

| Payload | Description |
|---------|-------------|
| `{ "action": "play" }` | Resume playback. |
| `{ "action": "play", "mediaId": "uuid" }` | Play specific media by ID. |
| `{ "action": "content", "contentId": "uuid" }` | Open an addressable content node by ID — a `MenuItem` or a `ShowcaseItem` (Browse mode). See *Addressable content node*. |
| `{ "action": "screensaver" }` | Force the kiosk into its screensaver/idle state. Used by guide control and fleet orchestration (see *Fleet orchestration*). |
| `{ "action": "pause" }` | Pause current playback. |
| `{ "action": "stop" }` | Stop and return to idle/screensaver. |
| `{ "action": "next" }` | Next item in playlist/gallery. |
| `{ "action": "prev" }` | Previous item in playlist/gallery. |
| `{ "action": "home" }` | Return to main menu (Browse mode). |
| `{ "action": "seek", "value": <seconds> }` | Seek to a position in the current media. |
| `{ "action": "trigger_play", "mediaId": "uuid", "mediaUrl": "...", "mediaMimeType": "...", "mediaTitle": "..." }` | Play media in response to a physical trigger. The full envelope is provided so the Player can render without a separate CMS lookup. See *Trigger pipeline*. |

**Media resolution order for `mediaId`:**
1. Playlist items.
2. Guide-only content.
3. Menu item video attachments.

#### Volume

**Topic:** `umka/kiosks/{kioskSlug}/commands/volume`
**Subscriber:** Player.
**Payload:** integer `0`–`100` (bare number, not JSON-wrapped).

#### Locale

**Topic:** `umka/kiosks/{kioskSlug}/commands/locale`
**Subscriber:** Player.
**Payload:** ISO 639-1 code as a bare string (e.g. `"ru"`).

#### Loop toggle

**Topic:** `umka/kiosks/{kioskSlug}/commands/loop`
**Subscriber:** Player.
**Payload:** boolean (bare `true`/`false`).

Toggles whether current playback loops or plays once.

#### Power

**Topic:** `umka/kiosks/{kioskSlug}/commands/power`
**Subscriber:** Supervisor.
**Payload:** bare string.

| Payload | Description |
|---|---|
| `"off"` | Shut down the OS. |
| `"shutdown"` | Alias for `"off"`. |
| `"reboot"` | Reboot the OS. |

> Wake-on-LAN is published on `wol`, not `commands/power` — see below.

#### Wake-on-LAN

**Topic:** `umka/kiosks/{kioskSlug}/wol`
**Subscriber:** A WoL relay service on the museum LAN (the kiosk itself is powered off and cannot subscribe).
**Payload:** the kiosk's MAC address as a bare string.

The relay sends a magic packet to the address. There is no acknowledgement; success is inferred from a subsequent Supervisor heartbeat after boot.

### Status topics (Player → Server)

#### Kiosk status

**Topic:** `umka/kiosks/{kioskSlug}/status`
**QoS:** 0
**Retain:** true
**Published on:** state change, content switch, volume change, mode change, locale change, error, trigger completion.

**Payload:**
```json
{
  "kioskId": "kiosk-1-1",
  "state": "playing",
  "mode": "loop",
  "volume": 80,
  "locale": "ru",
  "currentContent": {
    "type": "video",
    "id": "uuid",
    "title": "Content Title",
    "position": 45.2,
    "duration": 120.5
  },
  "navigation": {
    "nodeId": "uuid",
    "path": ["root-uuid", "section-uuid", "uuid"],
    "showcaseOpen": false
  },
  "screensaverActive": false,
  "timestamp": "2026-05-25T10:30:00Z",
  "version": "1.0.0",
  "uptime": 3600,
  "error": null,
  "triggerEnded": false
}
```

**Error-state example:**
```json
{
  "kioskId": "kiosk-1-1",
  "state": "error",
  "mode": "browse",
  "volume": 0,
  "locale": "ru",
  "timestamp": "2026-05-25T14:32:00Z",
  "version": "1.0.0",
  "uptime": 0,
  "error": {
    "code": "INDEXEDDB_OPEN_FAILED",
    "message": "DOMException: Internal error",
    "timestamp": "2026-05-25T14:32:00Z"
  },
  "triggerEnded": false
}
```

**Fields:**

| Field | Type | Values |
|---|---|---|
| `kioskId` | string | Stable kiosk identifier (typically the slug). |
| `state` | string | `idle`, `playing`, `paused`, `loading`, `error`. |
| `mode` | string | `loop`, `browse`, `custom`, `off`. A Custom kiosk MAY report a specialisation string (e.g. `game`); consumers MUST treat unrecognised values as `custom`. |
| `volume` | integer | 0–100. |
| `locale` | string | ISO 639-1 code. |
| `currentContent` | object or null | Current playback info; `null` when idle. |
| `currentContent.type` | string | `video`, `article`. (Implementations MAY emit additional values; consumers MUST treat unknown values as opaque.) |
| `currentContent.position` | number (optional) | Playback position in seconds. |
| `currentContent.duration` | number (optional) | Total duration in seconds. |
| `navigation` | object (optional) | **Browse mode only.** The visitor's current position in the catalog. Omit in Loop/Custom. Reported as retained state so it self-heals after a dropped message; see *Addressable content node* and *Event→action bindings*. |
| `navigation.nodeId` | string or null | `id` of the addressable content node the visitor is currently on (`null` at the menu root). |
| `navigation.path` | string[] (optional) | Ancestor chain of node `id`s from root to `nodeId` inclusive. Lets a consumer bind at any granularity by testing id membership, without traversing the tree itself. |
| `navigation.showcaseOpen` | boolean (optional) | `true` while a showcase grid/carousel is open. |
| `screensaverActive` | boolean (optional) | `true` while the kiosk is showing its screensaver. Distinct from `state: "idle"`, which a Browse menu at rest may also report. |
| `timestamp` | string | ISO 8601. |
| `version` | string | **Always present.** Player version (semver). |
| `uptime` | integer | **Always present.** Seconds since Player start. |
| `error` | object or null | **Always present.** `null` when healthy; `KioskError` when `state` is `error`. |
| `triggerEnded` | boolean (optional) | Set to `true` exactly once when the Player finishes playing media that was launched by a physical trigger. See *Trigger pipeline*. |

**`KioskError` object:**

| Field | Type | Description |
|---|---|---|
| `code` | string | Machine-readable error code. |
| `message` | string | Human-readable error description. |
| `timestamp` | string | ISO 8601 timestamp of when the error occurred. |

**Reserved error codes.** When the listed conditions occur, implementations SHOULD emit the matching code so that monitoring/alerting layers can dispatch on a stable vocabulary. The reference Player currently emits only `INDEXEDDB_OPEN_FAILED_PERMANENT`; the rest are reserved for future emitters and SHOULD NOT be repurposed:

| Code | Condition |
|---|---|
| `INDEXEDDB_OPEN_FAILED` | Storage layer cannot initialise (file lock, corruption). The Player MAY attempt recovery before publishing. |
| `INDEXEDDB_OPEN_FAILED_PERMANENT` | Storage recovery has been attempted the maximum number of times and failed; the Player cannot operate. |
| `CONTENT_SYNC_FAILED` | CMS unreachable or API error during content sync. |
| `MEDIA_DOWNLOAD_FAILED` | Media file download error. |
| `SETTINGS_LOAD_FAILED` | No valid settings file found. |

Implementations MAY define additional codes; consumers MUST treat unknown codes as opaque error markers and surface `message` to the operator.

#### Player heartbeat

**Topic:** `umka/kiosks/{kioskSlug}/heartbeat`
**QoS:** 0
**Retain:** false
**Interval:** 10 seconds.

**Payload:**
```json
{
  "kioskId": "kiosk-1-1",
  "timestamp": "2026-05-25T10:30:00Z",
  "version": "1.0.0",
  "uptime": 3600
}
```

| Field | Type | Description |
|---|---|---|
| `kioskId` | string | Kiosk identifier. |
| `timestamp` | string | ISO 8601. |
| `version` | string | Player version (semver). |
| `uptime` | integer | Seconds since Player start. |

### Supervisor topics

The Supervisor publishes hardware-tier liveness and crash events on a separate topic namespace, and subscribes to power and app-lifecycle commands.

#### Supervisor system heartbeat

**Topic:** `umka/kiosks/{kioskSlug}/system/heartbeat`
**QoS:** 0
**Retain:** true (so a fresh subscriber sees current state immediately).
**Interval:** 10 seconds.

**Payload (running):**
```json
{
  "kioskId": "kiosk-1-1",
  "timestamp": "2026-05-25T10:30:00Z",
  "version": "1.0.0",
  "uptime": 3600,
  "player": {
    "status": "running",
    "pid": 1234,
    "lastHeartbeat": "2026-05-25T10:30:00Z",
    "restartCount": 0,
    "lastCrash": null,
    "crashedVersion": null
  },
  "system": {
    "cpuPercent": 45,
    "memoryPercent": 62,
    "networkConnected": true
  }
}
```

**Payload (graceful offline, on clean shutdown):**
```json
{
  "kioskId": "kiosk-1-1",
  "timestamp": "2026-05-25T10:30:00Z",
  "status": "offline",
  "graceful": true
}
```

This graceful payload is published with `retain: true, qos: 1` immediately before the Supervisor disconnects. Consumers MUST distinguish a *graceful* offline (operator-initiated shutdown) from an *unexpected* offline (LWT fired) — only the graceful payload carries `graceful: true`.

**Player status values:** `running`, `stopped`, `updating`, `unresponsive`, `restarting`.

**Last Will & Testament (LWT).** The Supervisor MUST register an LWT on this topic with payload:

```json
{
  "kioskId": "kiosk-1-1",
  "status": "offline",
  "connectedAt": "<supervisor-start-time>"
}
```

retained, QoS 1. On unexpected disconnect (network drop, hardware failure, supervisor crash), the broker publishes this payload — the absence of the `graceful: true` flag is the signal that the kiosk dropped off unexpectedly.

#### Supervisor crash event

**Topic:** `umka/kiosks/{kioskSlug}/system/crash`
**QoS:** 1
**Retain:** false
**Published on:** each Supervisor-initiated recovery attempt (after a Player crash or heartbeat timeout).

**Payload:**
```json
{
  "kioskId": "kiosk-1-1",
  "timestamp": "2026-05-25T10:30:00Z",
  "restartAttempt": 1,
  "maxRestarts": 3,
  "reason": "heartbeat_timeout"
}
```

`reason` is `"crash"` (Player process exited) or `"heartbeat_timeout"` (Player process alive but heartbeat stale). `restartAttempt` is the recovery attempt count within the current healthy window; `maxRestarts` is the circuit-breaker threshold from configuration.

### IoT device topics

IoT devices use a separate topic namespace scoped by museum.

**Format:** `museum/{museumId}/iot/{deviceId}/{type}`

Where `{museumId}` is the museum slug and `{deviceId}` is the device's unique identifier.

#### Device capability model

**A device's "type" is not a wire concept.** The protocol is *capability-based*: a device implements whichever of the standard actions and status fields apply to it, and nothing more. The only universal field is `state` (`on`/`off`/`unknown`); every other field is an optional capability a device opts into.

| Capability | Command action(s) | Status field(s) | `value` |
|---|---|---|---|
| On/off | `on`/`off`/`toggle` (+ `power_on`/`power_off` aliases) | `state` | — |
| Dimming | `brightness` | `brightness` | integer 0–100 |
| RGB colour | `color`, `scene` | `color`, `scene` | hex string; integer scene number |
| Temperature setpoint | `temperature` | `temperature` | integer °C |
| Scenario stepping | `reset`, `step` (→ wire `activation`) | `cur_step` | integer step number |

A device declares which capabilities it has to the CMS out of band (device config); the wire then carries only the actions and fields that device actually uses. The CMS device registry also holds **descriptive metadata** — a human name, a room/hall, and a `type`/category label — which is what UIs use to pick an icon, group devices, and decide which controls to render. This metadata is presentation-only: it never appears on the wire and carries no protocol meaning, so a deployment can add a new `type` label (and its icon) purely in CMS config without touching this standard. **Common devices map onto these capabilities** (illustrative, not an enumeration): a plain *light* uses on/off; a *dimmer* adds dimming; an *RGB fixture* adds colour/scene; an air *conditioner* uses on/off + temperature setpoint; a *temperature sensor* reports `temperature` read-only; a *physical trigger* (button) uses the on/off `state` only (see *Trigger pipeline*); a *scenario controller* uses scenario stepping.

Devices whose behaviour does not map onto the capabilities above are **out of scope** — like fieldbus protocols, they are integrated privately by the implementation until (and unless) a new capability is standardised here. Adding such a device does **not** require enumerating it in this standard.

#### IoT device status (Device → Server)

**Topic:** `museum/{museumId}/iot/{deviceId}/status`
**QoS:** 0
**Retain:** true
**Frequency:** On state change or at the bridge's polling cadence (configured per-device; the reference Bridge default is every 2 s).

**Payload:**
```json
{
  "deviceId": "light-living-01",
  "state": "on",
  "brightness": 80,
  "color": "#ffffff",
  "scene": null,
  "temperature": 22,
  "cur_step": 0,
  "timestamp": "2026-05-25T10:30:00Z"
}
```

| Field | Type | Notes |
|---|---|---|
| `deviceId` | string | Device identifier. |
| `state` | string | **Always a string**: `"on"`, `"off"`, or `"unknown"`. Numeric register values from the fieldbus MUST be derived into one of these. |
| `brightness` | integer (optional) | 0–100; present for lights and dimmers only. |
| `color` | string (optional) | Hex colour e.g. `"#ff8800"`; present for RGB lights only. |
| `scene` | integer or null (optional) | Scene preset number; present for RGB lights only. |
| `temperature` | integer (optional) | Setpoint or measured temperature in °C; present for conditioners and temperature sensors. |
| `cur_step` | integer (optional) | Current step number; present for scenario-controller devices. Note the snake_case key — historical artefact of the bridge's YAML register names; preserved on the wire for back-compat. |
| `timestamp` | string (optional) | ISO 8601; the publisher's local time. |

**State derivation precedence** (when the underlying device reports numeric values):
1. If a `power` register is present, use it (treated as boolean).
2. Else if `brightness > 0`, state is `"on"`.
3. Else if a `state` register is present, use it (treated as boolean).
4. Else `"unknown"`.

#### IoT device command (Server → Device)

**Topic:** `museum/{museumId}/iot/{deviceId}/command`
**QoS:** 1
**Retain:** false

**Payload:** JSON `{ "action": "<action>", "value": <optional> }`.

| Action | Description |
|---|---|
| `on`, `power_on` | Turn device on. `power_on` is an alias for `on`. |
| `off`, `power_off` | Turn device off. `power_off` is an alias for `off`. |
| `toggle` | Toggle on/off. |
| `brightness` | Set brightness; `value` is integer 0–100. |
| `color` | Set RGB colour; `value` is a hex string e.g. `"#ff8800"`. |
| `scene` | Activate scene preset; `value` is an integer scene number. |
| `temperature` | Set temperature setpoint; `value` is an integer in °C. |
| `reset` | (Scenario devices only.) Reset the scenario back to step 0. The CMS rewrites the wire payload to `{"action":"activation","value":<stepsLength+1>}` before publishing. |
| `step` | (Scenario devices only.) Advance to a specific scenario step. The CMS rewrites the wire payload to `{"action":"activation","value":<step>}`. |

The `reset` and `step` actions are CMS-level abstractions over a single wire-level `activation` action. A device implementing this protocol receives `{"action":"activation","value":<n>}` regardless of which REST action was issued; the semantics fall out of the value.

**Internal-only wire actions.** The CMS may also publish `{"action":"state","value":0}` on `museum/{museumId}/iot/{deviceId}/command` to force-reset a stuck trigger's register (see *Trigger pipeline*). This is not exposed as a REST action — only the trigger-reset machinery emits it.

Unknown actions MUST be logged and ignored.

### Trigger pipeline

The full lifecycle from a button press to the Player resuming idle:

```
1. Visitor presses physical button.
2. Bridge polls the trigger's discrete-input register and observes
   state transition 0 → 1.
3. Bridge publishes museum/{m}/iot/{deviceId}/status with state: "on".
4. CMS observes the transition (clean 0 → 1 only; re-presses while
   trigger is active MUST be ignored).
5. CMS publishes umka/kiosks/{slug}/commands/playback with
   { "action": "trigger_play", "mediaId": "...", "mediaUrl": "...",
     "mediaMimeType": "...", "mediaTitle": "..." }.
6. Player plays the media to completion.
7. Player publishes umka/kiosks/{slug}/status with triggerEnded: true.
8. CMS observes triggerEnded, looks up the trigger device that
   armed this kiosk, and publishes
   museum/{m}/iot/{deviceId}/command with { "action": "state", "value": 0 }.
9. Bridge writes 0 to the trigger register; trigger is released.
```

**Fallback timers.** Because step 7 may never arrive (Player crash, MQTT drop, media file outlasting expectations), the CMS SHOULD schedule fallback resets at conservative intervals (reference implementation: 65 s and 185 s after step 5). On the second fallback the CMS additionally clears all server-side trigger state so a new press can be honoured.

**Re-presses.** If the visitor presses the same button while the trigger is active (between steps 5 and 9), the CMS MUST suppress the re-fire. Implementations track this via an in-memory active-trigger set.

### Event→action bindings (optional capability)

The trigger pipeline above is one instance of a general pattern: an event observed on the bus causes a command to be published. A CMS MAY generalise this into configurable **bindings** that map kiosk-reported events to outbound commands — for example, lighting a fixture when a visitor opens a specific exhibit.

**Triggering events** are any of:
- a physical-trigger transition (the *Trigger pipeline* above);
- a media play/stop, derived from the kiosk's `status` (`state` and `currentContent`);
- a Browse navigation change — a new `navigation.nodeId`/`navigation.path` or a `screensaverActive` transition on the retained `status`.

**Actions** are any documented kiosk command (*Playback control*, e.g. `content`, `screensaver`, `trigger_play`) or IoT command (*IoT device command*).

**Rules:**
- Bindings MUST be keyed on **content-package identifiers** — addressable-node `id`s (`MenuItem`/`ShowcaseItem`) and `mediaId`s — never CMS-internal document IDs, so a binding authored against a package survives CMS schema changes.
- A binding SHOULD be **state-coupled** when its event is a state rather than an edge: it is active while its node `id` appears in the kiosk's reported `navigation.path`, and reverts when the id leaves the path. This makes "light the section while the visitor is anywhere inside it" and "return to all-on when the visitor backs out" fall out of one rule, and is robust to a dropped message because `navigation` is retained state, not an edge event.
- The standard does **not** require any particular node type to be addressable, nor does it prescribe how bindings are stored, named, or grouped — that is implementation-private. An implementation that models exhibits as `MenuItem`s and one that models them as `ShowcaseItem`s are both conformant, provided the bound nodes expose stable ids and are reported in `navigation`.

> **Conformance.** This capability is OPTIONAL. The reference implementation currently wires only the physical-trigger→media case (*Trigger pipeline*); app-state→IoT and app-state→kiosk bindings are reserved by this section but not yet emitted by the reference CMS.

> **Latency.** An app-state→IoT binding round-trips through the CMS (kiosk `status` → CMS rule evaluation → IoT `command`), so it inherits broker latency and CMS availability. Deployments needing sub-frame touch-to-light reaction MUST measure this path on representative hardware before relying on it.

### Architecture: reads direct, writes through CMS

In production deployments, the guide tablet has a **read-only** relationship with MQTT. It subscribes directly to status topics for low-latency live updates. All commands go through the CMS REST API, which publishes the corresponding MQTT message.

```
IoT device ──MQTT──▶ broker ──MQTT──▶ guide-app (subscribe, read-only)
                         │
                         └──MQTT──▶ CMS (subscribe, status cache)

guide-app ──REST──▶ CMS ──MQTT──▶ broker ──MQTT──▶ IoT device / kiosk / supervisor
```

**Rationale:**
- The CMS is the single authoritative publisher for all commands.
- The CMS enforces authentication (JWT) and writes audit logs.
- The guide app gets sub-second live status without REST round-trips.
- Devices only trust one MQTT publisher identity.

A guide-app session, network glitch, or frontend bug cannot cause a spurious MQTT publish.

### Heartbeat cadences and staleness windows

| Tier | Publish cadence | Staleness window | Detected by |
|---|---|---|---|
| Player heartbeat | 10 s | 30 s (3 missed) | CMS marks kiosk software-offline. |
| Supervisor heartbeat | 10 s | 30 s (3 missed) | CMS marks kiosk hardware-offline. |
| IoT device status | per-device, default 2 s | 3 s | CMS flips IoT status to `unknown`. |

Implementations MUST publish kiosk and supervisor heartbeats at 10 s. The IoT publish cadence is per-device — the reference Bridge defaults to 2 s and accepts overrides between ~0.5 s and 10 s. The IoT staleness window is fixed at 3 s on the CMS side: a single missed poll at the default 2 s cadence already trips `unknown`, which is intentionally aggressive — IoT devices SHOULD therefore be configured at ≤ 1.5 s cadence in deployments where false-`unknown` ticks are costly.

### Two-tier liveness

Consumers MUST combine the Player heartbeat (`heartbeat`) and the Supervisor heartbeat (`system/heartbeat`) into a derived liveness state. The four canonical states:

| Derived state | Supervisor heartbeat | Player heartbeat | Meaning |
|---|---|---|---|
| `on` | fresh | fresh | Kiosk hardware up; Player running. |
| `sw-off` | fresh, `player.status: "stopped"` | stale or missing | Hardware up; Player intentionally or unintentionally stopped. |
| `error` | fresh, `player.status: "unresponsive"` | stale | Hardware up; Supervisor's recovery exhausted. |
| `off` | stale or `status: "offline"` | (irrelevant) | Hardware offline. The `graceful` flag on the most recent retained `system/heartbeat` distinguishes operator-initiated shutdown (`graceful: true`) from an unexpected drop (LWT fired, `graceful` absent). |

Consumers SHOULD render `off` with `graceful: true` differently from `off` without it — the former signals a kiosk that was powered off intentionally, the latter signals a fault or network issue.

---

## REST API

The standard defines the wire contract; the URL shapes given here are advisory. Implementations MAY use a collection-query convention (e.g. Payload-style `GET /api/kiosks?where[slug][equals]=...`) provided the semantics below are preserved.

**Base URL:** Configured per kiosk in the settings file.

**Authentication:**
- All command-issuing endpoints (`/api/kiosk-command`, `/api/iot-command`, `/api/power-scenario`) MUST require an authenticated session (JWT or equivalent).
- Read-only status caches (`/api/kiosk-status`, `/api/iot-status`) MAY be anonymous on the internal network.
- Content reads consumed by the kiosk (media, articles, content packages, the kiosk's own configuration) MAY be anonymous to allow boot-time fetches before any session exists.

### Kiosk configuration read

```
GET /api/kiosks?where[slug][equals]={kioskSlug}
```

Returns the kiosk configuration including the assigned `contentPackageId`, mode, locale, idle timeout, MAC address, and any per-kiosk settings.

### Content package read

```
GET /api/content-packages/{id}
```

Returns the complete content package with media references resolved sufficiently for the kiosk to download all referenced files. Implementations using Payload's `depth` parameter SHOULD request `depth=3` or higher.

### Media file fetch

```
GET /api/media/{id}/file
```

Returns the binary file (video, image, audio).

### Kiosk command

```
POST /api/kiosk-command
Body: { "kioskId": "<id>", "action": "<action>", "value": <optional> }
```

The CMS maps the action to the appropriate MQTT publish per the topic table above.

**Content actions** (published on `commands/playback`, `commands/volume`, `commands/loop`, or `commands/app` as appropriate): `play`, `pause`, `stop`, `next`, `prev`, `home`, `seek`, `loop`, `content`, `volume`, `sync`, `mode`.

**Lifecycle actions** (handled by the Supervisor on `commands/app` / `commands/power` / `wol`): `power_on`, `power_off`, `reboot`, `app_start`, `app_stop`, `app_restart`.

> **`restart` vs `app_restart`.** The reference CMS accepts both. They are *not* interchangeable:
> - `app_restart` publishes the bare string `"restart"` on `commands/app`. The Supervisor receives it, clears the restart counter, and restarts the Player process. This is the documented "restart the kiosk software" path.
> - `restart` publishes `{"action":"restart"}` JSON on `commands/app`. The Supervisor ignores it (wrong shape); the Player receives it and calls `window.location.reload()` — a *renderer reload* without a process restart.
>
> Use `app_restart` for kiosk-software recovery. The plain `restart` action exists for development scenarios (force the renderer to re-fetch its bundle) and SHOULD NOT be wired into operator-facing UI.

### IoT command

```
POST /api/iot-command
Body: { "deviceId": "<id>", "action": "<action>", "value": <optional> }
```

The CMS resolves the device's museum and publishes on `museum/{museumId}/iot/{deviceId}/command`.

### Live status reads

```
GET /api/kiosk-status?slug={slug}      # one kiosk, or all if slug omitted
GET /api/iot-status?deviceId={id}      # one device, or all if deviceId omitted
```

These return the CMS's MQTT-cached view of liveness, applying the staleness windows above. The returned shape for a single-kiosk query (`?slug=…`) MUST include at least:

```json
{
  "online": true,            // player heartbeat fresh
  "kioskOnline": true,       // supervisor heartbeat fresh
  "gracefulOffline": false,  // most recent supervisor heartbeat had graceful: true
  "playerStatus": "running",
  "mode": "loop",
  "playback": "playing",
  "volume": 70,
  "version": "1.0.0",
  "error": null,
  "lastHeartbeat": "2026-05-25T10:30:00Z"
}
```

`online` reflects only Player-heartbeat freshness. Consumers wishing to render "intentional power-off" differently from "fault" MUST inspect `gracefulOffline` separately — see *Two-tier liveness*.

The list-form response (`?slug` omitted) MAY return a reduced shape per kiosk (the reference CMS omits `volume` from the list form). Single-kiosk queries are authoritative for the full shape.

The shape for an IoT device MUST include at least:

```json
{
  "deviceId": "<id>",
  "state": "on",
  "online": true,
  "lastUpdate": "2026-05-25T10:30:00Z"
}
```

plus whichever of `brightness`, `color`, `scene`, `temperature`, `curStep` match the device's capabilities (see *Device capability model*).

---

## Fleet orchestration (optional capability)

A CMS MAY expose **orchestrations** — operator-initiated, fleet-scoped sequences that fan out commands to a set of kiosks and/or IoT devices. Orchestrations add grouping, ordering, and an operator trigger on top of the command bus; they introduce no new wire messages. **Power scenarios** (below) are the one orchestration this standard specifies.

Other operator-initiated orchestrations are implementation-private and composed entirely from the documented commands — no new primitive is required. For example, a guide gesture that idles every kiosk in a hall and then resumes them one by one is a fan-out of `screensaver`/`pause` followed by per-kiosk `play` or `content` to resume; a kiosk idled this way is indistinguishable on the wire from one idled by any other means, so the grouping lives only in the orchestrating CMS. The standard deliberately does not spec such features — it guarantees the commands they need exist.

This capability is OPTIONAL; only power scenarios are implemented in the reference CMS.

### Power scenarios

For museum/hall-wide power on/off, the CMS MAY fan out per-device commands in a sensible order with progress tracking.

A conforming implementation:

- Allows at most one scenario active globally at a time.
- Exposes start, query-active, and query-by-id endpoints (REST shape below).
- Reports progress as a sequence of step records (`stepLog`).
- Distinguishes terminal states: `completed`, `completed_with_errors`, `failed`, `timeout`.

### Endpoints

```
POST /api/power-scenario
Body: {
  "scope": "museum" | "hall",
  "scopeId": "<id>",
  "action": "on" | "off",
  "filters": ["light", "conditioner", "kiosk"]   // optional
}
Response 202: { "scenarioId": "<id>" }
Response 409: { "error": "...", "activeScenarioId": "<id>" }

GET /api/power-scenario
Response: { "active": <scenario or null> }

GET /api/power-scenario/{id}
Response: <scenario with stepLog>
```

### Step ordering

The reference implementation runs steps in this order for both `on` and `off`:

1. Kiosks (Wake-on-LAN for `on`; `commands/power "off"` for `off`).
2. Conditioners.
3. Lights and ambient devices.

The rationale: kiosks have the longest delay between command issuance and visible effect (PC boot for `on`, Windows shutdown for `off`), so firing them first overlaps their settling time with the rest of the work. Lights are cheap and instantaneous; running them last means the operator sees the orchestration progress with the lights still on (`off`) or comes on at the end of the sequence (`on`) — useful as a visible "scenario done" signal.

Implementations MAY reorder if museum operations call for it (e.g. "all lights off first to dim the hall, then kiosks") — the standard does not mandate. If reordered, the implementation MUST document the chosen order.

### Retry, timeout, and skip rules

- Per-IoT-step: up to 3 attempts with a configurable delay (default 500 ms) between attempts.
- Skip-if-already-target: if the cached state already matches the target, the step is recorded as `skipped` and no command is published.
- Skip-if-blocked: an `on` command on a conditioner whose configured "winter mode" is active MUST be skipped with reason `winter mode`.
- Kiosks are fire-and-forget at the orchestration level — WoL or `commands/power` are published once; subsequent state is observed via the supervisor heartbeat.
- Overall scenario timeout: `max(expected * 1.2 + 10s, 30s)` where `expected` is the sum of per-step worst-case timings.

### Scenario state machine

```
pending → running → completed
                  → completed_with_errors
                  → failed
                  → timeout
```

The scenario row MUST persist enough state to be polled after the orchestrator process crashes (e.g. in a Postgres table or equivalent). Step records SHOULD include `result`, `attempts`, `stateBefore`, `stateAfter`, and an optional `reason` for skipped or failed steps.

---

## Content package format

### Addressable content node

Any entity in a content package that carries a stable `id` unique within the package — a `MenuItem` or a `ShowcaseItem` — is an **addressable node**. The protocol references nodes by `id` **opaquely**: it does not distinguish a "section" from an "object". That distinction is purely the node's depth/role in the author's tree — a branch (a `submenu` MenuItem, a showcase grid) reads as a "section", a leaf reads as an "object".

These ids are the single binding/navigation key across the system:
- the `content` command (*Playback control*) opens any addressable node by `id`;
- the kiosk reports its current node and ancestor chain in `status.navigation`;
- event→action bindings (*Event→action bindings*) are keyed on these ids.

Whether a visitor-facing exhibit is modelled as a `MenuItem` leaf (with its own detail page) or a `ShowcaseItem` tile is an authoring choice and does not change the wire. The reference Player resolves `content` ids against the `MenuItem` tree (including `submenuItems`); resolving `ShowcaseItem` ids is permitted but not yet implemented.

### Content package

```typescript
interface ContentPackage {
  id: string
  name: string
  version: string

  // For Browse mode
  menuItems?: MenuItem[]

  // For Loop mode (all configurations)
  playlist?: {
    items: MediaItem[]
    loopPlaylist: boolean
  }

  // For Showcase profile
  showcaseItems?: ShowcaseItem[]

  // Guide-only content (hidden from visitors)
  guideContent?: {
    items: MediaItem[]
  }

  // Screensaver configuration
  screensaver?: {
    type: 'video' | 'image' | 'carousel' | 'animation'
    media?: MediaItem[]
    title?: string
    subtitle?: string
    showStartButton?: boolean
    startButtonText?: string
  }
}
```

### Menu item (Browse mode)

```typescript
interface MenuItem {
  id: string
  title: string
  description?: string
  thumbnail?: MediaItem
  contentType: 'video' | 'article' | 'showcase' | 'submenu'

  video?: MediaItem
  article?: Article
  showcaseItems?: ShowcaseItem[]
  submenuItems?: MenuItem[]

  guideOnly?: boolean
}
```

### Media item

```typescript
interface MediaItem {
  id: string
  url: string
  title?: string
  mimeType: string
  durationSeconds?: number
  thumbnail?: string
  guideOnly?: boolean
}
```

### Showcase item

```typescript
interface ShowcaseItem {
  id: string
  title?: string
  description?: string
  image: MediaItem
}
```

### Article

```typescript
interface Article {
  id: string
  title: string
  content: any        // Rich text (implementation-specific format)
  coverImage?: MediaItem
}
```

### Supported formats

| Type | Formats |
|---|---|
| Video | MP4 (H.264/H.265) |
| Audio | MP3 |
| Images | PNG, JPEG |

---

## Kiosk configuration

### Settings file

The Player and Supervisor each read a JSON settings file on startup. The fields used by every conforming implementation:

```json
{
  "kioskSlug": "kiosk-main-hall",
  "serverUrl": "http://cms.museum.local:40000",
  "mqttUrl": "ws://cms.museum.local:49001",
  "mqttUsername": "kiosk",
  "mqttPassword": "...",
  "mode": "browse",
  "display": {
    "fullscreen": true,
    "cursor": false
  }
}
```

- The MQTT URL MAY be `mqtt://...` for native TCP or `ws://` / `wss://` for WebSocket transport.
- `mode` is the boot-time default; it may be overridden at runtime by an MQTT `commands/app` message.
- The Supervisor MAY consume a separate settings file containing additional watchdog tuning (restart counts, grace periods, lock-file path). See *Supervisor* below.

### Local storage

Implementations MUST cache content locally. The specific storage mechanism is not prescribed (IndexedDB, filesystem, SQLite, etc.).

**Required capabilities:**
- Store complete content packages.
- Store media files for offline playback.
- Track sync state (last sync timestamp, package version).
- Serve cached media for playback without server connectivity.

**Recovery from cache corruption.** Implementations using a database engine that can wedge (e.g. IndexedDB) SHOULD provide an automated recovery path. The reference implementation:
1. Renderer catches the open failure and asks the main process to wipe the database directory.
2. Main process deletes the storage directory and re-opens.
3. After a fixed maximum number of attempts (reference implementation: 2), the Player publishes `INDEXEDDB_OPEN_FAILED_PERMANENT` on its `status` topic and surfaces the error to operators.

---

## Supervisor (control plane)

The Supervisor is a separate process running on the same machine as the Player, responsible for lifecycle, OS power commands, and hardware-tier reporting.

### Responsibilities

- PC power control (power on via WoL is delegated to a separate WoL relay; the Supervisor handles shutdown and reboot).
- Player lifecycle management (start, stop, restart).
- Watchdog (automatic Player recovery on crash or heartbeat timeout).
- System health reporting (CPU, RAM, network connectivity).

The Supervisor observes the Player through three channels:
1. **OS process monitoring** — is the Player process alive?
2. **MQTT heartbeat subscription** — is the Player responsive?
3. **Filesystem lock file** — is the Player mid-update?

### Watchdog

The Supervisor implements tiered recovery when the Player is detected as crashed or frozen:

| Tier | Behaviour |
|---|---|
| 1 | Graceful stop (if process alive), then start. Wait 30 s. Check if running and heartbeat fresh. |
| 2 | Force-kill the process tree, start. Wait 30 s. Check again. |
| Circuit breaker | After N failed restarts within a healthy window (reference implementation: 3 within 300 s), set `player.status` to `unresponsive` and stop attempting recovery. Reset after a cooldown (reference implementation: 30 minutes). |

**Configuration parameters** (reference implementation):

| Parameter | Config path | Default | Meaning |
|---|---|---|---|
| `maxRestarts` | `watchdog.maxRestarts` | 3 | Circuit-breaker threshold. |
| `healthyResetTime` | `watchdog.healthyResetTime` | 300 s | Window after which a healthy Player clears the restart counter. |
| `updateGracePeriod` | `watchdog.updateGracePeriod` | 180 s | How long the Supervisor pauses recovery when the update lock file is present. |
| `startupGracePeriod` | `watchdog.startupGracePeriod` | 60 s | How long after Player start the Supervisor delays first health-check action. |
| `heartbeatTimeout` | `player.heartbeatTimeout` | 30 s | How long a missing Player heartbeat is tolerated. (Note: lives under `player.*` in the reference config because it describes the Player's reporting cadence, not the watchdog's reaction tuning.) |

### Offline-mode opt-out

If the Supervisor has never observed a Player heartbeat in the current session (e.g. cold boot with no network), it MUST NOT restart the Player on heartbeat-timeout. The Player is presumed to be running in offline mode and is rendering cached content; killing it would interrupt visitor experience for no gain.

### Player update coordination

The Player uses its own update mechanism. The Supervisor does not participate in updates — it only pauses the watchdog during the update window:

1. The Player or its installer writes a lock file at a known path before starting the update.
2. The Supervisor detects the lock file and pauses the watchdog for up to `updateGracePeriod`.
3. The Player completes update, restarts, and the lock file is removed.
4. The Supervisor resumes normal monitoring.

If the lock file persists beyond the grace period, the Supervisor SHOULD remove the lock and resume the standard recovery flow.

### Power commands

On receiving `"off"`, `"shutdown"`, or `"reboot"` via `commands/power`, the Supervisor MUST:
1. Publish the graceful-offline `system/heartbeat` payload (retained, QoS 1).
2. Stop the Player gracefully (the Supervisor MAY ask the Player to quit via `commands/app {"action":"quit"}` on platforms where SIGTERM is a hard kill, then force-kill on timeout).
3. Disconnect from the MQTT broker cleanly.
4. Execute the OS shutdown/reboot.

### Wake-on-LAN

Wake-on-LAN is published by the CMS to `umka/kiosks/{slug}/wol` with the MAC address as a bare string. A separate WoL relay service on the museum LAN consumes this and emits the magic packet. The Supervisor itself is not running when the kiosk is off; the relay decouples WoL from the kiosk machine.

> The simplified reference implementation MAY omit the Supervisor and handle power, app lifecycle, and watchdog inside the Player itself. This is acceptable for development and small deployments but not recommended for production — a hung Player with no separate Supervisor is unreachable until physical intervention.

---

## Security

**Minimum requirements:**
- MQTT over TLS (port 8883) or wss for production.
- REST API over HTTPS for any deployment crossing the museum LAN boundary.
- Per-user authentication on the CMS (Payload-style JWT in the reference implementation).

**Recommended:**
- MQTT client certificates or per-client username/password with ACLs.
- Isolated VLAN for the kiosk network.
- OS-level kiosk mode (auto-logon, disabled task manager, no desktop access).
- Disabled external USB/input devices.

**Acknowledged limitations of the reference implementation:**
- The reference broker accepts anonymous connections; the museum VPN boundary carries the security burden.
- The CMS has no MFA and no brute-force protection (`maxLoginAttempts: 0`); the deployment assumption is a trusted internal network.
- Kiosk MQTT credentials are stored in cleartext in the settings file on disk.

These are operational risks documented for awareness, not endorsements.

---

## Compliance checklist

An Umka-compatible Player MUST:

- [ ] Implement at least one standard mode (Loop, Browse, or Custom).
- [ ] Follow the MQTT topic structure and message formats.
- [ ] Publish `status` on state changes (retained).
- [ ] Publish `heartbeat` every 10 seconds.
- [ ] Operate from locally stored content (local-first).
- [ ] Synchronise content via REST API on boot (best-effort) and on `commands/app { "action": "sync" }`.
- [ ] Filter guide-only content from visitor displays.
- [ ] Auto-reconnect to MQTT on connection loss.
- [ ] Publish `triggerEnded: true` once on completion of trigger-initiated playback.

An Umka-compatible Supervisor MUST:

- [ ] Subscribe to `commands/power` (bare-string `off`/`shutdown`/`reboot`).
- [ ] Subscribe to `commands/app` (bare-string `start`/`stop`/`restart`).
- [ ] Publish `system/heartbeat` every 10 seconds, retained, with the documented payload.
- [ ] Register an LWT on `system/heartbeat` for unexpected-drop detection.
- [ ] Publish a graceful-offline payload (`graceful: true`) on clean shutdown.
- [ ] Publish `system/crash` on each recovery attempt.
- [ ] Apply offline-mode opt-out when the Player has never had a real heartbeat in the current session.
- [ ] Honour the update lock file for at least `updateGracePeriod`.

An Umka-compatible CMS MUST:

- [ ] Expose `POST /api/kiosk-command` and `POST /api/iot-command` with the documented action sets.
- [ ] Expose `GET /api/kiosk-status` and `GET /api/iot-status` returning the documented shapes with the documented staleness windows applied.
- [ ] Apply the documented staleness windows (30 s for kiosk/supervisor, 3 s for IoT).
- [ ] Drive the trigger pipeline including fallback resets.

**Recommended:**
- [ ] Support all three operating modes.
- [ ] Support locale switching.
- [ ] Smooth transitions between content.
- [ ] Idle timeout with auto-reset and a visible warning before reset.
- [ ] IoT trigger integration with fallback timers.
- [ ] Cache eviction for orphaned media.
- [ ] Report `navigation` and `screensaverActive` on `status` in Browse mode.
- [ ] Event→action bindings keyed on content-package node ids (app-state → kiosk/IoT).
- [ ] Fleet orchestration with progress tracking (power scenarios).

---

## What this standard does NOT define

- **UI/UX design** — Visual design, animations, and layout are per-client.
- **Storage mechanism** — IndexedDB, filesystem, SQLite — implementor's choice.
- **Video codec support** — Beyond the required formats, implementations may support additional codecs.
- **Game logic** — Custom Mode applications define their own behaviour.
- **Hardware-specific features** — Keystone correction, multi-monitor, etc.
- **Update mechanism** — How the kiosk software is updated.
- **Supervisor implementation** — The standard defines the MQTT contract for system control; the supervisor implementation (language, packaging, service manager) is not prescribed.
- **Fieldbus protocols** — How IoT devices talk to the bridge (Modbus, KNX, etc.) is out of scope; only the MQTT side is normative.
- **Auth mechanism** — JWT, OAuth, mTLS — the standard only requires that command endpoints are authenticated.

---

## Version history

| Version | Date | Changes |
|---|---|---|
| 1.26.0 | Jan 2026 | Initial standard: Loop, Browse modes, MQTT protocol. |
| 1.26.1 | Feb 2026 | Custom mode, guide-only content, IoT triggers, local-first architecture, CMS-agnostic API. |
| 1.26.2 | Feb 2026 | Control plane separation, hardware-agnostic clarifications, multi-level Browse navigation, power-on command. |
| 1.26.3 | Feb 2026 | Aligned with Sentinel reference implementation: updated watchdog timings (5 s Tier 1, 30-min circuit-breaker cooldown), added crash diagnostics to service heartbeat, removed disk metrics from heartbeat schemas. |
| 1.26.3.2 | Mar 2026 | Aligned status payload with implementation: added mandatory `version`, `uptime`, `error`; expanded mode values; added IoT device topics; documented reads-direct/writes-through-CMS architecture; corrected system-command payload format; added `temperature` IoT action; defined `KioskError` and standard error codes. |
| 1.26.5.0 | May 2026 | **Breaking topic-namespace correction**: system-level commands moved from `system/power`/`system/app` (JSON `{command}` payload) to `commands/power`/`commands/app` (bare-string payload); `system/*` is now reserved exclusively for supervisor publishes. Added: `system/crash` topic (with `restartAttempt`/`maxRestarts` fields); `wol` topic for Wake-on-LAN; LWT + graceful-offline marker; `triggerEnded` field on kiosk status; full trigger pipeline (with `trigger_play` envelope and fallback timers); `INDEXEDDB_OPEN_FAILED_PERMANENT` error code (other codes reserved but not yet emitted); IoT REST actions `reset`/`step` (collapsing to wire `activation`) and `power_on`/`power_off` aliases; explicit two-tier liveness (`on`/`sw-off`/`error`/`off` derivation); pinned heartbeat cadences (10 s for kiosk and supervisor; 2 s default for IoT) and staleness windows (30 s kiosk/supervisor; 3 s IoT); IoT status `state` string and derivation precedence; clarified `restart` vs `app_restart` (renderer reload vs supervisor restart); IoT status field `cur_step` documented as snake_case on the wire. Optional capability: power-scenario orchestration (REST contract + state machine + ordering rules — reference order is kiosks → conditioners → lights for both `on` and `off`). Retired: `POST /api/kiosks/{slug}/sync-complete` (never implemented); `system/power`/`system/app` topic prescriptions; `diskPercent` field from supervisor heartbeat schema (note: reference Sentinel still emits `0` as of this version; consumers MUST ignore). Clarified: REST API URL shapes are advisory; settings-file shape; cache-eviction SHOULD; offline-mode supervisor opt-out as a MUST. Known implementation gaps documented in `docs/plans/2026-05-26-standard-impl-gaps.md`. |
| 1.26.5.1 | May 2026 | **Non-breaking additions, aligned with the as-built implementation.** Mode enum narrowed to the three base modes (`loop`/`browse`/`custom`); `showcase`/`projector`/`audio` removed as wire `mode` values (they were never in the CMS schema — Showcase is now a Browse content pattern via `showcaseItems`; Projector/Audio are configuration profiles); `game` documented as a Custom specialisation string (the reference CMS keeps emitting it; consumers treat unrecognised values as `custom`). Added: *Addressable content node* concept (MenuItem/ShowcaseItem referenced opaquely by `id`); `navigation` (`nodeId`/`path`/`showcaseOpen`) and `screensaverActive` on the retained `status`; `screensaver` playback command; broadened `content` command to any addressable node. New optional capability: *Event→action bindings* (kiosk state → kiosk/IoT commands, keyed on content-package ids, state-coupled via `navigation.path`) — generalises the trigger pipeline; reference CMS wires only the physical-trigger→media case so far. Renamed "Power-scenario orchestration" → *Fleet orchestration* to frame it as the general operator-initiated fan-out pattern; power scenarios remain the only specified instance, and the section now notes that custom orchestrations (e.g. a hall freeze) are composed from existing commands rather than spec'd as features. Reframed IoT devices as a *capability model* (on/off, dimming, colour, temperature, scenario stepping) rather than an enumerated type list — device "type" is now explicitly descriptive, not a wire concept, and devices that don't map onto the standard capabilities are out of scope (integrated privately, like fieldbus protocols), so adding a new device needs no spec change. All additions are OPTIONAL/Recommended; a v1.26.5.0-conformant Player, Supervisor, and CMS remain conformant. |

---

## License

This specification is an **open standard**. Any implementation following this specification is considered Umka-compatible.

Reference implementations are released under the MIT License.

Implementations of this standard will naturally share protocol-level code because they implement the same specification. This is by design.

# Umka Kiosk Standard v1.26.2

**Status:** Draft
**Date:** February 2026
**License:** Open Standard (Implementation: MIT)

---

## 1. Overview

The Umka Kiosk Standard defines a universal protocol for museum multimedia kiosk systems. This standard covers:

- Operating modes and their configurations
- Real-time control protocol (MQTT)
- Content synchronization (REST API)
- Local-first content architecture
- Integration with guide tablets and IoT devices

### 1.1 Design Principles

1. **Local-first** — Kiosks always operate from locally stored content. Server connectivity is used for synchronization and remote control, not for primary operation
2. **CMS-agnostic** — The standard defines the API contract between kiosk and server. The server-side CMS implementation is not prescribed
3. **Hardware-agnostic** — The player runs on a Windows PC and supports various output devices: monitors, TVs, touchscreens, LED panels, projectors, and audio systems
4. **Standards-based** — Any implementation following this spec is Umka-compatible. Implementations will share protocol code by design

### 1.2 Terminology

| Term | Definition |
|------|-----------|
| **Umka** | Museum kiosk management system: server (CMS), admin interface, and guide tablet app |
| **Kiosk Software** | Universal multimedia software for kiosks (Umka Player), built on Electron |
| **Loop Mode** | Kiosk operating mode with automatic cyclic playback of a media playlist (video player) |
| **Browse Mode** | Interactive kiosk operating mode with a menu/catalog for visitor self-service content selection (showcase) |
| **Custom Mode** | Kiosk operating mode with non-standardized functionality (e.g., interactive game) |
| **Screensaver** | Visual representation of the idle state |
| **Content Package** | Collection of content (media files, settings) assigned to a specific kiosk |
| **IoT Device** | Physical device (button, sensor) connected to the system via Umka API |
| **Physical Trigger** | Hardware button that sends an event to the system to launch content on a kiosk |
| **Guide Tablet** | Tablet application for museum guides to control kiosk playback during tours |

---

## 2. Operating Modes

Umka defines three standard operating modes. Each mode can be configured for different hardware setups and use cases.

### 2.1 Loop Mode

Automatic cyclic playback of a media playlist (video, images, audio). Playlist order and image display duration are configured in the CMS.

**Common behavior:**
- Plays content from an ordered playlist
- Loops back to the beginning after last item
- Remote control via guide tablet (MQTT)
- Content synchronized from server, played from local storage

#### Configuration: Continuous (No Touch)

**Use case:** LED panels, video walls, non-interactive displays

```
Playlist[0] → Playlist[1] → ... → Playlist[n] → (loop)
```

- Automatic playback in infinite loop
- Video, images, and audio in playlist
- Automatic transitions between playlist items
- No user interaction on device

#### Configuration: Interactive (Touch + Screensaver)

**Use case:** Touchscreen kiosks with video playback

```
Screensaver → (touch "Start") → Playlist → (end/idle) → Screensaver
```

- Screensaver displayed in idle state
- "Start" button on screensaver to begin playback
- "Back" button during playback to return to screensaver
- Return to screensaver after playback completes
- Video controls: play/pause, seek, volume
- Controls appear on touch, auto-hide after inactivity timeout
- Configurable idle timeout to return to screensaver

#### Configuration: Triggered (IoT Button)

**Use case:** Exhibits with physical buttons near displays

```
Screensaver (no "Start" button) → (IoT signal) → Playlist → (end) → Screensaver
```

- Waits for signal from physical button (IoT device)
- Launches playback on trigger event received via MQTT
- Returns to idle state after playback completes
- No touch interaction required on kiosk

#### Configuration: Audio-Only

**Use case:** Background music in museum halls

```
Idle (silence) → (command) → Audio Playlist → (loop)
```

- No video output (display not required)
- Plays audio files (MP3, WAV) through speakers
- Each hall is a separate "kiosk" in audio configuration
- Control: volume, track selection, play/pause via guide tablet

#### Configuration: Projector (Passive Display)

**Use case:** Short-throw projectors, projection mapping

```
Black screen → (trigger/command) → Playlist → (end) → Black screen
```

- Completely passive (no on-screen UI controls)
- Triggered by MQTT command, guide tablet, or IoT device
- Can play once and return to black, or loop
- Can display a static image instead of black screen in idle state
- No touch interaction

---

### 2.2 Browse Mode

Interactive mode with a menu/catalog for visitor self-service content selection.

**Common behavior:**
- Hierarchical menu navigation with unlimited nesting depth
- Multiple content types (video, articles, image galleries)
- Touch-optimized interface
- Idle timeout returns to screensaver
- Remote control via guide tablet (MQTT)

#### Content Showcase (Catalog)

- Display objects as a grid of cards with images and titles
- Scrollable showcase (vertical or horizontal depending on screen orientation)
- Tap card to view detailed content
- Grid adapts to screen orientation and resolution

#### Detailed Object View

- Object title and description
- Media list: photos, videos
- Navigation through media items (scroll, swipe, or carousel — implementation-specific)
- Caption and description for each media item (when available)
- Full-screen media viewing
- Video playback with controls

#### Screensaver System

- Eye-catching content displayed during idle state
- Carousel of images and/or video with automatic rotation
- Configurable title and subtitle text
- Optional "Start" button with configurable text
- Configurable idle timeout duration

---

### 2.3 Custom Mode

Non-standardized kiosk functionality. Custom Mode applications implement their own UI and logic but integrate with the Umka system for monitoring and power management.

**Use case:** Interactive games, custom interactive experiences

**Required Umka integration:**
- Heartbeat publishing (every 10 seconds)
- Power management commands (shutdown, reboot)
- Status reporting (online/offline)

**Optional integration:**
- Content synchronization
- Guide tablet control
- Idle timeout and auto-reset

**Game-specific requirements (when applicable):**
- Touch-optimized interface
- Return to initial state on inactivity (configurable timeout)
- Game mechanics and scenario defined per implementation

---

## 3. Functional Requirements

### 3.1 General (All Modes)

- Display of museum logo (configured in CMS)
- Configurable inactivity timeout before returning to idle state
- Automatic reset to default state on user inactivity
- Smooth transitions between playlist items (Loop mode)
- Adaptive layout for different screen resolutions and orientations

### 3.2 Guide Mode

The kiosk in any mode (Loop, Browse) supports remote control from a guide tablet via MQTT. Guide controls are NOT displayed on the kiosk itself.

**Guide capabilities:**
- Select and launch media content on kiosk
- Access content from "Guide Folder" (hidden from visitors)
- Playback control: pause, play, seek
- Toggle looped playback
- Volume control
- Exit guide mode (return kiosk to default content)

### 3.3 Guide-Only Content (Папка экскурсовода)

Content marked as guide-only:
- **Hidden** from visitor-facing displays in all modes
- **Accessible** only via guide tablet control
- **Filtered** automatically from playlists and menus
- **Stored** locally alongside regular content for offline guide access

### 3.4 Idle Timeout and Auto-Reset

1. Software tracks user activity (screen touches), including when video is paused
2. On reaching the inactivity timeout, a warning appears with countdown (e.g., "Are you still here?")
3. Warning includes a "Stay" button — pressing it dismisses warning and resets timer
4. If user does not respond before countdown expires — kiosk returns to default state
5. On reset: volume and video position revert to defaults

Inactivity timeout and countdown duration are configurable per kiosk.

### 3.5 IoT Device Integration

**Physical Triggers (IoT Buttons):**
- Integration via Umka messaging protocol (MQTT)
- Receive trigger events from IoT devices
- Launch configured content on the associated kiosk
- Trigger-to-kiosk and trigger-to-content mapping configured in CMS

### 3.6 Multi-Language Support

- Dynamic locale switching via MQTT command
- Localized content delivery from CMS
- Persistent locale preference per kiosk

**Standard locales:** `ru` (Russian), `en` (English). Additional locales can be added per museum.

---

## 4. Local-First Architecture

Umka kiosks follow a **local-first** architecture. The kiosk always operates from locally stored content. Server connectivity is used for synchronization, not for primary operation.

### 4.1 Principle

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

- Content is **always played from local storage**, regardless of server connectivity
- Sync runs in the background and does not interrupt playback
- Server going offline has no immediate effect on kiosk operation
- New content becomes available only after successful sync + local cache

### 4.2 Synchronization

Sync is **trigger-based**, not polling. The kiosk downloads content only when instructed.

**Trigger:** CMS publishes an MQTT `sync` command to `umka/kiosks/{slug}/commands/app`. This happens automatically when a content package is saved (via `afterChange` hook) or when an admin clicks "Sync" in the control panel.

**What hits the server:**
1. Package metadata — always fetched from CMS REST API (`GET /api/content-packages/{id}?depth=3`). This is the small JSON describing all media references. **Always a server call.**
2. Media files — each file is checked against the local cache:
   - If file exists locally (by media ID) AND checksum matches → **skip (no server call)**
   - If file is missing or checksum differs → **download from server**

**What stays local:**
- Playback always reads from local storage (`media-cache://` protocol in Electron)
- The player **never streams from the server** during normal operation
- If the server goes down mid-playback, nothing changes — content is already local

**Current limitation (TD-001):** Checksum-based diffing is planned but not yet implemented. Currently, files are matched by media ID only. If a file is replaced on the server (same ID, different content), the kiosk will not detect the change and will serve stale content until the cache is cleared. See `plans/tech-debt.md` TD-001.

**Sync behavior summary:**
- Background download does not interrupt current playback
- Switches to new content after all files are cached
- Failed downloads are skipped (other files still sync); errors logged

### 4.3 Connectivity Loss

When server connection is lost:

1. Content continues playing from local storage (no interruption)
2. Heartbeat stops — server marks kiosk as offline
3. MQTT remote control unavailable until reconnection
4. Auto-reconnect attempts continue in the background
5. On reconnection: resume heartbeat, check for content updates

**Note:** Guide commands and IoT triggers require server connectivity and are unavailable during offline periods.

---

## 5. MQTT Protocol

### 5.1 Topic Structure

**Format:** `umka/kiosks/{kioskSlug}/{category}/{type}`

Where `{kioskSlug}` is the unique text identifier for each kiosk.

**Categories:**
- `commands/*` — Server/Guide → Player (content and playback control)
- `system/*` — Server → Service (power and app lifecycle control)
- `status` — Player → Server (state reporting)
- `heartbeat` — Player → Server (app health monitoring)
- `system/heartbeat` — Service → Server (system health monitoring)

### 5.2 System Topics (Server → Service)

In production deployments, power and app lifecycle commands are handled by a separate control plane service that runs independently of the player. This ensures the kiosk remains remotely manageable even if the player application hangs or crashes.

#### Power Management
**Topic:** `umka/kiosks/{kioskSlug}/system/power`

| Payload | Description |
|---------|-------------|
| `"on"` | Power on via Wake-on-LAN (magic packet) |
| `"shutdown"` | Shutdown OS |
| `"reboot"` | Reboot OS |

#### Application Lifecycle
**Topic:** `umka/kiosks/{kioskSlug}/system/app`

| Payload | Description |
|---------|-------------|
| `"start"` | Launch the player application |
| `"stop"` | Stop the player application |
| `"restart"` | Stop and relaunch the player application |

#### Service Heartbeat
**Topic:** `umka/kiosks/{kioskSlug}/system/heartbeat`
**QoS:** 0
**Retain:** true
**Interval:** Every 10 seconds

**Payload:**
```json
{
  "kioskId": "kiosk-1-1",
  "timestamp": "2026-02-08T12:00:00Z",
  "version": "1.0.0",
  "uptime": 3600,
  "player": {
    "status": "running",
    "pid": 1234,
    "lastHeartbeat": "2026-02-08T12:00:00Z",
    "restartCount": 0
  },
  "system": {
    "cpuPercent": 45,
    "memoryPercent": 62,
    "diskPercent": 78,
    "networkConnected": true
  }
}
```

**Player status values:** `running`, `stopped`, `updating`, `unresponsive`, `restarting`

> **Note:** The simplified MIT reference implementation does not include the control plane service. Power and app commands are handled within the player itself. Production deployments SHOULD use a separate service for reliability.

---

### 5.3 Content Command Topics (Server/Guide → Player)

#### Content Sync
**Topic:** `umka/kiosks/{kioskSlug}/commands/app`

| Payload | Description |
|---------|-------------|
| `{ "action": "sync" }` | Trigger content resync from CMS |
| `{ "action": "mode", "value": "loop" }` | Change operating mode |

**Valid modes:** `loop`, `browse`, `custom`

---

#### Playback Control
**Topic:** `umka/kiosks/{kioskSlug}/commands/playback`

| Payload | Description |
|---------|-------------|
| `{ "action": "play" }` | Resume playback |
| `{ "action": "play", "mediaId": "uuid" }` | Play specific media by ID |
| `{ "action": "play", "contentId": "uuid" }` | Play specific menu item by ID |
| `{ "action": "pause" }` | Pause current playback |
| `{ "action": "stop" }` | Stop and return to idle/screensaver |
| `{ "action": "next" }` | Next item in playlist/gallery |
| `{ "action": "prev" }` | Previous item in playlist/gallery |
| `{ "action": "home" }` | Return to main menu (Browse mode) |

**Media resolution order for `mediaId`:**
1. Playlist items
2. Guide-only content
3. Menu item video attachments

---

#### Volume Control
**Topic:** `umka/kiosks/{kioskSlug}/commands/volume`

**Payload:** Integer 0–100

```json
75
```

---

#### Locale Control
**Topic:** `umka/kiosks/{kioskSlug}/commands/locale`

**Payload:** ISO 639-1 locale code

```json
"ru"
```

---

#### Loop Control
**Topic:** `umka/kiosks/{kioskSlug}/commands/loop`

**Payload:** Boolean

```json
true
```

Toggles whether current playback loops or plays once.

---

#### IoT Trigger
**Topic:** `umka/kiosks/{kioskSlug}/commands/trigger`

**Payload:**
```json
{ "deviceId": "button-uuid", "contentId": "media-uuid" }
```

Received when an IoT device (physical button) triggers content on this kiosk.

---

### 5.4 Status Topics (Player → Server)

#### Kiosk Status
**Topic:** `umka/kiosks/{kioskSlug}/status`
**QoS:** 0
**Retain:** true

**Payload:**
```json
{
  "kioskId": "uuid",
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
  "timestamp": "2026-02-06T10:30:00Z"
}
```

**Fields:**

| Field | Type | Values |
|-------|------|--------|
| `state` | string | `idle`, `playing`, `paused`, `loading`, `error` |
| `mode` | string | `loop`, `browse`, `custom` |
| `volume` | integer | 0–100 |
| `locale` | string | ISO 639-1 code |
| `currentContent` | object or null | Current playback info |
| `currentContent.type` | string | `video`, `article`, `showcase` |
| `currentContent.position` | number (optional) | Playback position in seconds |
| `currentContent.duration` | number (optional) | Total duration in seconds |

**Published on:** playback state change, content switch, volume change, mode change, locale change.

---

#### Heartbeat
**Topic:** `umka/kiosks/{kioskSlug}/heartbeat`
**QoS:** 0
**Interval:** Every 10 seconds

**Payload:**
```json
{
  "kioskId": "uuid",
  "timestamp": "2026-02-06T10:30:00Z",
  "version": "1.0.0",
  "uptime": 3600,
  "diskFreeGB": 45.2
}
```

| Field | Type | Description |
|-------|------|-------------|
| `kioskId` | string | Kiosk UUID |
| `timestamp` | string | ISO 8601 |
| `version` | string | App version (semver) |
| `uptime` | integer | Seconds since app start |
| `diskFreeGB` | number (optional) | Free disk space |

**Server-side offline detection:**
- No heartbeat for >30 seconds → mark offline
- No heartbeat for >5 minutes → alert administrator

---

## 6. REST API

The standard defines the API contract between kiosk and server. Server-side implementation (CMS choice) is not prescribed by this standard.

**Base URL:** Configured per kiosk in `settings.json`.

### 6.1 Get Kiosk Configuration

```http
GET /api/kiosks/{kioskSlug}
```

**Response:**
```json
{
  "id": "kiosk-uuid",
  "slug": "kiosk-main-hall",
  "mode": "browse",
  "contentPackageId": "package-uuid",
  "locale": "ru",
  "settings": {
    "idleTimeoutSeconds": 120,
    "volume": 80
  }
}
```

### 6.2 Get Content Package

```http
GET /api/content-packages/{id}
```

**Response:** Complete content package (see Section 7 for format).

### 6.3 Get Media File

```http
GET /api/media/{id}/file
```

**Response:** Binary file (video, image, audio).

### 6.4 Confirm Sync

```http
POST /api/kiosks/{kioskSlug}/sync-complete
```

**Body:**
```json
{
  "packageId": "package-uuid",
  "timestamp": "2026-02-06T10:30:00Z"
}
```

---

## 7. Content Package Format

### 7.1 Content Package

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

  // Guide-only content (hidden from visitors)
  guideContent?: {
    items: MediaItem[]
  }

  // Screensaver configuration
  screensaver?: {
    type: 'video' | 'image' | 'carousel' | 'animation'
    media?: MediaItem[]         // For carousel: multiple items
    title?: string
    subtitle?: string
    showStartButton?: boolean
    startButtonText?: string
  }
}
```

### 7.2 Menu Item (Browse Mode)

```typescript
interface MenuItem {
  id: string
  title: string
  description?: string
  thumbnail?: MediaItem
  contentType: 'video' | 'article' | 'showcase' | 'submenu'

  // Content references (depending on contentType)
  video?: MediaItem
  article?: Article
  showcaseItems?: ShowcaseItem[]
  submenuItems?: MenuItem[]

  // Guide-only flag
  guideOnly?: boolean
}
```

### 7.3 Media Item

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

### 7.4 Showcase Item

```typescript
interface ShowcaseItem {
  id: string
  title?: string
  description?: string
  image: MediaItem
}
```

### 7.5 Article

```typescript
interface Article {
  id: string
  title: string
  content: any        // Rich text (implementation-specific format)
  coverImage?: MediaItem
}
```

### 7.6 Supported Formats

| Type | Formats |
|------|---------|
| Video | MP4 (H.264/H.265) |
| Audio | MP3 |
| Images | PNG, JPEG |

---

## 8. Kiosk Configuration

### 8.1 Settings File

```json
{
  "kioskId": "uuid-of-kiosk",
  "kioskSlug": "kiosk-main-hall",
  "serverUrl": "https://umka.museum.local",
  "mqttUrl": "mqtt://umka.museum.local:1883",
  "museumId": "museum-uuid",
  "mode": "loop",
  "network": {
    "macAddress": "00:11:22:33:44:55"
  },
  "display": {
    "fullscreen": true,
    "cursor": false
  }
}
```

### 8.2 Local Storage

Implementations MUST cache content locally. The specific storage mechanism is not prescribed (IndexedDB, filesystem, SQLite, etc.).

**Required capabilities:**
- Store complete content packages
- Store media files for offline playback
- Track sync state (last sync timestamp, package version)
- Serve cached media for playback without server connectivity

---

## 9. Power Management & Control Plane (Service)

### 9.1 Control Plane Separation

In production deployments, system-level control (power, app lifecycle, watchdog) SHOULD be handled by a separate control plane service running independently of the player. This ensures remote manageability even when the player is unresponsive.

**Service responsibilities:**
- PC power control (power on via WoL, shutdown, reboot)
- Player lifecycle management (start, stop, restart)
- Watchdog (automatic player recovery on crash or freeze)
- System health reporting (CPU, RAM, disk, network)

**Player responsibilities (content plane):**
- Content playback and navigation
- Content settings (volume, locale, loop)
- Content sync with CMS
- Player-level status and heartbeat

The service observes the player through three channels:
1. **OS process monitoring** — is the player process alive?
2. **MQTT heartbeat subscription** — is the player responsive?
3. **Filesystem lock file** — is the player mid-update?

> **Note:** The simplified MIT reference implementation does not include the control plane service. It handles power and lifecycle commands within the player itself. This is acceptable for development and small deployments but not recommended for production.

### 9.2 Wake-on-LAN (Power ON)

- BIOS: Wake-on-LAN enabled
- NIC: WoL support enabled
- Magic packet: UDP broadcast on port 9

### 9.3 Shutdown / Reboot

On receiving power command via `system/power` topic, the service MUST:
1. Publish updated status (player.status = "stopped")
2. Stop the player process gracefully
3. Execute OS shutdown/reboot command

### 9.4 Watchdog (Automatic Recovery)

The service implements three-tier recovery when the player is detected as crashed or frozen:

1. **Tier 1 — Graceful restart:** Launch the player, wait 30 seconds
2. **Tier 2 — Force restart:** Kill the player process, relaunch, wait 30 seconds
3. **Tier 3 — Circuit breaker:** After 3 failed restarts, stop attempting. Report `player.status = "unresponsive"` in heartbeat

Circuit breaker resets after 5 minutes of healthy player operation.

### 9.5 Player Update Coordination

The player uses its own update mechanism (e.g., electron-builder auto-update). The service does not participate in updates — it only pauses the watchdog during the update window.

1. Player writes `updating.lock` file before starting the update
2. The service detects lock file, pauses watchdog (grace period: 3 minutes)
3. Player completes update, restarts, deletes lock file
4. The service resumes normal monitoring

If the lock file persists beyond the grace period, the service assumes the update failed and proceeds with Tier 1 recovery.

---

## 10. Security

**Minimum Requirements:**
- MQTT over TLS (port 8883) for production
- REST API over HTTPS
- Kiosk authentication via UUID

**Recommended:**
- MQTT client certificates
- Isolated VLAN for kiosk network
- API key authentication
- OS-level kiosk mode (prevent user access to desktop)
- Disable external USB/input devices

---

## 11. Compliance Checklist

An Umka-compatible player MUST:

- [ ] Implement at least one standard mode (Loop, Browse, or Custom)
- [ ] Follow MQTT topic structure and message formats
- [ ] Publish status on state changes
- [ ] Publish heartbeat every 10 seconds
- [ ] Operate from locally stored content (local-first)
- [ ] Synchronize content via REST API
- [ ] Filter guide-only content from visitor displays
- [ ] Auto-reconnect MQTT on connection loss

**Production deployments SHOULD:**
- [ ] Use a separate control plane service for power and lifecycle control
- [ ] Implement watchdog with tiered recovery
- [ ] Report system health (CPU, RAM, disk) via service heartbeat
- [ ] Support update coordination via lock file

**Recommended:**
- [ ] Support all three operating modes
- [ ] Support locale switching
- [ ] Smooth transitions between content
- [ ] Idle timeout with auto-reset
- [ ] IoT trigger integration
- [ ] Touch gesture optimization

---

## 12. What This Standard Does NOT Define

The following are explicitly left to each implementation:

- **UI/UX design** — Visual design, animations, and layout are per-client
- **Storage mechanism** — IndexedDB, filesystem, SQLite — implementor's choice
- **Video codec support** — Beyond the required formats, implementations may support additional codecs
- **Game logic** — Custom Mode applications define their own behavior
- **Hardware-specific features** — Keystone correction, multi-monitor, etc.
- **Update mechanism** — How the kiosk software itself is updated
- **Service implementation** — The standard defines the MQTT contract for system control; the service implementation (language, packaging, service manager) is not prescribed

---

## 13. Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.26.0 | Jan 2026 | Initial standard: Loop, Browse modes, MQTT protocol |
| 1.26.1 | Feb 2026 | Custom mode, guide-only content, IoT triggers, local-first architecture, CMS-agnostic API |
| 1.26.2 | Feb 2026 | Control plane separation, hardware-agnostic clarifications, multi-level Browse navigation, power ON command |

---

## 14. License

This specification is an **open standard**. Any implementation following this specification is considered Umka-compatible.

Reference implementations are released under the MIT License.

Implementations of this standard will naturally share protocol-level code because they implement the same specification. This is by design.

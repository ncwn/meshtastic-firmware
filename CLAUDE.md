# meshtastic-firmware (v4 fork)

> This is a fork of `meshtastic/firmware` on the `v4` branch.
> When merging upstream releases, consult the V4 Modifications section
> to understand which conflicts are expected vs accidental.

## Upstream Base

- **Tag:** v2.7.22.96dd647
- **Commit:** 01bd4cfb73bb7bc20ea4cf08c36d66c45b4bea35
- **Channel:** prerelease (alpha)
- **Upstream repo:** meshtastic/firmware
- **Fork repo:** ncwn/meshtastic-firmware

## Build

- PlatformIO-based — see `platformio.ini` for targets
- Build: `pio run -e <target>`
- Flash: `pio run -e <target> -t upload`
- Monitor: `pio device monitor`

## Rules

- Always merge upstream, **never rebase v4**
- Update the V4 Modifications section below when changing files
- Do NOT modify protobufs without checking the Dependencies section
- Feature work goes on branches off v4, merged back to v4
- After pushing v4, update the wrapper repo submodule SHA

## V4 Modifications

<!-- When you modify a file, add an entry here:

### path/to/file.cpp
- **What:** Brief description of the change
- **Why:** Reason this modification is needed for the v4 project
- **Conflict risk:** Low / Medium / High when merging upstream
-->

### .gitignore
- **What:** Added ignores for the SELFCIUS generated PlatformIO overlay symlinks: `/platformio_override.ini`, `/variants/selfcius/`, and `/src/selfcius/`
- **Why:** The wrapper repo creates these symlinks from `build/setup.sh` so SELFCIUS build environments and sources can load without committing wrapper-local paths into the public firmware fork
- **Conflict risk:** Low - append-only ignore rules

### src/modules/Modules.cpp
- **What:** Added a conditional SELFCIUS include and ifdef-guarded calls to `selfcius::initOfficerModules()` / `selfcius::initRelayMeshModules()` inside `setupModules()`
- **Why:** Provides the single firmware entry point for wrapper-owned SELFCIUS modules. Stock builds compile these lines out because `SELFCIUS_OFFICER` and `SELFCIUS_RELAY_MESH` are not defined.
- **Conflict risk:** Low - insertion is in the existing "Put your module here" area before `RoutingModule`, which must remain last

### src/modules/AdminModule.cpp
- **What:** Guarded the OTA admin request path so it only uses `MeshtasticOTA` when WiFi is enabled, and returns a warning instead of rebooting on unsupported builds
- **Why:** SELFCIUS trims WiFi for ESP32 officer/relay builds. Without this guard, `AdminModule` references OTA symbols that are not compiled in and breaks the build.
- **Conflict risk:** Medium - upstream OTA/admin changes could touch the same switch case

### platformio.ini
- **What:** Excluded `selfcius/` from the default Arduino `build_src_filter`
- **Why:** The wrapper repo exposes SELFCIUS sources into `src/selfcius` via symlink for custom environments. Stock Meshtastic builds must ignore that tree unless a SELFCIUS-specific environment explicitly opts back in.
- **Conflict risk:** Medium - upstream build filter changes in `platformio.ini` could overlap

### CLAUDE.md
- **What:** Added a SELFCIUS dependency tracking section listing the Meshtastic internal APIs the current overlay depends on
- **Why:** Upstream merges need one place to check which internals are part of the active SELFCIUS fork surface before resolving conflicts or refactors
- **Conflict risk:** Low - documentation-only change in the fork guidance file

### New Files

<!-- Files added that don't exist in upstream -->

_None yet._

### Deleted Files

<!-- Upstream files removed intentionally -->

_None yet._

## Dependencies

### protobufs (nested submodule)
- **Current policy:** Pin upstream `meshtastic/protobufs`; do not make protobuf changes part of initial v4 work.
- **Rationale:** Initial DTN work can use existing Meshtastic payload surfaces. Proto changes require coordinated regeneration across firmware, android, and apple, so they should be introduced only when a concrete requirement needs them.
- **Future trigger:** If v4 later needs custom mesh messages, port numbers, generated API fields, or shared protocol definitions that cannot fit cleanly in existing payloads, promote protobufs to a tracked v4 fork before landing the protocol change.
- **Escalation path:** Execute the "Forking protobufs later" path in the wrapper repo, update `upstream-versions.json`, document app generation impact, and record all protobuf edits in this file.
- **Action on upstream merge:** Accept upstream's protobufs pointer as-is unless a tracked v4 protobuf fork has been created.

## SELFCIUS Dependency Tracking

SELFCIUS overlay code currently depends on these Meshtastic internal APIs.
Check this table during upstream merges and update it whenever wrapper-owned
SELFCIUS code starts using a new internal surface.

| SELFCIUS usage | Meshtastic API | File | Merge risk |
|----------------|----------------|------|------------|
| Officer private-port receive/send path | `SinglePortModule`, `allocDataPacket()` | `mesh/SinglePortModule.h` | Low |
| Relay packet observation | `MeshModule` and `isPromiscuous` | `mesh/MeshModule.h` | Low |
| Periodic SELFCIUS worker threads | `concurrency::OSThread` | `concurrency/OSThread.h` | Low |
| DTN packet send submission | `service->sendToMesh()` | `mesh/MeshService.h` | Low |
| Officer GPS read path | `gps`, `gps->p`, `gps->hasLock()`, `gps->newStatus` | `gps/GPS.h` | Medium |
| Meshtastic-maintained local position state | `localPosition` and NodeDB helpers | `mesh/NodeDB.h` | Medium |
| Relay hop metrics | `meshtastic_MeshPacket::hop_start`, `hop_limit` | generated mesh packet types | Low |
| Officer SOS button observation | `inputBroker`, `InputEvent`, `INPUT_BROKER_USER_PRESS`, `CallbackObserver` | `input/InputBroker.h` | Medium |
| Officer OLED status/alert pages | `graphics::Screen`, `UIFrameEvent`, `OLEDDisplay`, `OLEDDisplayUi`, `ScreenFonts`, `screen->startAlert()`, `screen->endAlert()`, `screen->showSimpleBanner()` | `graphics/Screen.h`, OLED display headers | Medium |
| SELFCIUS scheduler wake-up | `concurrency::mainDelay.interrupt()` | `concurrency/Periodic.h` | Low |
| Wrapper logging and error reporting | `LOG_INFO`, `LOG_DEBUG`, `LOG_WARN`, `LOG_ERROR` | Meshtastic logging macros | Low |
| Transitive include from every SELFCIUS .cpp | `configuration.h` | `configuration.h` | Low |
| DTN filesystem abstraction | `FSCom` | `FSCommon.h` | Low |
| DTN filesystem locking | `spiLock` | `SPILock.h` | Medium |
| SOS scheduler jitter source | `esp_random()` | ESP-IDF random API | Low |

**Last verified:** 2026-04-27 against `v2.7.22.96dd647-7-g119839956` after Phase 1A/1B bench gate

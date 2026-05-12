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

## SELFCIUS Bench Identity

- USB serial paths are not durable node identity. Always run the wrapper scanner before flashing or resetting: `../scripts/selfcius-scan-nodes.sh` from this submodule, or `./scripts/selfcius-scan-nodes.sh` from the wrapper root.
- Use explicit `--port` values for flash/reset commands, but choose that port from fresh scanner output using node ID, MAC, role, and `pioEnv`; do not rely on `/dev/cu.usbserial*` names.
- Known bench identities:

| Node ID | Role | Firmware Env | Notes |
|---------|------|--------------|-------|
| `!9ea202b4` | ROUTER | `selfcius-relay-mesh` | Board A relay |
| `!9ea3d218` | CLIENT | `selfcius-officer` | Officer D218 |
| `!db528154` | CLIENT | `selfcius-officer` | Officer 8154 |
| non-Meshtastic | — | `selfcius-relay-lorawan` | Board B standalone ESP32-S3; identify by serial boot banner `SELFCIUS Board B — v0.2.0` |

- Officer GPS provisioning requires Meshtastic config readback: `position.rx_gpio=5`, `position.tx_gpio=4`, and `position.gps_mode=ENABLED`.
- Never remove an SX1262 antenna while transmitting; verify LoRaWAN failure recovery without antenna-disconnect tests.

## Rules

- Always merge upstream, **never rebase v4**
- Update the V4 Modifications section below when changing files
- Do NOT modify protobufs without checking the Dependencies section
- Feature work goes on branches off v4, merged back to v4
- After pushing v4, update the wrapper repo submodule SHA
- Treat SELFCIUS evidence as bench validation unless a gate document explicitly says field validation

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

### src/mesh/PhoneAPI.cpp
- **What:** Skipped the recursive filesystem manifest scan for `SPECIAL_NONCE_ONLY_NODES` BLE config requests.
- **Why:** SELFCIUS officers can have many persisted `/selfcius/rec/*.dat` custody files; rebuilding the file manifest for node-info-only requests wastes heap and caused Officer 8154 to abort in `getFiles()` during BLE config sync.
- **Conflict risk:** Medium - upstream phone API/config-sync changes may touch the same startup state machine.

### platformio.ini
- **What:** Excluded `selfcius/` from the default Arduino `build_src_filter`
- **Why:** The wrapper repo exposes SELFCIUS sources into `src/selfcius` via symlink for custom environments. Stock Meshtastic builds must ignore that tree unless a SELFCIUS-specific environment explicitly opts back in.
- **Conflict risk:** Medium - upstream build filter changes in `platformio.ini` could overlap

### CLAUDE.md
- **What:** Updated guidance with durable SELFCIUS bench node identity, scanner-first flash/reset rules, GPS provisioning readback requirements, and bench-validation wording
- **Why:** USB serial device paths change between sessions; future agents must map hardware by node ID/MAC/role/`pioEnv` and avoid overstating bench evidence as field readiness
- **Conflict risk:** Low - documentation-only update to fork guidance
- **What:** Added a SELFCIUS dependency tracking section listing the Meshtastic internal APIs the current overlay depends on
- **Why:** Upstream merges need one place to check which internals are part of the active SELFCIUS fork surface before resolving conflicts or refactors
- **Conflict risk:** Low - documentation-only change in the fork guidance file
- **What:** Updated SELFCIUS dependency tracking for Phase 1C GPS timestamp/RTC checks and relay receive/store surfaces
- **Why:** Phase 1C now uses RTC-relative GPS timestamp checks and relay-local DTN storage paths that need explicit upstream merge checks
- **Conflict risk:** Low - documentation-only update to fork guidance
- **What:** Updated SELFCIUS dependency tracking for Phase 1C review-fix surfaces: receive-queue mutexes and relay response suppression
- **Why:** Review fixes depend on FreeRTOS mutex helpers and `MeshModule::ignoreRequest`, so upstream merges need those surfaces called out explicitly
- **Conflict risk:** Low - documentation-only update to fork guidance
- **What:** Removed `GPSStatus::getLastFixMillis()` from the active SELFCIUS dependency list
- **Why:** Phase 1C treats Meshtastic last-fix age as freshness-unverified until a reliable GPS freshness signal is selected
- **Conflict risk:** Low - documentation-only update to fork guidance
- **What:** Updated Phase 1C verification tracking after the provenance hardware reflash
- **Why:** Relay and officer bench nodes were reflashed from the latest Phase 1C branch tip, so the fork guidance must no longer describe the current bench flash as the older `9fecb59a5` build
- **Conflict risk:** Low - documentation-only update to fork guidance
- **What:** Added Phase 1C compatibility migration from legacy `SDTN` stored-record frames to current `SDT2` frames with ingress provenance
- **Why:** Phase 1C extended persisted DTN records with ingress node IDs; legacy bench records must migrate without being mistaken for corrupt segments or silently removed
- **Conflict risk:** Low - wrapper-owned SELFCIUS storage codec/backend only

### src/selfcius/common/dtn/selfcius_dtn_storage.h
- **What:** Added a backend append-reason hook so DTN callers can distinguish storage failure modes beyond a bare `-1`
- **Why:** Hardware validation on Officer D218 surfaced `store_result=5` without enough context to tell full-storage from LittleFS write/open failures
- **Conflict risk:** Low - wrapper-owned DTN backend interface used only by SELFCIUS storage implementations

### src/selfcius/common/dtn/selfcius_dtn_store.h
- **What:** Exposed the last DTN backend append result through `DtnStore`
- **Why:** Officer-side logs need the backend failure reason that triggered `BackendFailure` during live bench validation
- **Conflict risk:** Low - wrapper-owned DTN store surface

### src/selfcius/common/dtn/selfcius_dtn_store.cpp
- **What:** Captured backend append failure reasons when `storage.append()` fails
- **Why:** Preserves the true LittleFS failure boundary for officer diagnostics instead of collapsing every append failure into the same opaque result
- **Conflict risk:** Low - wrapper-owned DTN store logic

### src/selfcius/common/dtn/selfcius_littlefs_storage.h
- **What:** Added typed LittleFS append failure reasons
- **Why:** Officer validation needs to tell apart full-storage, no-free-slot, encode, open, and short-write failures without destructive probing
- **Conflict risk:** Low - wrapper-owned LittleFS backend surface

### src/selfcius/common/dtn/selfcius_littlefs_storage.cpp
- **What:** Returned structured LittleFS append/write failure reasons instead of a single generic failure path
- **Why:** Makes Officer D218 `store_result=5` diagnostics actionable during bench validation and preserves non-destructive debugging
- **Conflict risk:** Low - wrapper-owned LittleFS backend implementation

### src/selfcius/common/dtn/selfcius_memory_storage.h
- **What:** Added last-append-result tracking to the native in-memory DTN backend
- **Why:** Keeps host-native tests aligned with the new DTN backend failure introspection API
- **Conflict risk:** Low - wrapper-owned native test backend

### src/selfcius/common/dtn/selfcius_memory_storage.cpp
- **What:** Recorded the last append result in the native in-memory DTN backend
- **Why:** Supports red-green tests for DTN backend failure provenance
- **Conflict risk:** Low - wrapper-owned native test backend

### src/selfcius/officer/selfcius_officer_module.cpp
- **What:** Expanded officer DTN store failure logs to include symbolic store-result names, LittleFS append failure reasons, and current stored-record count
- **Why:** Hardware validation on Officer D218 showed `store_result=5` but not the concrete storage boundary that caused it
- **Conflict risk:** Low - wrapper-owned officer overlay diagnostics

### src/selfcius/relay_lorawan/lorawan_driver.cpp
- **What:** Normalized the RadioLib ABP session RX timing to the custom TTS network's 5-second RX1 / 6-second RX2 schedule after session restore or activation.
- **Why:** Hardware E2E on `ttn.hazemon.in.th` showed backend ACK downlinks were being scheduled around 5 seconds after uplink while Board B was opening an earlier RX window, causing custody records to remain unreleased despite backend ACK queueing.
- **Conflict risk:** Low - wrapper-owned Board B LoRaWAN driver, but revisit if RadioLib session-buffer offsets change.

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
| Relay packet observation and private-port receive/store path | `MeshModule`, `isPromiscuous`, `ignoreRequest`, decoded payload metadata from `meshtastic_MeshPacket` | `mesh/MeshModule.h`, generated mesh packet types | Medium |
| Periodic SELFCIUS worker threads | `concurrency::OSThread` | `concurrency/OSThread.h` | Low |
| Officer and relay callback receive queue locking | `freertosinc.h`, `StaticSemaphore_t`, `xSemaphoreCreateMutexStatic()`, `xSemaphoreTake()`, `xSemaphoreGive()` | `freertosinc.h` | Medium |
| DTN packet send submission | `service->sendToMesh()` | `mesh/MeshService.h` | Low |
| Officer GPS read path | `gps`, `gps->p`, `gps->hasLock()`, `getValidTime(RTCQualityDevice)` | `gps/GPS.h`, `gps/RTC.h` | Medium |
| Meshtastic-maintained local position state | `localPosition` and NodeDB helpers | `mesh/NodeDB.h` | Medium |
| Relay hop metrics | `meshtastic_MeshPacket::hop_start`, `hop_limit` | generated mesh packet types | Low |
| Officer SOS button observation | `inputBroker`, `InputEvent`, `INPUT_BROKER_USER_PRESS`, `CallbackObserver` | `input/InputBroker.h` | Medium |
| Officer OLED status/alert pages | `graphics::Screen`, `UIFrameEvent`, `OLEDDisplay`, `OLEDDisplayUi`, `ScreenFonts`, `screen->startAlert()`, `screen->endAlert()`, `screen->showSimpleBanner()` | `graphics/Screen.h`, OLED display headers | Medium |
| SELFCIUS scheduler wake-up | `concurrency::mainDelay.interrupt()` | `concurrency/Periodic.h` | Low |
| Wrapper logging and error reporting | `LOG_INFO`, `LOG_DEBUG`, `LOG_WARN`, `LOG_ERROR` | Meshtastic logging macros | Low |
| Transitive include from every SELFCIUS .cpp | `configuration.h` | `configuration.h` | Low |
| Officer and relay DTN filesystem abstraction | `FSCom` | `FSCommon.h` | Low |
| Officer and relay DTN filesystem locking | `spiLock` | `SPILock.h` | Medium |
| SOS scheduler jitter source | `esp_random()` | ESP-IDF random API | Low |

**Last software verified:** 2026-04-28 against `daf00521e` / firmware `2.7.22.daf0052` during Phase 1C finding-fix verification.
**Last hardware verified:** 2026-04-28 against `daf00521e` / firmware `2.7.22.daf0052` for relay observe/store bench proof; officers remained on prior Phase 1C firmware during the final relay-only finding-fix pass.

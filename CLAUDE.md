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

### src/FSCommon.cpp
- **What:** `getFiles()` no longer descends into the `/selfcius/rec` DTN custody record directory when building a file listing/manifest (guarded by an exact-path check; `listDir()` and its delete path are untouched).
- **Why:** This is the non-`SPECIAL_NONCE_ONLY_NODES` half of the PhoneAPI manifest fix above. A loaded officer/relay holds hundreds of `/selfcius/rec/*.dat` custody files; enumerating them into the `want_config` file manifest delayed config-complete on every CLI connect (the V4-relevant leg of the CLI-wedge, which does not reset on serial open). Skipping the subtree during descent avoids the enumeration cost entirely; the directory only exists on SELFCIUS builds, so stock firmware is unaffected.
- **Conflict risk:** Medium - upstream filesystem/manifest changes may touch `getFiles()`.

### platformio.ini
- **What:** Excluded `selfcius/` from the default Arduino `build_src_filter`
- **Why:** The wrapper repo exposes SELFCIUS sources into `src/selfcius` via symlink for custom environments. Stock Meshtastic builds must ignore that tree unless a SELFCIUS-specific environment explicitly opts back in.
- **Conflict risk:** Medium - upstream build filter changes in `platformio.ini` could overlap

### variants/selfcius/platformio.ini
- **What:** Added env:selfcius-officer-v4 (extends env:heltec-v4) for the Heltec V4 officer role; reuses the officer -D flags + build_src_filter, inherits V4 16MB partitions + FEM handling.
- **Why:** Defines wrapper-owned SELFCIUS PlatformIO environments without changing stock Meshtastic build targets.
- **Conflict risk:** Low - wrapper-owned overlay configuration loaded only by the SELFCIUS symlink setup

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
- **What:** Updated SELFCIUS dependency tracking for the ADR-014 officer GPS sampler path: `nodeDB->localPosition` plus trusted RTC time, not direct `gps`, `gps->p`, or `gps->hasLock()` reads.
- **Why:** The D2 capture-gate fix moved the live sampler away from instantaneous GPS lock state, so upstream merge checks must track the current NodeDB/RTC surfaces and not reintroduce stale GPS-pointer assumptions.
- **Conflict risk:** Low - documentation-only update to fork guidance

### src/selfcius/common/selfcius_config.h
- **What:** Added `SELFCIUS_DTN_PRIVATE_CHANNEL_INDEX` as the shared compile-time DTN channel index for officer transmit and relay admission.
- **Why:** Stage 1 private-channel migration needs one firmware-wide channel index before provisioning scripts and bench PSK rollout wire a non-default value.
- **Conflict risk:** Low - wrapper-owned SELFCIUS config header.
- **What:** Added `SELFCIUS_RELAY_ALLOWLIST_ENABLED`, defaulting to `0`, with a 0/1 static assertion for compile-time lab allowlist overrides.
- **Why:** ADR-009 admission now relies on channel-key membership; the per-node relay allowlist must default open in firmware source while remaining available as explicit lab defense-in-depth.
- **Conflict risk:** Low - wrapper-owned SELFCIUS config header.
- **What:** Added relay replay-floor and DTN metadata cap constants.
- **Why:** Stage-1 increment 4 needs a bounded persistent per-origin monotonic floor table for replay rejection that survives relay record eviction and reboot.
- **Conflict risk:** Low - wrapper-owned SELFCIUS config header.
- **What:** Wrapped the three peer-SOS carry timing constants (`SELFCIUS_DTN_PEER_SOS_CARRY_MIN_INTERVAL_MS`, `SELFCIUS_DTN_PEER_SOS_CARRY_MAX_INTERVAL_MS`, and `SELFCIUS_DTN_PEER_SOS_PRESSURE_DEFER_MS`) in `#ifndef` guards, with defaults unchanged.
- **Why:** G2 carry-forward bench needs compressed timing overrides while production keeps shipping values.
- **Conflict risk:** Low - wrapper-owned SELFCIUS config header.

### src/selfcius/common/dtn/selfcius_dtn_storage.h
- **What:** Added a backend append-reason hook so DTN callers can distinguish storage failure modes beyond a bare `-1`
- **Why:** Hardware validation on Officer D218 surfaced `store_result=5` without enough context to tell full-storage from LittleFS write/open failures
- **Conflict risk:** Low - wrapper-owned DTN backend interface used only by SELFCIUS storage implementations
- **What:** Added a minimal optional backend metadata blob API.
- **Why:** Relay replay-floor metadata must persist independently of individual DTN record files so replay protection survives cap eviction and reboot.
- **Conflict risk:** Low - wrapper-owned DTN backend interface used only by SELFCIUS storage implementations

### src/selfcius/common/dtn/selfcius_relay_dtn_store.h
- **What:** Added in-memory per-origin replay-floor state for relay-observed records.
- **Why:** Stage-1 increment 4 requires relay inbound monotonic sequence floors to reject equal/lower origin sequences per origin.
- **Conflict risk:** Low - wrapper-owned relay DTN store surface.
- **What:** Moved the relay replay-floor metadata scratch buffer onto `RelayDtnStore` instead of using per-call stack arrays.
- **Why:** ~4KB stack frames in the acceptance path were the most likely reboot vector on the 8KB ESP32 loop stack, so the scratch buffer now lives on the heap-backed relay store object.
- **Conflict risk:** Low - wrapper-owned relay DTN store surface.

### src/selfcius/common/dtn/selfcius_relay_dtn_store.cpp
- **What:** Loads, persists, rebuild-merges, and enforces per-origin relay replay floors, mapping floor hits to `RejectedByPolicy`.
- **Why:** Relay inbound replay rejection must survive record purge/eviction and reboot while advancing only after a record is actually accepted/stored.
- **Conflict risk:** Low - wrapper-owned relay DTN store logic.
- **What:** Reused the object-owned replay-floor metadata scratch buffer in load and persist paths instead of allocating 4KB scratch arrays on the stack.
- **Why:** ~4KB stack frames in the acceptance path were the most likely reboot vector on the 8KB ESP32 loop stack, so the metadata path now avoids that stack pressure.
- **Conflict risk:** Low - wrapper-owned relay DTN store logic.
- **What:** At the per-origin record cap, a routine (non-SOS) GPS record now also reclaims a DELIVERED (`BoardBStored`) same-origin record (dropped the prior `!sos` gate on `evictDeliveredForOrigin`); undelivered/in-flight records stay protected (backpressure).
- **Why:** Only SOS could reclaim before, so a sustained-GPS officer with no SOS saturated its 64-slot quota and every further record `cap_rejected` -- which stops UART export (only `Captured` records forward) and silently stalled the whole custody chain (hardware-confirmed 2026-06-16: relay `gps=147 disp=cap_rejected fwd=0`, Board B `UART bytes=0`). Preserves REQ:SR-4 delivered-trail/eviction-priority (store still bounded at the cap); does not shrink the store.
- **Conflict risk:** Low - wrapper-owned relay DTN store acceptance logic.

### src/selfcius/common/dtn/selfcius_dtn_store.h
- **What:** Exposed the last DTN backend append result through `DtnStore`
- **Why:** Officer-side logs need the backend failure reason that triggered `BackendFailure` during live bench validation
- **Conflict risk:** Low - wrapper-owned DTN store surface
- **What:** Added a bounded coordinate-free DTN snapshot API for aggregate record counts and per-origin sequence ranges.
- **Why:** AIT campus field checking needs post-walk officer readback evidence for own/heard GPS and SOS records without exposing real coordinates.
- **Conflict risk:** Low - wrapper-owned DTN store read-only introspection.

### src/selfcius/common/dtn/selfcius_dtn_store.cpp
- **What:** Captured backend append failure reasons when `storage.append()` fails
- **Why:** Preserves the true LittleFS failure boundary for officer diagnostics instead of collapsing every append failure into the same opaque result
- **Conflict risk:** Low - wrapper-owned DTN store logic

### src/selfcius/common/dtn/selfcius_littlefs_storage.h
- **What:** Added typed LittleFS append failure reasons
- **Why:** Officer validation needs to tell apart full-storage, no-free-slot, encode, open, and short-write failures without destructive probing
- **Conflict risk:** Low - wrapper-owned LittleFS backend surface
- **What:** Exposed support for the optional DTN metadata blob API.
- **Why:** Relay replay-floor metadata needs a small persistent LittleFS sidecar separate from record slots.
- **Conflict risk:** Low - wrapper-owned LittleFS backend surface

### src/selfcius/common/dtn/selfcius_littlefs_storage.cpp
- **What:** Returned structured LittleFS append/write failure reasons instead of a single generic failure path; added watchdog-safe directory scans during `init()`/`clear()`, deferred `.dat.tmp` orphan cleanup to a post-scan pass, and bounded slot enumeration to the storage cap. `updateStatus()` now reuses the atomic temp+rename path of `replace()` instead of truncating the record file in place
- **Why:** Makes Officer D218 `store_result=5` diagnostics actionable during bench validation, and Board B 512-record real-flash drills showed long LittleFS scans can trip the watchdog and that stale `.dat.tmp` orphans must be reconciled on mount without destroying `.dat` custody files. In-place `updateStatus()` truncation left a corruption window where a reset during a custody status transition could lose the only on-flash copy of a record (audit F28)
- **Conflict risk:** Low - wrapper-owned LittleFS backend implementation
- **What:** Added a keyed metadata sidecar read/write path, using temp-then-rename writes, and clears it with DTN storage.
- **Why:** Relay replay floors must persist across record eviction and reboot while keeping the previous floor intact if a metadata write fails mid-update; destructive storage clear operations must still reset the sidecar.
- **Conflict risk:** Low - wrapper-owned LittleFS backend implementation

### src/selfcius/common/dtn/selfcius_memory_storage.h
- **What:** Added last-append-result tracking to the native in-memory DTN backend
- **Why:** Keeps host-native tests aligned with the new DTN backend failure introspection API
- **Conflict risk:** Low - wrapper-owned native test backend
- **What:** Added an in-memory metadata blob buffer cleared with storage.
- **Why:** Native tests need replay-floor persistence across relay store object rebuilds without using LittleFS.
- **Conflict risk:** Low - wrapper-owned native test backend

### src/selfcius/common/dtn/selfcius_memory_storage.cpp
- **What:** Recorded the last append result in the native in-memory DTN backend
- **Why:** Supports red-green tests for DTN backend failure provenance
- **Conflict risk:** Low - wrapper-owned native test backend
- **What:** Implemented read/write support for the in-memory metadata blob.
- **Why:** Relay replay-floor tests need metadata to survive record purge and store reconstruction in the native environment.
- **Conflict risk:** Low - wrapper-owned native test backend

### src/selfcius/officer/selfcius_officer_module.cpp
- **What:** Expanded officer DTN store failure logs to include symbolic store-result names, LittleFS append failure reasons, and current stored-record count
- **Why:** Hardware validation on Officer D218 showed `store_result=5` but not the concrete storage boundary that caused it
- **Conflict risk:** Low - wrapper-owned officer overlay diagnostics
- **What:** Sends SELFCIUS DTN packets on `SELFCIUS_DTN_PRIVATE_CHANNEL_INDEX` instead of hardcoded channel 0.
- **Why:** Prepares officer firmware for the dedicated private Meshtastic channel while preserving channel-0 behavior until provisioning changes the index.
- **Conflict risk:** Low - wrapper-owned officer overlay module.
- **What:** Threads `SELFCIUS_DTN_PRIVATE_CHANNEL_INDEX` through the officer receive drain path and packet admission check instead of hardcoded channel 0.
- **Why:** Keeps officer receive symmetry with transmit/admission so the future non-default private channel can work without silently breaking peer-officer DTN.
- **Conflict risk:** Low - wrapper-owned officer mesh transport module.
- **What:** Logs a coordinate-free DTN snapshot after officer storage rebuild, including aggregate counts and per-origin suffix/sequence ranges.
- **Why:** Field-check evidence needs a durable post-run readback path; serial reconnects reboot the board, so the boot log must expose LittleFS-backed state after rebuild.
- **Conflict risk:** Low - wrapper-owned officer diagnostics only.

### src/selfcius/relay_mesh/selfcius_relay_module.cpp
- **What:** Passes an explicit `RelayAdmissionConfig` using `SELFCIUS_DTN_PRIVATE_CHANNEL_INDEX`, `SELFCIUS_RELAY_ALLOWLIST_ENABLED != 0`, and forwarded SOS disabled.
- **Why:** Wires the relay admission seam into the live Board A path while making channel-key membership the default sender admission layer and keeping forwarded-origin SOS gated off until per-record auth exists.
- **Conflict risk:** Low - wrapper-owned relay overlay module.
- **What:** Gated relay rebuild allowlist policy on the same admission config as live packet processing, added the channel-0 boot warning, and made relay replay-floor table exhaustion audit-visible.
- **Why:** Prevents default-open live admission records from being purged on reboot by a stale hardcoded allowlist, surfaces lab/public-channel builds at boot, and makes replay-floor table exhaustion visible as a distinct in-memory audit counter.
- **Conflict risk:** Low - wrapper-owned relay overlay and SELFCIUS store/audit logic.

### src/selfcius/relay_lorawan/lorawan_driver.cpp
- **What:** Normalized the RadioLib ABP session RX timing to the custom TTS network's 5-second RX1 / 6-second RX2 schedule after session restore or activation.
- **Why:** Hardware E2E on `ttn.hazemon.in.th` showed backend ACK downlinks were being scheduled around 5 seconds after uplink while Board B was opening an earlier RX window, causing custody records to remain unreleased despite backend ACK queueing.
- **Conflict risk:** Low - wrapper-owned Board B LoRaWAN driver, but revisit if RadioLib session-buffer offsets change.

### src/selfcius/relay_lorawan/board_b_store.h
- **What:** Added a rebuild service hook and payload-level stored-key counting for Board B diagnostics.
- **Why:** Full 512-record real-flash rebuild and latest-GPS replacement drills need watchdog-safe scans plus proof that the expected newer origin/sequence/hash is present and the older same-origin key is absent.
- **Conflict risk:** Low - wrapper-owned Board B record store API.

### src/selfcius/relay_lorawan/board_b_store.cpp
- **What:** Services the optional hook during rebuild/count scans and exposes `countStoredKey()` for payload-level replacement proof. Latest-GPS supersession now collapses only `Received` records: a newer same-origin GPS no longer replaces an in-flight `UplinkPending`/`Uplinked` record (drop-older still applies against in-flight via `latestSeqForOrigin`).
- **Why:** Hardware drills showed long LittleFS scans can trip the watchdog, and count-only rebuild evidence cannot prove same-origin latest-GPS replacement. Superseding an in-flight record deleted it before its backend fingerprint ACK arrived, losing custody/audit of a record that had already been transmitted (audit F50).
- **Conflict risk:** Low - wrapper-owned Board B record store implementation.

### src/selfcius/relay_lorawan/src/main.cpp
- **What:** Added compile-gated USB bench commands for clearing records, creating/checking `.dat.tmp` orphans, and printing GPS replacement proof, plus watchdog servicing during Board B rebuilds.
- **Why:** Board B real-flash storage drills must be repeatable through reusable tooling and must prove payload-level replacement without enabling bench-only USB injection in production firmware.
- **Conflict risk:** Low - standalone wrapper-owned Board B firmware, compile-gated for bench-only commands.

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
| Officer no-position SOS timestamp read path | `getValidTime(RTCQualityDevice)` | `gps/RTC.h` | Medium |
| Officer GPS sampler freshness-gated capture (ADR-014 D2) | `nodeDB->localPosition`, `nodeDB->hasLocalPositionSinceBoot()`, and explicit complete-coordinate checks | `mesh/NodeDB.h` | Medium |
| Relay hop metrics | `meshtastic_MeshPacket::hop_start`, `hop_limit` | generated mesh packet types | Low |
| Officer SOS button observation | `inputBroker`, `InputEvent`, `INPUT_BROKER_USER_PRESS`, `CallbackObserver` | `input/InputBroker.h` | Medium |
| Officer OLED status/alert pages | `graphics::Screen`, `UIFrameEvent`, `OLEDDisplay`, `OLEDDisplayUi`, `ScreenFonts`, `screen->startAlert()`, `screen->endAlert()`, `screen->showSimpleBanner()` | `graphics/Screen.h`, OLED display headers | Medium |
| SELFCIUS scheduler wake-up | `concurrency::mainDelay.interrupt()` | `concurrency/Periodic.h` | Low |
| Wrapper logging and error reporting | `LOG_INFO`, `LOG_DEBUG`, `LOG_WARN`, `LOG_ERROR` | Meshtastic logging macros | Low |
| Transitive include from every SELFCIUS .cpp | `configuration.h` | `configuration.h` | Low |
| Officer and relay DTN filesystem abstraction | `FSCom` | `FSCommon.h` | Low |
| Officer and relay DTN filesystem locking | `spiLock` | `SPILock.h` | Medium |
| SOS scheduler jitter source | `esp_random()` | ESP-IDF random API | Low |

**Last software verified:** 2026-05-12 on the wrapper `selfcius/mesh-first-prd` branch: SELFCIUS native suite 303/303 passed, `selfcius-officer` and `selfcius-relay-mesh` builds passed, and Board B standalone envs built successfully.
**Last hardware verified:** 2026-05-12 on bench firmware `2.7.22.4af0965` plus Board B `v0.2.0`: all Meshtastic nodes were flashed with clean LittleFS images, factory-reset/configured with scanner-confirmed identities, Board B was chip-erased/reflashed/uploadfs, and the custom TTS backend ACK path was bench-proven to Board B custody release. Treat this as bench validation only, not field readiness.

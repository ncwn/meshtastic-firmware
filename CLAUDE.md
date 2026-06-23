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

## Nested Submodules

- `protobufs`: `origin` = `ncwn/protobufs`, `upstream` = `meshtastic/protobufs`, branch `v4`. The firmware pins the v4 fork commit (run `git submodule status protobufs` for the exact SHA); the v4 branch carries the SELFCIUS `admin.proto` additions on top of upstream `v2.7.21-6-ge30092e`.
- `meshtestic`: upstream Meshtastic test fixture submodule, unchanged.

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

### .gitmodules
- **What:** Repointed the nested `protobufs` submodule to `ncwn/protobufs` with branch `v4`, while keeping `upstream` as `meshtastic/protobufs` in the local checkout.
- **Why:** SELFCIUS epoch provisioning requires a deliberate protobuf fork path before changing `admin.proto`, matching the v4 fork workflow used by the other Meshtastic submodules.
- **Conflict risk:** Medium - protobuf upstream syncs and generated-code changes must coordinate with the pinned nested submodule commit.

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

### src/mesh/generated/meshtastic/admin.pb.{h,cpp}
- **What:** Regenerated nanopb admin bindings after adding the `SelfciusEpoch` admin payload in the nested protobufs submodule.
- **Why:** Officer firmware needs a typed admin request/response surface to read or set the SELFCIUS origin epoch without overloading unrelated Meshtastic admin fields.
- **Conflict risk:** Medium - generated admin bindings must stay in lock-step with `protobufs/meshtastic/admin.proto`.

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
- **What:** Added env:selfcius-relay-mesh-v4 (extends env:heltec-v4) for a Heltec V4 Board-A relay UART bench path, overriding only the relay UART pins to GPIO26/33 while leaving the V3 relay env on GPIO6/7.
- **Why:** Heltec V4 reserves the legacy V3 relay UART pins for FEM/board functions; the V4 relay bench needs a board-specific env without changing V3 defaults.
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
- **What:** Added relay replay-floor and DTN metadata cap constants; increased the metadata scratch cap to fit v2 epoch-aware replay-floor entries.
- **Why:** Stage-1 increment 4 needs a bounded persistent per-origin monotonic floor table for replay rejection that survives relay record eviction and reboot; the officer-epoch recovery path stores `(originNodeId,maxEpoch,maxSequence)` per floor.
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
- **What:** Added in-memory per-origin replay-floor state for relay-observed records; floor entries now track max accepted epoch plus max accepted sequence.
- **Why:** Stage-1 increment 4 requires relay inbound monotonic sequence floors to reject equal/lower origin sequences per origin, and the officer-epoch recovery path needs a higher epoch to reset the sequence floor without clearing relay storage.
- **Conflict risk:** Low - wrapper-owned relay DTN store surface.
- **What:** Moved the relay replay-floor metadata scratch buffer onto `RelayDtnStore` instead of using per-call stack arrays.
- **Why:** ~4KB stack frames in the acceptance path were the most likely reboot vector on the 8KB ESP32 loop stack, so the scratch buffer now lives on the heap-backed relay store object.
- **Conflict risk:** Low - wrapper-owned relay DTN store surface.
- **What:** Added a per-record relay policy-reject cause surface (`invalid_record` vs `replay_floor`) to the relay store/trace path.
- **Why:** Field diagnostics must distinguish no-valid-fix records from stale replay-floor lockout before protocol-level epoch work.
- **Conflict risk:** Low - wrapper-owned relay DTN store surface.

### src/selfcius/common/dtn/selfcius_relay_dtn_store.cpp
- **What:** Loads, persists, rebuild-merges, and enforces per-origin relay replay floors, mapping floor hits to `RejectedByPolicy`; v1 metadata loads as epoch 0 and future writes persist v2 `(origin,maxEpoch,maxSequence)` entries.
- **Why:** Relay inbound replay rejection must survive record purge/eviction and reboot while advancing only after a record is actually accepted/stored; higher officer epochs must be accepted even with lower sequence numbers so field recovery does not require manual relay erasure.
- **Conflict risk:** Low - wrapper-owned relay DTN store logic.
- **What:** Reused the object-owned replay-floor metadata scratch buffer in load and persist paths instead of allocating 4KB scratch arrays on the stack.
- **Why:** ~4KB stack frames in the acceptance path were the most likely reboot vector on the 8KB ESP32 loop stack, so the metadata path now avoids that stack pressure.
- **Conflict risk:** Low - wrapper-owned relay DTN store logic.
- **What:** At the per-origin record cap, a routine (non-SOS) GPS record now also reclaims a DELIVERED (`BoardBStored`) same-origin record (dropped the prior `!sos` gate on `evictDeliveredForOrigin`); undelivered/in-flight records stay protected (backpressure).
- **Why:** Only SOS could reclaim before, so a sustained-GPS officer with no SOS saturated its 64-slot quota and every further record `cap_rejected` -- which stops UART export (only `Captured` records forward) and silently stalled the whole custody chain (hardware-confirmed 2026-06-16: relay `gps=147 disp=cap_rejected fwd=0`, Board B `UART bytes=0`). Preserves REQ:SR-4 delivered-trail/eviction-priority (store still bounded at the cap); does not shrink the store.
- **Conflict risk:** Low - wrapper-owned relay DTN store acceptance logic.

### src/selfcius/common/relay/selfcius_relay_processor.{h,cpp}
- **What:** Sorts relay-parsed records by origin, epoch, then sequence before store admission, and accepts parser-only V2 LiveMesh GPS-position batches by converting them into the existing relay `GpsRecord` path.
- **Why:** V2 epoch-bearing batches must process lower epochs before higher epochs for the same origin so replay-floor updates are deterministic; the relay needs parser support before officer V2 transmit, Board B, or backend export are enabled.
- **Conflict risk:** Low - wrapper-owned relay receive path.

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

### src/selfcius/common/dtn/selfcius_stored_record_codec.{h,cpp}
- **What:** Bumped stored-record frames to `SDT3` by storing `originEpoch` beside the existing schema-1 GPS record body; older `SDT2`/`SDTN` frames still decode as epoch 0 and rewrite to current format.
- **Why:** Relay/officer LittleFS persistence must preserve nonzero epoch identities before Board B/UART/backend export are fully epoch-aware; otherwise epoch-bearing records reach the relay but fail storage encode.
- **Conflict risk:** Medium - on-flash SELFCIUS record format with compatibility migration.

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

### src/selfcius/common/protocol/selfcius_dtn_packet.{h,cpp}
- **What:** Added `originEpoch` to the in-memory GPS record/key model while keeping schema-1 GPS batches wire-compatible; schema-1 parse maps epoch to `0`, and schema-1 encode rejects nonzero epochs.
- **Why:** Epoch must become part of custody identity without silently transmitting a nonzero epoch through the legacy 27-byte packet format.
- **Conflict risk:** Medium - shared SELFCIUS DTN identity surface used by officer, relay, Board B, and tests.

### src/selfcius/common/protocol/selfcius_v2_packet.{h,cpp}
- **What:** Added `originEpoch` to the existing disabled V2 record envelope and V2 canonical hash, restricted the legacy V1-compatible GPS hash scheme to epoch `0`, and updated V2 GPS batch capacity from 6 to 5 records.
- **Why:** Protocol-level replay-floor recovery needs epoch in the V2 identity/hash before V2 transmit is enabled; keeping `SELFCIUS_DTN_V2_TRANSMIT_ENABLED=0` preserves parser-first rollout.
- **Conflict risk:** Medium - disabled V2 protocol wire layout changed intentionally; coordinate with any future V2 transmit rollout.

### src/selfcius/officer/selfcius_sequence_floor.h, src/selfcius/officer/selfcius_sequence_state.h
- **What:** Added durable `originEpoch` beside the officer sequence floor, defaulting legacy floor-only state to epoch `0` and preserving epoch when reserving future sequence floors.
- **Why:** Replay-floor recovery needs a stable officer generation value that survives normal firmware reflashes and is stamped independently from the monotonic sequence counter.
- **Conflict risk:** Medium - shared officer custody identity state; coordinate with provisioning and relay replay-floor rollout.

### src/selfcius/officer/gps/selfcius_gps_sampler.{h,cpp}
- **What:** Threaded `originEpoch` through GPS sample input and stamped it onto valid-fix, stale-SOS, and no-lock SOS records.
- **Why:** Every officer-originated custody record must carry the persisted generation before V2 transmit and epoch-aware replay floors are enabled.
- **Conflict risk:** Low - wrapper-owned officer GPS capture path with native coverage.

### src/selfcius/officer/mesh/selfcius_mesh_transport.cpp
- **What:** Encodes officer live mesh batches with the existing V2 LiveMesh packet when any selected GPS record has a nonzero origin epoch; epoch-0 records continue using legacy schema-1 encoding.
- **Why:** Protocol-level replay-floor recovery requires bumped-epoch officers to transmit epoch-bearing records instead of failing legacy schema-1 encode and staying silent.
- **Conflict risk:** Medium - changes the officer live mesh wire path for nonzero-epoch records while preserving epoch-0 compatibility.

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
- **What:** Handles the SELFCIUS admin epoch request/response, allowing local admin clients to read or set the persisted `originEpoch` and read the reserved sequence floor.
- **Why:** Factory reset/reprovision must explicitly set an officer generation instead of relying on a local counter erased by reset.
- **Conflict risk:** Medium - wrapper-owned officer module, but depends on the generated admin protobuf surface.

### src/selfcius/common/relay/selfcius_relay_log_format.{h,cpp}
- **What:** Relay per-record logs now append `policy_reason=<none|invalid_record|replay_floor>` with native totality/format tests and summarizer token binding.
- **Why:** Bench/field log analysis needs to classify `policy_rejected` records without destructive relay-floor resets.
- **Conflict risk:** Low - wrapper-owned SELFCIUS log contract.

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
- **What:** Reworked the ABP downlink path for the TTS `MAC_V1_0_3` reprovision: re-pin a fixed datarate (`setADR(false)+setDatarate(LORAWAN_DR)`) at the top of every uplink so a network `LinkADRReq` cannot drag this fixed-rate device to a dwell-invalid DR, and zero the confirmed-downlink counter to the `0xFF` sentinel on session restore to suppress a spurious `FCTRL_ACK` on the first post-reboot uplink (documented 16-bit `AFCntDown` rollover ceiling). The RX1/RX2 normalization above is retained; the earlier debug-only `rxDelays[]+=offset` / `scanGuard=800` GODMODE pokes were dropped (they broke the clean production build) → RadioLib defaults.
- **Why:** Resolved the long-open production fPort3 ACK blocker — TTS `MAC_V1_0_4` split downlink counters (NFCntDown vs AFCntDown) break RadioLib 7.6.0's rev-0 single-counter MIC check (false FCnt rollover → `RADIOLIB_ERR_MIC_MISMATCH -1112`); the 1.0.3 reprovision + DR re-pin + counter-reset fix it with no protocol change. HW full chain proven (`6674:1:1325`, `released=1`).
- **Conflict risk:** Low - wrapper-owned Board B LoRaWAN driver; revisit on RadioLib session-buffer or MAC-version changes.

### src/selfcius/relay_lorawan/board_b_store.h
- **What:** Added a rebuild service hook and payload-level stored-key counting for Board B diagnostics.
- **Why:** Full 512-record real-flash rebuild and latest-GPS replacement drills need watchdog-safe scans plus proof that the expected newer origin/sequence/hash is present and the older same-origin key is absent.
- **Conflict risk:** Low - wrapper-owned Board B record store API.
- **What:** Renamed the latest-GPS helper surface to generation-scoped `(originNodeId,originEpoch)` freshness checks.
- **Why:** Board B must not drop or collapse a higher-epoch low-sequence record behind an older generation's high replay floor.
- **Conflict risk:** Low - wrapper-owned Board B record store API.
- **What:** Added the on-disk-only `ReceivedUplinked` status (enum value 4) and documented the `everUplinked` field as rebuild-restored, not RAM-default.
- **Why:** `everUplinked` (the "was transmitted" flag gating backend-ACK release) was RAM-only, so a transmitted-then-stale-requeued record came back from a reboot as not-releasable and its queued fPort3 ACK was ignored — stranding custody (GPS self-heals via latest-GPS supersession; SOS does not). Persisting it via `ReceivedUplinked` lets `rebuildIndex` restore `everUplinked=true` across a reboot. The struct stays a pure aggregate (no default member initializer — the xtensa toolchain rejects it at the brace-init sites).
- **Conflict risk:** Medium - persistent on-disk status semantics; old stores (status 0-3) migrate cleanly, a firmware rollback strands status-4 records as inert (no false release).

### src/selfcius/relay_lorawan/board_b_store.cpp
- **What:** Services the optional hook during rebuild/count scans and exposes `countStoredKey()` for payload-level replacement proof. Latest-GPS supersession now collapses only `Received` records: a newer same-origin GPS no longer replaces an in-flight `UplinkPending`/`Uplinked` record (drop-older still applies against in-flight via `latestSeqForOrigin`).
- **Why:** Hardware drills showed long LittleFS scans can trip the watchdog, and count-only rebuild evidence cannot prove same-origin latest-GPS replacement. Superseding an in-flight record deleted it before its backend fingerprint ACK arrived, losing custody/audit of a record that had already been transmitted (audit F50).
- **Conflict risk:** Low - wrapper-owned Board B record store implementation.
- **What:** Made Board B exact ACK release and latest-GPS freshness use the full epoch-aware record key; rebuild collapse is now scoped per `(originNodeId,originEpoch)` generation.
- **Why:** Phase 5 custody identity must survive relay->Board B->LoRaWAN without treating epoch 11 seq 17 as older than epoch 10 seq 960000.
- **Conflict risk:** Low - wrapper-owned Board B record store implementation.
- **What:** `requeueStaleUplinkPending` now persists `ReceivedUplinked` (keeping RAM `status=Received` + `everUplinked=true`), and `rebuildIndex` restores `everUplinked=true` for both persisted `UplinkPending` and `ReceivedUplinked` (both only ever mean "transmitted" in production), while plain `Received` stays false (the safe direction).
- **Why:** Makes the backend-ACK release survive a reboot for an in-flight or stale-requeued record without ever releasing a never-transmitted record. Native 485/485 incl. three reboot pins (uplinkpending/requeued release=1, never-uplinked release=0); HW-proven (`released=1` vs the pre-fix `released=0 ignored=1`).
- **Conflict risk:** Medium - wrapper-owned Board B record store implementation; pairs with the board_b_store.h on-disk status change.

### src/selfcius/proto/selfcius_uart_frame.{h,cpp}
- **What:** Added `originEpoch` to relay-to-Board-B GPS batch records and kept the UART frame size bounded with a protocol static assert.
- **Why:** Board B must receive the same epoch-aware custody identity the relay admitted before any backend/full-chain epoch rollout.
- **Conflict risk:** Medium - wrapper-owned UART wire format between Board A and Board B; both sides must be flashed together.

### src/selfcius/proto/selfcius_lorawan_payload.{h,cpp}
- **What:** Added `originEpoch` to fPort1 GPS uplink records and fPort3 backend ACK records; reduced the proven 52-byte ACK cap to 3 records, DR0-DR2 uplink caps to 1 record, DR3/DR4 caps to 3 records, and DR5 cap to 7 records.
- **Why:** Backend custody ACKs must exact-match `(originNodeId,originEpoch,originSequence,payloadHash)` and uplink batches must still fit AS923 payload budgets plus the local uplink buffer.
- **Conflict risk:** Medium - wrapper-owned LoRaWAN/backend contract; coordinate with TTN decoder and backend ingest.

### src/selfcius/relay_mesh/uart_bridge/selfcius_uart_export_driver.cpp
- **What:** Tracks pending UART batch records by full `recordKeyFor(record)` instead of `(originNodeId,originSequence)`.
- **Why:** Relay custody status updates after Board B ACK/NACK must apply to the exact epoch-bearing record.
- **Conflict risk:** Low - wrapper-owned relay-to-Board-B export driver.

### src/selfcius/relay_lorawan/src/main.cpp
- **What:** Added compile-gated USB bench commands for clearing records, creating/checking `.dat.tmp` orphans, printing GPS replacement proof, and proving wrong-epoch vs exact-epoch ACK release, plus watchdog servicing during Board B rebuilds.
- **Why:** Board B real-flash storage drills must be repeatable through reusable tooling and must prove payload-level replacement and epoch-aware custody ACK identity without enabling bench-only USB injection in production firmware.
- **Conflict risk:** Low - standalone wrapper-owned Board B firmware, compile-gated for bench-only commands.
- **What:** Marks LoRaWAN uplink-pending records using the full epoch-aware record key and includes epoch in Board B duplicate/superseded/conflict logs.
- **Why:** Phase 5 backend ACK release and bench logs must distinguish generations with reused sequence numbers.
- **Conflict risk:** Low - standalone wrapper-owned Board B firmware.

### New Files

<!-- Files added that don't exist in upstream -->

_None yet._

### Deleted Files

<!-- Upstream files removed intentionally -->

_None yet._

## Dependencies

### protobufs (nested submodule)
- **Current status:** FORK EXECUTED. `protobufs` now tracks `ncwn/protobufs` branch `v4` (the escalation path below was triggered by the SELFCIUS officer-epoch `admin.proto` addition). `upstream` remains `meshtastic/protobufs`.
- **Rationale:** Initial DTN work used existing Meshtastic payload surfaces. Proto changes require coordinated regeneration across firmware, android, and apple, so they were introduced only when the epoch-provisioning admin message needed a dedicated payload.
- **Regeneration:** After editing `protobufs/meshtastic/*.proto`, regenerate firmware bindings with `./bin/regen-protos.sh` (needs `nanopb-0.4.9/generator-bin/protoc`; the repo's prebuilt is Linux x86 — on other hosts provide a local nanopb generator) and commit the regenerated `src/mesh/generated/...` with the schema bump.
- **Action on upstream merge:** Merge `upstream/v4`-relevant protobuf changes into `ncwn/protobufs` v4 (never rebase); keep the SELFCIUS `admin.proto` additions; regenerate and re-pin.
- **Outstanding:** `upstream-versions.json` does not yet track the nested protobufs fork (it lists only the three top-level submodules); add nested tracking before relying on `scripts/upstream-sync.sh` for protobufs.

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

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

_No modifications yet — v4 branch starts clean from upstream base._

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

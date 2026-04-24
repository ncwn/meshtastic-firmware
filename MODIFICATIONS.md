# V4 Modifications — meshtastic-firmware

> This file tracks all intentional changes made on the `v4` branch relative to
> upstream `meshtastic/firmware`. When merging a new upstream release, consult
> this list to understand which conflicts are expected vs accidental.

## Upstream Base

- **Tag:** v2.7.22.96dd647
- **Commit:** 01bd4cfb73bb7bc20ea4cf08c36d66c45b4bea35
- **Channel:** prerelease (alpha)

## Changed Files

<!-- Format:
### path/to/file.cpp
- **What:** Brief description of the change
- **Why:** Reason this modification is needed for the v4 project
- **Conflict risk:** Low / Medium / High when merging upstream
-->

_No modifications yet — v4 branch starts clean from upstream base._

## New Files

<!-- Files added that don't exist in upstream -->

_None yet._

## Deleted Files

<!-- Upstream files removed intentionally -->

_None yet._

## Dependencies

### protobufs (nested submodule)
- **Current policy:** Pin upstream `meshtastic/protobufs`; do not make protobuf changes part of initial v4 work.
- **Rationale:** Initial DTN work can use existing Meshtastic payload surfaces. Proto changes require coordinated regeneration across firmware, android, and apple, so they should be introduced only when a concrete requirement needs them.
- **Future trigger:** If v4 later needs custom mesh messages, port numbers, generated API fields, or shared protocol definitions that cannot fit cleanly in existing payloads, promote protobufs to a tracked v4 fork before landing the protocol change.
- **Escalation path:** Execute the "Forking protobufs later" path in the wrapper repo, update `upstream-versions.json`, document app generation impact, and record all protobuf edits in this file.
- **Action on upstream merge:** Accept upstream's protobufs pointer as-is unless a tracked v4 protobuf fork has been created.

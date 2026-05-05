# Bounded local client-app metadata storage over AdminMessage

- Start Date: 2026-05-05
- RFC PR: [Meshtastic/rfcs#12](https://github.com/Meshtastic/rfcs/pull/12)
- Affected Components: Protobufs, Firmware, all companion clients

## Summary

A small, bounded, local-node-only metadata store for Meshtastic companion apps, exposed through AdminMessage, so clients have an explicit place for non-secret convenience metadata without overloading user-visible fields or unrelated configuration surfaces.

Each record is keyed by a short ASCII `app_id`, carries an opaque payload of up to 512 bytes plus a small schema/version integer and a firmware-set timestamp, survives reboot, and never leaves the locally-connected node. The feature is intentionally **namespaced, not owned**: the firmware enforces shape and capacity but does not authenticate which companion app is writing.

## Motivation

Companion apps sometimes need a tiny amount of device-local convenience metadata that is not part of the mesh protocol and should not be advertised, routed, synchronized, or interpreted by firmware. Examples include an onboarding-seen flag, a schema marker, a migration marker, or a pairing hint.

Without an explicit storage surface, the tempting alternatives are poor fits. User identity fields such as `User.long_name` are visible protocol state and surface in NodeInfo on every other node's screen. Canned messages are user-facing message presets. Module configuration is semantically unrelated to companion app bookkeeping. Adding ad-hoc fields to `User` or `NodeInfo` for one-off app needs is mesh-visible, costs airtime, and couples firmware to per-app intent.

This RFC proposes a small, explicit, bounded store so clients have a safer place for non-secret convenience metadata without overloading user-visible fields or unrelated configuration surfaces. The store:

- gives clients an explicit, firmware-bounded storage slot,
- is local-only and opaque (no airtime cost, no NodeInfo bloat, no MQTT exposure),
- caps blast radius via strict size/record limits,
- requires no firmware logic per app, since the firmware only ever sees opaque bytes.

## Ecosystem Impact

- **Protobufs**: additive. One new top-level message (`ClientAppData`), one new on-disk wrapper (`LocalClientAppData`), four new `AdminMessage.payload_variant` fields (104..107). No renumbering, no breaking changes to existing types.
- **Firmware**: a small new module (`ClientAppDataStore`, ~150 LoC), a new persistence file (`/prefs/clientappdata.proto`), four new dispatch cases in `AdminModule`, and one new init line in `main.cpp`. Two `virtual` keywords added to `NodeDB::loadProto/saveProto` for testability (mirrors the existing `MockNodeDB::getMeshNode override` pattern in `test_traffic_management`).
- **Clients**: opt-in. Apps that don't use the feature are unaffected. Apps that do use it must treat it as optional and gracefully fall back when the firmware doesn't support it (older firmware will reply with `NOT_AUTHORIZED` or no reply at all for the new tags).
- **Coordination**: no client coordination required beyond agreeing on `app_id` strings (see "Convention vs Registry" under Unresolved Questions).

## Protocol Buffer Changes

All edits live in `meshtastic/admin.proto`, `meshtastic/admin.options`, `meshtastic/localonly.proto`, and a new `meshtastic/localonly.options`. The full diff is `+114 / -0` across four files, additive only. `buf format`, `buf lint`, and `buf breaking` against `master` all pass clean.

### New top-level message in `admin.proto`

```proto
message ClientAppData {
  string  app_id     = 1;   // ^[a-z0-9._-]{1,32}$
  uint32  version    = 2;   // app-defined schema version; firmware does not interpret
  bytes   payload    = 3;   // opaque to firmware, max 512 bytes
  fixed32 updated_at = 4;   // unix epoch seconds, firmware-set on write, 0 if no valid time
}
```

### New `AdminMessage.payload_variant` fields

```proto
ClientAppData set_client_app_data            = 104;
string        get_client_app_data_request    = 105;
ClientAppData get_client_app_data_response   = 106;
string        delete_client_app_data_request = 107;
```

Field numbers 104..107 sit immediately after `sensor_config = 103` in the next contiguous gap. The proto comment includes `TODO(maintainer): confirm field-number allocation` (see Unresolved Questions).

### New on-disk wrapper in `localonly.proto`

```proto
message LocalClientAppData {
  repeated ClientAppData records = 1;   // capped at 4 by localonly.options
  uint32   version               = 2;
}
```

### nanopb sizing (`.options` files)

```
*ClientAppData.app_id max_size:33
*ClientAppData.payload max_size:512
*AdminMessage.get_client_app_data_request max_size:33
*AdminMessage.delete_client_app_data_request max_size:33
*LocalClientAppData.records max_count:4
```

## Technical Details

The companion application sends an `AdminMessage` containing one of the four new oneof fields to its locally-connected node. The firmware's `AdminModule` dispatches the case, validates inputs against `^[a-z0-9._-]{1,32}$` for `app_id` and ≤512 bytes for `payload`, mutates the in-memory bounded slot table, and persists via the existing `NodeDB::saveProto` helper to `/prefs/clientappdata.proto`. On read, the firmware looks up the record by `app_id` and replies with a populated `ClientAppData`; on miss it replies with a `ClientAppData` whose `app_id` is empty.

Slot semantics:

- The store has exactly 4 fixed slots (no heap allocation).
- Overwriting an existing `app_id` reuses the same slot.
- Deleting compacts the trailing records to free the slot for re-use.
- Adding a fifth distinct `app_id` returns `NoSpace` → `Routing_Error_BAD_REQUEST`.

Lifecycle:

- Loaded on boot from `/prefs/clientappdata.proto`. Missing or undecodable file → empty store (no log noise on first boot).
- Persisted synchronously after every successful `set` / `remove`.
- Survives reboot via the standard Portduino/ESP/nRF filesystem-backed prefs.
- Cleared by `factoryReset()` via the existing `rmDir("/prefs")` path. No new firmware code required.

`updated_at` is **always** server-set via the firmware's existing `getValidTime(RTCQualityDevice)` helper. Caller-supplied values are ignored. If the firmware has no valid wall-clock time at write, the field is `0`. This gives clients a cheap way to detect that another admin-capable client has overwritten or deleted-then-recreated their record.

### Compatibility

- **Existing clients are unaffected.** The four new oneof tags are additive; clients that don't know about them never construct or receive them.
- **Older firmware is unaffected.** Companion apps built against this RFC will get either `Routing_Error_NOT_AUTHORIZED` (today's catch-all for unknown admin payloads on older builds) or no reply at all when talking to firmware that doesn't yet implement the feature. Both must be treated as "feature unavailable, fall back to app-local storage."
- **No version bump needed.** The change is wire-additive; existing get-config / get-owner / get-canned-message flows are untouched.
- **Feature detection.** `DeviceMetadata.firmware_version` already lets clients gate behavior by firmware version. Apps SHOULD probe with a `get_client_app_data_request` and treat any error or timeout as "not supported."

### Security and Privacy

This feature provides namespaced app metadata, not strong ownership isolation. The firmware validates `app_id` shape, payload size, and record limits, but it does not authenticate that a specific companion app owns a given `app_id`. Any admin-capable client may be able to write or delete any `app_id` unless future firmware policy adds stronger ownership controls. Clients must therefore treat stored metadata as untrusted, optional, and recoverable. Sensitive data, secrets, paid entitlement state, identity keys, session keys, trust authority, blocklists, and security-critical decisions must not be stored here.

Specifically:

- `app_id` prevents accidental name collisions between apps. It does NOT prove caller ownership.
- Any admin-capable client (local USB/BLE, or remote `pki_encrypted` admin) may overwrite or delete any `app_id`.
- Clients must treat payloads as untrusted, optional, and recoverable. The companion app's own local/cloud/export storage remains the source of truth. Node-local `ClientAppData` is only a convenience hint or cache.
- If a payload is missing, malformed, overwritten, stale, or fails an app-level integrity check, the client should ignore it and recreate safe default metadata from its own source of truth.
- Apps may include an app-level integrity check, MAC, or signature inside `payload` if useful, but this only detects tampering. It does not prevent another admin-capable client from overwriting or deleting the record.

What is NOT an appropriate use:

- Secrets of any kind (API tokens, BYO keys, passwords)
- Identity keys, session keys, ephemeral key shares
- Paid-entitlement / subscription / license state (a non-paying client could overwrite it)
- Trust authority, blocklists, contact-verification state
- Anything used to make security, routing, authentication, or purchase decisions

Mesh / privacy properties:

- Records are **never broadcast over LoRa**.
- Records are **never included in NodeInfo**.
- Records are **never relayed via MQTT**.
- The firmware never interprets `payload`. It is opaque bytes.
- Records are NOT part of the `backup_preferences` / `restore_preferences` flow in v1 (see Unresolved Questions).

#### Remote admin behavior (v1 proposal)

Conservative starting point, listed as an unresolved maintainer question:

- **Local clients** (`mp.from == 0`, USB/BLE bond) may `set_client_app_data`, `get_client_app_data_request`, and `delete_client_app_data_request`.
- **Remote authenticated admin** (`mp.pki_encrypted` matching one of `config.security.admin_key[3]`) may `get_client_app_data_request`. Reads follow the same authorization path as every other `get_*_request` tag.
- **Remote `set` and `delete`** are rejected with `Routing_Error_NOT_AUTHORIZED`. The firmware's `AdminModule` short-circuits before any store mutation.

### Performance

Storage budget per node:

- Worst-case on-disk size = `meshtastic_LocalClientAppData_size` = **2258 bytes** (4 × ~560-byte records + nanopb overhead).
- Worst-case in-memory footprint = same, statically allocated as `meshtastic_LocalClientAppData store_;`. No heap.

Wire / airtime cost:

- **Zero.** Local-only by construction. The firmware never originates a `ClientAppData` packet on its own.
- Each AdminMessage carrying `ClientAppData` is bounded by AdminMessage's existing transport budget (USB-CDC / BLE GATT). The 512-byte payload cap fits comfortably in a single BLE write.

CPU / flash:

- One indirect call per `loadProto`/`saveProto` (added `virtual` for testability). Negligible vs flash I/O cost.
- One `O(records_count)` linear scan per `get`/`set`/`remove`. With `records_count ≤ 4`, this is constant time.

#### Error / response behavior

Per the existing AdminMessage convention:

- `set_client_app_data`: success returns nothing; the existing auto-ACK at the end of `AdminModule::handleReceivedProtobuf` emits `Routing_Error_NONE`. Failures (invalid `app_id`, payload too large, no slot free, storage write failure, remote sender) reply with `Routing_Error_BAD_REQUEST` or `Routing_Error_NOT_AUTHORIZED`.
- `get_client_app_data_request`: success returns `get_client_app_data_response` with the populated `ClientAppData`. Miss returns the same response with `app_id == ""`. Invalid `app_id` returns `Routing_Error_BAD_REQUEST`.
- `delete_client_app_data_request`: success (record removed OR record was already missing) returns auto-ACK `Routing_Error_NONE` (idempotent, matching `remove_by_nodenum` / `remove_favorite_node` / `delete_file_request` precedent). Invalid `app_id`, storage error, or remote sender reply with `Routing_Error_BAD_REQUEST` or `Routing_Error_NOT_AUTHORIZED`.

This intentionally follows existing AdminMessage conventions and avoids introducing a one-off status-envelope pattern. **No new `ClientAppDataStatus` enum, no new `ClientAppDataResponse` envelope.** If the maintainers later want a richer admin-wide error model, that is a separate, cross-cutting RFC that this feature would adopt.

#### Limits (v1 proposal)

| | |
|---|---|
| `app_id` regex | `^[a-z0-9._-]{1,32}$` |
| `payload` max | 512 bytes |
| record max | 4 records |
| `updated_at` | firmware-set `fixed32` unix epoch seconds, may be 0 if no valid time |
| on-disk file | `/prefs/clientappdata.proto` |
| factory reset | clears the store via existing `rmDir("/prefs")` path |

#### Anti-goals

This feature is explicitly **NOT**:

- arbitrary NVRAM access
- a filesystem
- a database
- a plugin runtime
- mesh-visible metadata
- secure credential storage
- protected / private app storage
- an identity / auth / security / routing primitive
- a replacement for app-local / cloud / export storage

#### Testing

- **27 native tests** in `test/test_client_app_data/test_main.cpp` cover: store-level validation matrix (`isValidAppId` accept/reject), CRUD, payload-size bounds, overwrite-without-slot-consumption, delete-then-get NotFound, max-record enforcement, delete-frees-slot, server-set `updated_at`, persistence-survives-reinit, storage-error injection, and factory-reset semantics (modeled via mock `loadProto` returning `OTHER_FAILURE`).
- Both `bash bin/test-native-docker.sh -f test_default` (regression) and `bash bin/test-native-docker.sh -f test_client_app_data` pass clean. Zero compile errors, zero linker errors, zero warnings on any feature file.
- **Dispatch-path tests** (`AdminModule` cases for tags 104..107) are **harness-limited**: exercising them in isolation would require standing up channels / service / MeshPacket fakes that the native test harness does not provide, and would re-test glue logic the store-level cases already cover. The dispatch glue compiles and links via the same coverage build that runs the test suite. Documented inline at the top of `test/test_client_app_data/test_main.cpp`.

## Drawbacks

- Adds four new tags + two new messages to a frequently-edited proto. Every additional admin tag is a small forever-cost on the protocol surface.
- Without a per-app ownership primitive, the feature can be misused (an aggressive client could iterate `app_id`s and clobber peers). Documentation alone does not enforce good behavior.
- The bare-type response with empty-`app_id` NOT_FOUND sentinel is a slight novelty (existing get responses don't need a sentinel because they always have a meaningful default value). It's a small wart, but introducing a status envelope for one feature would be a bigger one.
- The 4-record cap is conservative. Apps that grow into "I want a few more slots" will have a hard time bumping it: increasing the cap is a wire-format change requiring nanopb regen and broad maintainer review.

## Rationale and Alternatives

### Why this design

- **Bounded > unbounded.** A fixed 4-slot, 512-byte-per-record table is small enough that the worst-case footprint (2.3 KB) is comfortable on every supported target, and small enough that no client is tempted to build a database on top.
- **Local-only > globally-replicated.** This is the primary anti-feature versus alternatives. We never want this metadata leaving the node.
- **Opaque > structured.** Firmware never inspects `payload`. Apps can iterate schemas without a firmware change.
- **AdminMessage > new portnum.** Apps already speak AdminMessage to the local node. Reusing the surface is cheap and consistent.
- **`NodeDB::saveProto` > new persistence layer.** Existing pattern, atomic writes via `SafeFile`, free factory-reset support.

### Alternatives considered

- **Add fields to `User` / `NodeInfo`.** Rejected: would burn airtime, would broadcast app state to every other node, would require firmware to interpret each app's intent.
- **Add a new portnum + carrier message instead of `AdminMessage`.** Rejected: no benefit; portnums are mesh-visible and AdminMessage is already the right surface for "configure my locally-connected node."
- **Per-app filesystem (`/apps/<app_id>/state.bin`).** Rejected: balloons firmware complexity, opens VFS surface to clients, doesn't compose with existing `NodeDB::saveProto` helpers, hard to factory-reset cleanly.
- **Stronger ownership primitive (each `app_id` bound to a per-app key).** Deferred to a future RFC: would require a new key-management surface in the firmware that doesn't exist today, and would tie this small-feature RFC to a much larger conversation. The v1 proposal explicitly documents the lack of ownership and the resulting client-side discipline.
- **Status-bearing response envelope.** Rejected for v1: would introduce a one-off pattern just for this feature. Existing AdminMessage operations report errors via `Routing_Error_*`, which suffices here.

### Cost of not doing this

Without an explicit storage surface, companion apps remain tempted to repurpose user-visible fields like `long_name` or unrelated configuration surfaces for app-defined convenience metadata. Such workarounds would pollute user-visible state and broadcast app-internal data over LoRa.

## Prior Art

- **`store_ui_config` / `get_ui_config_response` (admin tags 44, 46).** Closest precedent in the same proto: an opaque-ish blob the client tells the firmware to persist locally. No delete operation, no per-key lookup. We extend this pattern to support multiple keyed records.
- **`backup_preferences` (admin tag 24).** Local-only persistence of node config to flash or SD, addressed by enum location. Demonstrates the AdminMessage local-persistence pattern.
- **iOS `NSUserDefaults` / Android `SharedPreferences`.** Mobile-platform precedent for "tiny, app-keyed, per-device kv store with no security boundary." Same shape, same caveats.
- **Web Storage `localStorage`.** Per-origin namespacing without strong ownership across origins on the same machine. Same trust model as proposed here (a privileged caller can stomp anyone's slot).

## Unresolved Questions

These are listed in the proto comment and in the implementation PR body as `TODO(maintainer):` items.

- Are field numbers **104–107** acceptable, or should they be allocated from a different range?
- Should remote authenticated admin be allowed to **read** `client_app_data`, or should all operations be local-only?
- Should there ever be **per-client / per-app ownership** or write protection? The firmware has no per-app identity primitive today (`session_passkey` is a node-minted nonce; `admin_key[3]` identifies operators, not apps). Adding one is out of scope for v1, but would change the trust model substantially.
- Is **4 records × 512 bytes** = 2.3 KB peak storage acceptable across all constrained targets (STM32WL, low-flash nRF52 variants)?
- Should there be a **compile-out flag** (`MESHTASTIC_EXCLUDE_CLIENT_APP_DATA`) for constrained builds?
- Should **`delete` of a missing `app_id`** return success or error? Current implementation treats it as idempotent success, matching `remove_by_nodenum` / `remove_favorite_node` / `delete_file_request`.
- Is the **empty-`app_id` get-miss sentinel** acceptable, or do maintainers want to wait for a future broader AdminMessage status-envelope model?
- Should **`updated_at`** be kept (firmware-set), omitted, or app-provided? Current proposal: firmware-set, may be 0 when RTC is unavailable.
- Should **`app_id` naming** be convention-only (current proposal) or registry-based (e.g. a curated list in this repo's README or in a separate spec page)?
- Should `LocalClientAppData` be included in **`backup_preferences`** / **`restore_preferences`**, or kept out of the backup blob (current default: out, so the store survives reboot but not backup-restore)?
- Should the firmware ever **emit a `ClientNotification`** when an `app_id` is overwritten by another caller, so apps can detect stomping in real time? Current proposal: no, since it adds noise and surfaces a problem clients should already be defensive about.

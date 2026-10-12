# Technical design: persistent UHK80 USB Left in Agent

Status: proposed implementation design; documentation only. 2026-10-12.

This design implements the approved [USB Left PRD](uhk80-left-usb-host-prd.md).
The PRD is authoritative for scope and acceptance; its reviewed SHA256 is
`98809853754272d21fa51771e97ff427372df16314f9128b3f0d0c22214b3617`.
Source was read from the Agent checkout at the independently verified revision
`c17265f5ee6babe941a8d4ea247cd18a8d6f6e4b`. Recheck the revision and applicable
guidance before implementation.
All identifiers introduced below are **proposed**, unless explicitly identified
as existing. No Agent build, automated test, configuration write or hardware
exercise was performed to produce this document.

## 1. Decisions and boundaries

Expose one Add USB Left control and config-only removal in the existing
Connections screen. Store the existing `HostConnections.UsbLeft` value, 2, in one
of the existing 22 positional slots. USB Right remains type 1. The serializer,
configuration version and `ConnectionsAction` opcodes remain unchanged.

Check the exact provisional firmware version `18.0.65535` on both UHK80 halves
only when creating a left entry. Each actual Add performs a fresh, correlated
read through Electron's existing device queue. A cached poll result can inform
the screen but cannot authorize an addition. Once created, the entry is an
ordinary configuration record: there is no creation provenance, pending-addition
save gate, storage baseline, or relay-specific rule in save/import/restore/update.

Keep right-half configuration ownership and the current discovery filters.
Configuration edits never select a host, modify bindings or send input. The gate
establishes compiled support for setup; firmware owns negotiation, availability,
switching and release of held input. No connected/active/ready indicator is
derived from the support result.

The implementation uses a small feature helper, typed IPC messages and existing
reducers/effects. It adds no firmware command, background setup service, global
capability registry or general poller redesign. Firmware naming reconciliation
and transport cleanup belong to [firmware issue #11](https://github.com/leadZERO/UHK-Firmware/issues/11).
Historical handoff instructions to rename macros or apply names across products
do not override the PRD.

## 2. Current implementation and proposed ownership

| Location | Verified behavior and intended change |
| --- | --- |
| [Host model](../packages/uhk-common/src/config-serializer/config-items/host-connection.ts) and [configuration](../packages/uhk-common/src/config-serializer/config-items/user-configuration.ts) | `UsbRight = 1`, `UsbLeft = 2`, `HOST_CONNECTION_COUNT_MAX = 22`; address serialization is restricted to BLE/dongle. `index` is runtime metadata assigned from array position on load. Keep encoding and `defaultHostConnections()` unchanged. Add round-trip tests. |
| `packages/uhk-common/src/util/is-firmware-support-usb-left-relay.ts` (new) | Export `UHK80_USB_LEFT_RELAY_FIRMWARE_VERSION = '18.0.65535'` and a small exact-version predicate. Do not alter generated `VERSIONS` or bundled firmware metadata. |
| [Common IPC names](../packages/uhk-common/src/util/ipcEvents.ts), common models and exports | Add only feature read/reply/invalidation message names and DTOs described below. No configuration fields are added. |
| [Electron device service](../packages/uhk-agent/src/services/device.service.ts) | Reuse `getHardwareModules(false)`, `stopPollUhkDevice()`, `startPollUhkDevice()` and `QueueManager`. Add the correlated read handler and narrowly scoped invalidation hooks. |
| [USB operations](../packages/uhk-usb/src/uhk-operations.ts), [HID device](../packages/uhk-usb/src/uhk-hid-device.ts) | Reuse `getHardwareConfiguration()`, `getDeviceConnectionStateAsync()` and existing version reads. No USB protocol change is required. Read failures already throw and close the HID handle. |
| [Renderer bridge](../packages/uhk-web/src/app/services/device-renderer.service.ts) | Send the typed feature request and dispatch the matching reply/invalidation actions. Do not infer support from `hardwareModulesLoaded`, whose existing payload has no request or connection identity. |
| [Device actions](../packages/uhk-web/src/app/store/actions/device.ts), [device reducer](../packages/uhk-web/src/app/store/reducers/device.ts), [device effects](../packages/uhk-web/src/app/store/effects/device.ts), [selectors](../packages/uhk-web/src/app/store/index.ts) | Hold transient support/read state and drive refresh, request deadlines and cancellation. Keep the feature effects in the existing device effect class. |
| `packages/uhk-web/src/app/util/usb-left-host.ts` (new) | Pure support interpretation, slot planning and mutation-result helpers used by production selectors/reducers/effects. Web-only state/UI logic stays in `uhk-web`. Tests exercise these actual helpers. |
| [User-config actions](../packages/uhk-web/src/app/store/actions/user-config.ts), [reducer](../packages/uhk-web/src/app/store/reducers/user-configuration.ts), [local-edit effect](../packages/uhk-web/src/app/store/effects/user-config.ts) | Add atomic config-only commands, a transient operation result and explicit `saveUserConfig$` registration. No paired-device deletion or auto-save actions are reused. |
| [Connections component](../packages/uhk-web/src/app/components/device/host-connections/host-connections.component.ts), [template](../packages/uhk-web/src/app/components/device/host-connections/host-connections.component.html), [styles](../packages/uhk-web/src/app/components/device/host-connections/host-connections.component.scss) | Add support explanation, Add/name form, inline outcome, row focus and USB-only confirmation. Continue using the table, fold behavior, rename, drag/drop and switchover controls. |
| [Defaults generator](../packages/uhk-common/scripts/generate-user-configs.ts), [source defaults](../packages/uhk-common/user-config-80.json) | Make new UHK80 USB Right default `USB Right`, explicitly retain UHK60's existing generated defaults. Do not migrate names in existing configurations. |
| [Device key editor](../packages/uhk-web/src/app/components/popover/tab/device/device-tab.component.ts), [connection action](../packages/uhk-common/src/config-serializer/config-items/key-action/connections-action.ts) | Reuse current slot-based choices; regression coverage is expected, new selection semantics are not. |

The [paired-device effect](../packages/uhk-web/src/app/store/effects/dongle-pairing.effect.ts)
calls `deleteHostConnection` and dispatches `SaveConfigurationAction(true)` on
success. Its Electron counterpart deletes bonds. USB removal must not enter that
pipeline, even with an empty address.

```mermaid
sequenceDiagram
    participant UI as Connections
    participant E as Device effects
    participant D as DeviceService queue
    participant R as User-config reducer
    participant L as Existing local-edit effect
    UI->>E: Add intent and candidate name
    E->>D: Correlated support read
    D->>D: Verify owner, read both versions, recheck owner
    D-->>E: Versions, identity and generation
    E->>E: Reject expired or superseded context
    E->>R: AddUsbLeftHost (same synchronous turn)
    R->>R: Validate latest slots/name; mutate once or no-op
    R-->>L: Accepted added result
    L->>L: Recalculate, save local edit, show Save
    Note over UI,L: Physical save occurs only through the ordinary user save flow
```

## 3. Support data and bounded freshness

### 3.1 Existing data is sufficient for versions, not request identity

`getHardwareModules(false)` reads right-side `firmwareVersion` and inserts a
`ModuleInfo` for `LEFT_HALF_MODULE`; its `info` is empty when the half is absent.
Locate the peer by `module.id === LEFT_HALF_MODULE.id` (and its known slot), never
by array offset or an attached key cluster's version. Require exactly one matching
half record with a usable version.

The general poller runs every 250 ms but fetches module versions principally when
the observed connection state changes. Its catch branch logs an error and can
leave the prior renderer snapshot in place. `connectedDevice.id` identifies a
product, not a physical keyboard. Therefore existing module state alone cannot
prove freshness or bind an Add request to the configuration being edited.

Introduce these feature-only DTOs in common models; the concrete union should use
the project's action-class and IPC naming conventions:

```ts
type UsbLeftSupportRequest = {
    requestId: string;
    contextId: number;             // renderer's current operation context
    expectedDeviceId: number;      // UHK80_RIGHT only on the desktop path
    expectedUniqueId: number;      // from the loaded hardware configuration
    expectedGeneration?: number;   // last known feature generation in Electron
};

type UsbLeftSupportReply = {
    requestId: string;
    contextId: number;
    generation: number;
} & (
    { success: true; deviceId: number; uniqueId: number;
      modules: HardwareModules; leftHalfConnected: boolean;
      observedPendingPairings: NewPairedDevice[] }
    | { success: false; reason: 'no-device' | 'device-changed' |
        'peer-unverified' | 'read-failed' | 'expired'; message?: string }
);
```

Use `UHK_DEVICE_IDS_TYPE` for product IDs in actual code. A third IPC event,
`usbLeftSupportInvalidated`, carries the new generation and a reason. All of these
are application messages over the existing bridge, not new firmware properties.
An uncorrelated `ModulesInfoLoaded` action may invalidate contradictory data but
must never authorize an Add or overwrite a correlated support result.

The named version predicate uses strict equality to the canonical version string.
Missing/malformed versions return unverified; a valid different version is
unsupported. Do not use `>=`, and do not accept suffixes or extra components via
the existing permissive version parser. `18.0.4`, `18.0.65534`, `18.1.0` and
`19.0.0` all fail the development gate. Device model, both halves and current
identity are additional conditions, not part of this one-version predicate.

### 3.2 Read handler and identity

Add `readUsbLeftSupport`/`readUsbLeftSupportReply` handlers following the existing
`ipcMain.on(... queueManager.add({ asynchronous: true, ... }))` pattern. The
request reads no user configuration and never changes renderer configuration.

1. Capture a main-process monotonic deadline when the IPC request is received,
   before queuing. At execution, reject requests beyond that deadline or whose
   expected feature generation has been superseded.
2. Stop and await the existing poller. Read current connection state; require
   permission, a single ordinary UHK80 right configuration interface, no bootloader,
   and no update/recovery in progress. Left USB enumeration is irrelevant.
3. Read `operations.getHardwareConfiguration()` and compare both `deviceId` and
   `uniqueId` with the request. A numeric ID of zero is not rejected merely for
   being falsy; absent identity is unverified. Product identity alone is insufficient.
4. Capture the current feature generation. Call `getHardwareModules(false)`.
   Read current connection/half state and hardware configuration again. Require
   the same owner before/after, the same generation, an available right interface
   and a still-connected peer for a supported result. The two hardware reads reuse
   the existing hardware-config command and do not read or overwrite user edits.
   Preserve pending-pairing observations from both connection reads in the
   reply, since the normal poller was paused; they reserve slots during this Add.
5. If any awaited operation throws, the identity changes, the peer disappears,
   or the deadline expires, return a failure and invalidate previous feature
   support. Never merge partial new versions with an old successful snapshot.
6. In `finally`, close the query handle using the existing service convention and
   restore polling if this handler paused it. Send at most one terminal reply.
   Guard replies to destroyed renderer windows and catch cleanup/send failures.
   **The queued method must always resolve:** `QueueManager.process()` has no
   enclosing error recovery, so allowing this handler to reject could stall later
   saves/queries. This work does not repair QueueManager itself.

The ordinary USB write implementation throws and closes its handle on failed
transfer; a failed command cannot be retried internally as a successful support
snapshot for a different device. The before/after owner checks also reject a
same-model keyboard replacement that the ordinary connection-state equality
comparison misses. Do not cache a successful Add authorization across requests.

Maintain `usbLeftSupportGeneration` in DeviceService, independent of the normal
`savedState`. Bump it and emit a feature invalidation when observed right
availability/permission/multi-device state changes, the left connection state
changes, either firmware version changes, the identified owner changes, a feature
read fails, or the poller's connection/module query fails. Add a small hook in
the existing poller's state-change and catch branches; retain its cadence, normal
state payloads, navigation and version-refresh policy. Emit once per transition
to invalid rather than on every repeated error. Do not invalidate for active
keymap/status-buffer changes alone. Firmware update/recovery start and completion
(including failure) invalidate through their existing lifecycle paths.

This is observed identity and freshness, not an instantaneous physical guarantee.
No finite set of reads can prove that a cable remains attached after the final
response. Add creates a local configuration record; A12's runtime safety evidence
is consequently required even with a correct gate.

### 3.3 Renderer state, deadlines and refresh policy

Keep one transient `usbLeftSupport` field in device state:

- `status`: `unverified`, `checking`, `supported`, `unsupported` or `demo`;
- reason plus the two reported versions, current owner/generation and request ID;
- at most one pending request and its purpose (`display` or `add`).

Use `USB_LEFT_SUPPORT_READ_DEADLINE_MS = 10_000` as the proposed per-request
deadline. This is setup policy, not relay transport timing. Confirm it on paired
hardware; raising the timeout must preserve late-response rejection semantics.

On Connections entry, request once when a loaded UHK80 context and live desktop
configuration device are available. Refresh on an explicit **Check firmware**
action or on a stable reconnect while that screen is active. Keep the display
result until a known invalidation or the next read; it is the last checked pair,
not an authorization to mutate. Do not add a display TTL, periodic refresh or
automatic retry loop. Cancel route-scoped request timers/subscriptions on
component destruction.

Every actual Add uses its own new request. Start the 10-second renderer deadline
before sending IPC, including queue wait. If it expires, stop showing a spinner,
mark unverified, retain all edits and
ignore a later reply. The main process also checks its local deadline before and
between reads. Do not use `Promise.race` to release the device queue while a HID
operation is still running: the underlying operation must settle before cleanup
and queue advancement. Existing 1-second HID read timeouts apply; this feature
does not promise to repair a general hung native driver or poller.

Use request-ID matching in addition to an RxJS `exhaustMap` or equivalent
single-flight rule. Disabling the button alone is not the double-click defense.
Cancellation discards the pending command; it does not imply cancellation of an
already-issued native HID read. One read may finish in the background and its
reply be discarded. Do not accumulate retries behind it; the renderer keeps one
transport request outstanding until its terminal response or connection teardown.

Clear eligibility immediately on observed disconnect, replacement, relevant
module change, feature/poller read failure, update/recovery, changing demo model,
or replacement of the editable configuration. Older replies cannot overwrite a
newer failure/invalidation. The next Add always rechecks physical owner identity.

## 4. Editable context and asynchronous Add lifecycle

Maintain a small, **transient** `usbLeftEditContext` in user-config reducer state:
a monotonically increasing context ID, the product owning the loaded
editable configuration, and its hardware unique ID when known. Establish it on
successful device configuration load, or the established demo model/default load.
Keep the product with the editable configuration across disconnect so config-only
removal can use the correct local-storage key. Clear physical Add eligibility
on disconnect. Do not derive editable ownership from a newly connected model
until its configuration has loaded.

This context contains no saved configuration, dirty baseline or record of how a
host entry was created. It exists only to reject obsolete UI operations and
choose the existing local persistence bucket. On desktop startup with only local
data and no known hardware owner, Add remains unverified. Browser model selection
uses its explicit demo context.

Increment/cancel this operation context at the existing configuration replacement
boundaries: device/model change, `LoadConfigFromDevice`/load completion,
`LoadUserConfigSuccess`, `PreviewUserConfiguration`,
`ApplyUserConfigurationFromFile`, `LoadResetUserConfiguration`, history/shared
configuration application and undo that replaces the whole configuration. Audit
their existing action chains; where they already converge on the listed load
actions, handle the convergence once. Ordinary rename, key edits and slot reorder
do not cancel Add: they are preserved and revalidated against current state.
`SaveUserConfigSuccess` is an ordinary local-edit completion, not a replacement.

Proposed actions in the current action files:

| Action | Responsibility |
| --- | --- |
| `RefreshUsbLeftSupport`, `UsbLeftSupportReply`, `InvalidateUsbLeftSupport` | Feature query, request timeout and display state; no configuration mutation. |
| `RequestAddUsbLeftHost({ requestId, contextId, name })` | User intent; starts a new read only after synchronous preflight. No list/index snapshot is retained for later insertion. |
| `CancelUsbLeftHostOperation` | Abandons a pending Add/name interaction on cancel, route exit, configuration replacement, removal or invalidation. |
| `AddUsbLeftHost({ requestId, contextId, name, observedPendingPairings })` | Internal config-only command dispatched after the accepted fresh read. Reducer makes the final slot/name decision against its current state and conservatively retains any slot reservations observed by that read. |
| `RemoveUsbLeftHost({ requestId, contextId, index, expectedEntry })` | Config-only command after confirmation. `expectedEntry` is an in-memory row reference for stale-confirmation rejection, not serialized identity. |

Use existing action classes and the corresponding `Actions` unions. Effect code
samples the latest context/device/update state before dispatching Add and does
not `await`, defer or schedule between that check and dispatch. The reducer
checks context ID and reruns list validation at mutation time; an effect-sampled
array is never assigned wholesale over newer edits. Device effects/selectors read
the context from user-config state; do not maintain a second counter. One helper
defines which actions supersede the pending feature operation.

The Add sequence is:

1. Inspect the latest list. If it contains one left entry, reveal/focus that row
   and report its current name without mutation or support read. If there are
   duplicates, report their one-based slots and leave the list unchanged. Reject
   oversized/full/blocked-slot cases before doing I/O.
2. Use the candidate `USB Left` initially. If another entry uses that exact name,
   show an inline required name field and **Add**/**Cancel** controls. Do not
   insert a temporary row or rename the conflicting entry. Confirming a unique
   candidate starts a new support read. The user may also cancel during checking.
3. Freeze the submitted candidate in the pending command, show Checking firmware,
   and disable additional Add submissions. All other configuration edits remain
   available. Changing the candidate cancels that command; a new submission
   creates a new request, subject to the one-outstanding-read rule.
4. Accept a reply only if its request/context/generation and owner match the
   current pending intent and editable owner, its deadline has not expired,
   physical eligibility is still valid, and both versions pass the exact gate.
5. Dispatch `AddUsbLeftHost` synchronously. The reducer validates the latest list
   as described below and records one transient result. A name conflict that arose
   during the read returns to the name form without mutation; confirming another
   candidate starts a fresh read. A unique left entry introduced in the meantime
   is reused. No stale free-slot index survives the query.
6. On an `added` result, focus the new row and let `saveUserConfig$` handle the
   edit. On `existing`, focus the row only. On any refusal, retain current edits
   and display the actionable reason. Clear pending request state after either
   outcome; the stored entry carries no authorization metadata.

| Concurrent event | Required result |
| --- | --- |
| Double click or duplicated Add action | One in-flight read and at most one insertion. Repeated setup after insertion is a no-op. |
| User edits unrelated keys/settings or reorders hosts during the read | Commit into the first eligible slot in the latest list; preserve those edits and all key-action IDs. |
| Another action fills the planned empty slot, reserves a paired slot, creates a name conflict or adds a left entry | Recompute the entire local Add decision at commit; use a different eligible slot, report conflict, or reuse one entry as appropriate. |
| Device swap/disconnect, peer/version change, failed read or updater/recovery start | Cancel Add, invalidate support and ignore the old reply. Never mutate a newly loaded keyboard's configuration. |
| Import/restore/history/shared config/reset, demo-model change or navigation away | Cancel old intent. The replacement configuration follows its ordinary behavior; no deferred insertion into it. |
| User confirms removal while an Add is pending | Cancel pending Add before the remove command; a late reply cannot recreate the removed entry. |
| A removal confirmation is open while its row is renamed, reordered or replaced | Reject the stale confirmation and ask the user to confirm the current row. Do not locate a substitute by name. |
| Peer disappears after Add has committed | Keep the edit and normal Save behavior. No retroactive removal or new save restriction. |

## 5. Atomic configuration operations

### 5.1 Add helper and reducer

The pure helper receives the latest `hostConnections`, the requested name and
the currently known reserved slots from `state.newPairedDevices`, combined with
`observedPendingPairings` from the accepted support read. These observations are
runtime pairing data; do not consume, modify or serialize them. Any populated
BLE/dongle record is already occupied regardless of its current bond status.
Known `newPairedDevices[].slot` values reserve those empty positions too. If an
unconfigured bond lacks a slot and its safe placement cannot be identified, return
`pairings-pending` and direct the user to complete the existing paired-device flow
before adding; do not guess which slot is safe or change that flow.
Retaining an observation for the duration of this one Add may conservatively
refuse a slot that has just been freed; retry reads current state. It must not
overwrite the normal pairing state or create a persistent reservation system.

Return a discriminated result:
`added(index, connections)`, `existing(index)`, `duplicates(indices)`, `oversized`,
`full`, `pairings-pending`, `name-conflict`, `invalid-name` or `stale-context`.

Apply checks in this deterministic order:

1. Verify the current editable product is UHK80 and the operation context matches.
   Other products cannot create a left entry through this action.
2. Count left entries. More than one returns `duplicates`; exactly one returns
   `existing` without changing its name, index or switchover. If the list is also
   oversized, show that existing configuration warning separately; reuse never
   writes or repairs it.
3. With no left entry, reject `length > HOST_CONNECTION_COUNT_MAX`. Do not truncate.
4. Validate the new name as a nonblank single-line value. Preserve its submitted
   characters; do not silently trim or normalize it. Conflict comparison uses
   exact case-sensitive equality, matching the current rename reducer, across
   existing entries. This validation applies to new names only; do not rewrite
   previously saved names or introduce global import validation.
5. Scan positional IDs 0 through 21. An existing record is eligible only when its
   type is `Empty` and the slot is not reserved by pending pairing information.
   For a short valid list, missing trailing positions are free candidates. Never
   treat an occupied/reserved record as free because its name is blank. If none
   is eligible, return `full` or `pairings-pending` with the correct explanation.
6. Copy the array. For a short list, materialize only the missing positions needed
   to reach the chosen slot, using indexed `emptyHostConnection()` instances.
   This bounded extension cannot exceed 22. Do not pad/reindex or replace other
   existing records. Create a fresh `HostConnection` with `type = UsbLeft`,
   `index = chosenIndex`, the accepted name and `switchover = false`; leave
   `address` undefined. Assign only that slot.

The reducer returns a shallow `UserConfiguration` copy, following nearby code,
and the new array only for `added`; all unrelated members and existing host
records retain their values and identities. Never clone through JSON, which would
drop runtime fields. Do not modify keymaps, macros, paired-device state or any
`ConnectionsAction.hostConnectionId`.

Store `lastUsbLeftEdit = { requestId, outcome, index? }` outside UserConfiguration
as an ephemeral action result. This permits the effect/UI to distinguish an
accepted mutation from an idempotent no-op or refusal. Clear it when the context
is replaced. It is not persisted, is not per-entry provenance, and is never read
by physical save/import/update logic.

### 5.2 Removal

Render a separate **Remove USB Left** control only for a USB Left row in the
UHK80 editing context. Use the existing `mwlConfirmationPopover` style and include
the one-based slot and configured name. Explain: “Keys assigned to this slot will
still select the now-empty slot. Save to Keyboard to keep this change.”

Capture `expectedEntry` when the confirmation is opened. At confirmation dispatch,
first cancel pending Add; then validate the context, integer array index and
`current[index] === expectedEntry && current[index].type === UsbLeft`. If any
condition fails, return a stale-row result without mutation. An already empty
slot is an idempotent no-op. Explicit removal of a displayed duplicate left row
is permitted; choosing which duplicate to remove belongs to the user.

Replace only that slot with `emptyHostConnection()` and explicitly set its
`index` to the same position. Do not splice, compact, reorder, change bindings,
delete USB Right, call `DeleteHostConnectionAction`, unpair any address or
dispatch `SaveConfigurationAction`. Removal does not read firmware support and
works with old/mixed/absent relay firmware and with retained local UHK80 edits.
No new oversized-list repair workflow is introduced.

## 6. Local persistence, physical save and failure behavior

Add `ActionTypes.AddUsbLeftHost` and `ActionTypes.RemoveUsbLeftHost` explicitly to
the `ofType` list in `UserConfigEffects.saveUserConfig$`. For these two actions,
check the post-reducer `lastUsbLeftEdit.requestId` and require `added` or `removed`
before persistence. Other action handling stays as it is. This filter prevents
refusals/reuse/stale commands from generating Save visibility or overwriting
local storage.

Use the latest reducer configuration, recalculate its length, and call the
existing `DataStorageRepositoryService.saveConfig`. Its current implementation
synchronously calls `localStorage.setItem` then returns `of(null)`; it is not a
physical keyboard write. For the new USB actions, use the product attached to
the editable context. This matters after disconnect: `app.hardwareConfig` is
cleared and `getConnectedDevice()` can be absent, while `saveConfig` dereferences
the product ID to choose `config-uhk-80` or `config`. Do not silently save an
offline UHK80 edit into the UHK60 key. No local-storage schema/key changes are
needed.

On success, keep the established sequence: `SaveUserConfigSuccessAction(config)`,
`DismissUndoNotificationAction` and `ShowSaveToKeyboardButtonAction`. The first
action updates the existing local-edit reference in app state; it must not be
used to restore an older full configuration over subsequent edits. USB removal
uses confirmation and does not add a new undo system.

Handle synchronous local-storage errors for these USB actions inside the effect
using `defer` plus a scoped `catchError`, or equivalent try/catch. Retain the
in-memory edit, show the ordinary Save affordance subject to existing device
restrictions, and report “The change is in memory, but Agent could not save it
locally. Export a backup or save it to the keyboard before closing Agent.” Do not
emit local-save success or allow the error to terminate the editing effect.
Do not broaden this branch into a rewrite of all existing edit-error handling.

Physical saving remains `DeviceEffects.saveConfiguration$` and its existing
schema/capacity/version checks. `SaveConfigurationReplyAction` success/failure
retains its current meaning. A failed keyboard write retains edits and reports
failure; do not announce persistence from an Add result or local-storage success.
Save/reload/readback and a power cycle are acceptance tests, not a new automatic
readback procedure that might replace pending edits. Uncertain partial writes
and general save-race repairs remain separate work.

Explicitly test that Add-created and imported identical configurations take the
same save path after a peer loss or Agent restart. A disconnected device can
still make the ordinary physical save unavailable; relay eligibility adds no
further restriction. Export, history, restore, shared-config application,
firmware updater/recovery and their existing write ordering remain unchanged.

## 7. Connections UI and browser demo

Add a compact UHK80 setup block above the table. The introductory text is the
PRD's “Connect a second computer to the left USB port. Use a connection key to
choose which computer receives input.” Use normal buttons, inline name input and
an `aria-live="polite"` outcome. Preserve focus when checking fails; on a successful
add/reuse, focus the row/name field with a stable `data-index` reference.

The existing suffix fold naturally reveals a newly occupied slot because
`countTrailingEmptySlots` changes. Keep the array and drag/drop model intact; do
not render a filtered array or automatically reorder. A created row at slot 21
shows UI slot 22. Empty interior slots stay visible. A unique existing left entry
shows its saved name and existing rename/switchover controls even when support is
unavailable. Removal remains usable independently of Add.

| State | Presentation |
| --- | --- |
| UHK80, no left entry, last checked pair supported | Add USB Left; clicking performs a fresh read and creates only after success. |
| Check in progress | Checking firmware, one spinner; Add disabled and Cancel available. Unrelated edits remain available. |
| Supported right, absent/unknown left | “Connect the left half by bridge or its paired inter-half Bluetooth link, then check firmware.” No requirement to plug left USB into the Agent computer. |
| Explicitly unsupported half | Explain that both halves require the compatible experimental pair; show the reported versions and Check firmware. Do not offer an automatic update to the provisional version. |
| Failed/expired read | “Could not verify both halves. Check the connection and try again.” One inline error and manual retry; old success is cleared. |
| No live right configuration device | Explain the right-half setup connection; Add unverified. Existing viewing/edit/export behavior remains. |
| No eligible slot/pending paired slots | Explain capacity or completing pending pairing; never replace an occupied row. |
| Multiple left entries | Show their slot numbers and explicit removal controls; explain that Add cannot choose a duplicate to keep. |
| Name conflict | Inline name form before mutation; preserve the conflicting host and all current edits. |
| Unique left entry | Reveal its current row/name; repeated setup has no save side effect. |
| Browser UHK80 demo | Mark “Demo: simulated firmware support”; same Add/remove helpers and local edit behavior, zero physical calls. |
| UHK60 or other model | No USB Left creation controls and no generated-default changes for that model. |

For browser mode, create an explicit web-only UHK80 fixture containing a right
version and a `LEFT_HALF_MODULE` version set to the named provisional value.
Select it only when `runningInElectron === false` and the selected model is
`UHK_80_DEVICE`. Use the normal feature evaluator but mark the origin `demo` and
short-circuit IPC. Keep the existing general synthetic `getHardwareModules`
selector and generated firmware metadata unchanged. Switching to UHK60 cancels
pending operations. A mocked support fixture never enters the physical evaluator.

The device key editor already builds choices using array position and the
current name, plus Next, Previous and Last. Verify the new row appears there;
do not introduce a second host model, a name-based key binding or a `nextActive`
key opcode. Help may use the literal example `switchHost 'USB Left'`, one-based
numeric slot examples and `switchHost nextActive`. Do not generate macro text
from arbitrary custom names in this feature.

Firmware parser source currently treats single-quoted strings as literal, so a
backslash inside them is not an apostrophe escape. Double-quoted strings process
backslash escapes and `$` interpolation. A7 must exercise custom names containing
spaces, both quote types, backslashes and `$` using parser-supported forms, with
numeric slot selection as the existing alternative. If future UI adds a macro
copy button, it must use parser-verified escaping (including `$`), not JSON or
JavaScript string escaping by assumption. Preserve all user-authored macro text.

## 8. Product-specific generated defaults

Change the new-config USB Right name in `packages/uhk-common/user-config-80.json`
from `My PC` to `USB Right`. Do not add a USB Left record to default JSON: naming
does not enable support and Add remains gated.

In `generate-user-configs.ts`, make the products explicit before writing outputs:
use the source as the UHK80 defaults; create the UHK60 model from it as today,
then apply the **UHK60 generation-only** right-host name `My PC` before emitting
`user-config.json` and `src/user-config-60.ts`. Keep existing UHK60 layer/key-count
conversion intact. Target only the known default right-host record; fail the
generator's focused assertion if the source structure no longer matches its
single default right record, rather than renaming arbitrary user data.

Keep `defaultHostConnections()` returning its current `My PC` legacy migration
default. Never run the generation transform on loaded/imported/restored configs.
The Add helper alone supplies the newly created left name `USB Left`.
Factory-default creation/reset naturally consumes the new product default;
ordinary loading and backup restoration preserve every configured host name.

The common package's `user-config.json`, `src/user-config-60.ts` and
`src/user-config-80.ts` are generated and ignored in the inspected checkout.
Regenerate for validation; commit only source/generator/test changes according
to the repository's current tracking rules. Capture the pre-change generated
UHK60 outputs and compare the full results after regeneration, not just the
right-host label.

## 9. Validation plan and traceability

These are implementation acceptance requirements, not completed results.
Use Node's existing test runner with nearby `.test.ts` conventions and `.js`
relative imports. Exercise production helpers and, where practical, reducer/effect
boundaries; do not substitute a test-only reimplementation of the gate or slot
algorithm. Keep UI logic in web code even when extracting a small pure helper
makes it testable.

| Focused test group | Required cases and observations | PRD coverage |
| --- | --- | --- |
| `uhk-common` firmware predicate tests | Exact marker; absent/malformed version; old, disabled, numerically higher and suffix-bearing strings. Wrapper tests require UHK80 right plus exactly one left-half record, not an accessory. | A5 |
| Host/config serialization tests | JSON and binary round trips for types 1/2; names, switchover and all 22 slots; no USB address field; explicit runtime index reconstructed; direct-slot IDs 0 and 21 unchanged; unrelated settings/macros retained. | CFG-1, CFG-5, CFG-6; A1, A7, A12 |
| Web Add helper/reducer tests | No left, one custom left, two lefts, first/last free slot, short list, 22 full, oversized with/without left, pending paired reservation, unknown reservation, name conflict introduced before commit, blank name, no mutation on refusal, double dispatch/idempotence; no compaction or index rebinding. Assert pre-existing objects/runtime data preserved. | CFG-2, CFG-4, CFG-5, CFG-6; A2 |
| Removal/reorder tests | Remove only the confirmed left slot; stale confirmation after rename/reorder/config replacement; repeated remove; user chooses a duplicate; right/BLE/dongle and bindings preserved. Assert zero unpair calls and zero immediate `SaveConfigurationAction` emissions. | CFG-5, CFG-7; A3 |
| Asynchronous support/Add tests | Queue delay/deadline; matching/mismatched request and owner; same-model device swap; disconnect/reconnect; stale success after failure; missing peer; peer lost during query; changed firmware; updater/recovery; late/out-of-order response; double click; name/list/key edits while pending; import/history/model replacement; removal cancellation. Use a controllable clock and deferred read responses. | CFG-2, CFG-7; A2, A5, A13 |
| Local-edit/save tests | Both new commands are registered; only accepted mutations recalculate/persist/show Save; full config contains latest unrelated edits; UHK80 local key retained offline; local storage throw retains memory and effect subscription; no immediate physical write; ordinary keyboard failure retains edit; identical imported/added entries follow the same save rules without peer. | CFG-7, CFG-8; A5, A6, A12 |
| Defaults tests | Pre/post complete UHK60 generated-output equality; UHK80 right default only; no default left; legacy `defaultHostConnections()` unchanged; custom/old names survive current import/restore/round trips. | CFG-3; A4 |
| Browser/component checks | UHK80 explicit mocked support, UHK60 negative control, zero desktop IPC; folded suffix and slot 22 revealed, focus/accessible status, unique-name flow, conflict/duplicate/full/unsupported explanations, removal confirmation, existing key editor choices. | A2, A7, A10 |
| Desktop integration checks | Real IPC queue/handler continues after failed read; normal save and updater/recovery still run; both cables on one computer configure right; left-only stays on existing missing-device flow; stale support cleared on resulting firmware. | A5, A10, A11, A13 |

Add Node tests to the actual common/web `test` globs so root `npm test` runs them.
Extract only small feature-specific pure orchestration functions if needed to
drive RxJS streams without bootstrapping Angular. For effect wiring, use a focused
Node/Angular injection-context harness or an explicit desktop integration check;
name that boundary in the evidence. Pure-helper tests alone do not prove that an
action was registered in `saveUserConfig$` or that Electron releases its queue.

The current `uhk-agent` package has no test script and `uhk-usb` has only
`test:skip`; root CI does not thereby test those IPC/HID paths. Keep them as named
desktop integration obligations unless a small focused harness is added. A
repository-wide Angular/Electron test framework migration is outside scope.
Practical negative controls include an intentionally old peer version, a stale
request ID and a spy that fails if Bluetooth unpair or automatic keyboard save is
called by the new USB controls.

Follow the existing [.github/workflows/ci.yml](../.github/workflows/ci.yml) order:

```text
npm run install
npm run build
npm run lint
npm run build:web
npm test
```

Build precedes lint because generated files are required. The inspected engines
are Node `>=24.18.1 <25.0.0` and npm `>=11.13.0 <12.0.0`; recheck `.nvmrc`, package
engines and the workflow at implementation time. Use the existing OS matrix and
release job; the new tests join it without changing signing/release behavior.
Report pre-existing failures separately, including unexercised IPC boundaries.

### 9.1 Physical and compatibility evidence

| Acceptance stage | Required evidence before acceptance |
| --- | --- |
| Persistence | Final paired relay images with `CONFIG_UHK_USB_LEFT_DEV_SLOT` disabled; Add through Agent, ordinary save, readback, reload and power cycle keep one entry at the same slot; removal persists. Save-failure case keeps edits. (A6) |
| Unsupported/mixed firmware safety | Under each supported-schema older, relay-disabled and mixed pair, retain/load/import/restore/export/save the same valid type-2 entry, reboot, exercise unrelated hosts and direct/cycling selection. Check no misrouting, stuck keys/modifiers/buttons or loss of configuration access. Include both mixed-half directions. Equivalent entries from Add/import have identical save behavior. This is not established by enum/serialization tests. (A12) |
| Selection and input | Windows/macOS separate hosts; all halves/modules, keyboard, mouse/scroll, consumer/system controls; direct slots, Next/Previous/Last and name/index/nextActive macros; held input across switch, unavailable left, repeat/release; input works with Agent closed. (A7, A8) |
| Transport/lifecycle | Bridge present/absent/restored, paired inter-half BLE fallback, independent sleep/wake and restart, right-only use, Bluetooth/dongle hosts, both host cables on one computer and left-only attachment. Record Linux coverage or its absence. (A9, A10) |
| Update and trial | Manual paired installation recognized; ordinary update/recovery still usable and support re-evaluated afterward. Agree trial duration/active scenarios first, then log active days/hours, failures, fixes and retesting on identified final candidates; obtain tester acceptance before proposing upstream. (A13, A14) |

Every record identifies the Agent commit/build, both image hashes, relay/dev-slot
flags, OS and cable path, bridge/BLE state, expected result, actual result and
pass/fail/untested status. Host tests cannot establish physical isolation, release
latency, power or stack headroom. Firmware [CI #3](https://github.com/leadZERO/UHK-Firmware/issues/3)
owns retained firmware regressions and paired enabled/disabled builds;
[hardware #5](https://github.com/leadZERO/UHK-Firmware/issues/5) owns the hardware
matrix and resource measurements; [rollover #4](https://github.com/leadZERO/UHK-Firmware/issues/4)
remains a separate regression investigation. Reuse the known-working cable path
when the separate baseline enumeration fault blocks a case, and identify it.

If A12 demonstrates an unsafe saved record on compatible-schema firmware, stop
feature acceptance and fix that concrete compatibility defect with firmware
owners. Do not silently add a save-time relay gate, provenance tracking or global
validation system. The required safety evidence is still outstanding.

## 10. Implementation order, risks and release decisions

1. Reconfirm the PRD/source baseline, build environment and contribution guidance.
   Capture UHK60 generated defaults and representative current configurations as
   regression fixtures. Add the version constant/predicate and pure Add/remove
   helper tests; preserve serialization and legacy migrations.
2. Implement feature read IPC, request correlation, owner checks, deadlines and
   invalidation. Test the failed-handler queue continuation and stale-response
   cases before allowing Add to dispatch a mutation.
3. Implement reducer commands and `saveUserConfig$` registration, including no-op
   filtering, offline product key and local error behavior. Verify that the
   paired-device deletion/save effect receives no USB actions.
4. Connect the existing screen, explicit demo fixture and generated defaults.
   Check layout/focus, folded rows and existing key choices. Run the current CI
   commands and targeted desktop checks, recording baseline failures separately.
5. Build and identify the final Agent and paired firmware candidates with the
   development slot disabled. Complete A1-A13 and the planned extended active
   trial A14; targeted checks and normal daily use may proceed alongside each
   other. Both must be complete before upstream submission. Repeat affected
   checks and a documented trial after behavior changes.
6. Review evidence and maintainer feedback before proposing upstream. The normal
   bundled firmware download and custom updater flow remain unchanged.

The main bounded risks are a slow module read exceeding the setup deadline,
mis-correlated replies, a missing configuration-replacement cancellation hook,
omitted local-edit registration, and accidental UHK60 default changes. The named
tests above target each. Extra accessory/version/checksum reads inside
`getHardwareModules(false)` can make the whole support check fail conservatively;
do not accept a partially failed read or start a separate unbounded retry loop.
If measured cost is unsuitable, a separately reviewed small refactor may reuse
the same half-version reads; it must retain the identity/failure contract.

No blocking product question remains for this fork implementation. Outstanding
engineering evidence is the real read latency, IPC/device lifecycle checks,
unsupported/mixed saved-entry safety, final paired builds, physical matrix and
active-use trial. They are acceptance work, not completed validation.

Upstream enablement still requires agreement on the official firmware version,
which release builds enable relay, the supported installation path and whether
the provisional exact-match exception is retired. Coordinate firmware and Agent
through [firmware issue #6](https://github.com/leadZERO/UHK-Firmware/issues/6).
Manual installation of signed development images does not establish an
Agent-uploadable archive; archive manifest/path/checksum validation is separate
release work. Verify the [CLA and contribution process](../CONTRIBUTING.md) before
outreach; this design authorizes no upstream submission or hardware operation.

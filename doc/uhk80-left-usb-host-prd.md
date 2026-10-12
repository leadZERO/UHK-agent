# PRD: Persistent UHK80 USB Left host support in Agent

Status: draft for review; documentation only. Implementation is not included.
Revised: 2026-10-12; Add-only compatibility gate, ordinary saving, and explicit upstream validation requirements.

- Agent baseline: `c17265f5ee6babe941a8d4ea247cd18a8d6f6e4b` on this fork's `master`.
- Firmware baseline: `19b8a24835d36b3090b66b923d4dbad9b721182c` plus the local provisional relay-version change, [firmware PR #2](https://github.com/leadZERO/UHK-Firmware/pull/2).
- Work item: [Agent issue #1](https://github.com/leadZERO/UHK-agent/issues/1).
- Related firmware review work: [contract and cleanup follow-up](https://github.com/leadZERO/UHK-Firmware/issues/11).
- Firmware contract: [Agent integration handoff](https://github.com/leadZERO/UHK-Firmware/blob/19b8a24835d36b3090b66b923d4dbad9b721182c/doc-dev/technical/uhk80-dual-usb-hosts-design.md#agent-integration-handoff); the version contract below includes the local update not yet present at that revision.

## Outcome

A user running compatible experimental firmware on both UHK80 halves can add a
persistent **USB Left** connection in Agent, save it, and select either USB host
using existing connection keys or macros. The entry survives reload and power
cycles without the firmware's temporary development slot. Existing configuration,
custom names, slot bindings and Bluetooth pairings remain intact.

The right half owns keyboard configuration and input routing. Agent configures the
host list; it does not relay keystrokes or need to stay running after setup.
Input reaches the selected computer through right USB or left USB. The firmware
prefers the inter-half bridge and can use paired inter-half Bluetooth when the
bridge is absent. That Bluetooth link is distinct from a Bluetooth computer host.

## Current implementation: verified findings

| Area | Current behavior and implication |
| --- | --- |
| [Host model and serializer](../packages/uhk-common/src/config-serializer/config-items/host-connection.ts) | `UsbRight = 1`, `UsbLeft = 2`, both display labels, and 22 slots already exist. JSON readers support configuration major version 15. USB records already serialize without a Bluetooth address. No new host type or configuration format is needed for persistence. |
| [Defaults generator](../packages/uhk-common/scripts/generate-user-configs.ts) | Both UHK80 and UHK60 defaults derive from `user-config-80.json`, whose right-host name is `My PC`. Changing that source propagates to UHK60 unless the generator distinguishes the products. The separate `defaultHostConnections()` helper also supplies legacy schema migration without product context. |
| [Connections UI](../packages/uhk-web/src/app/components/device/host-connections/host-connections.component.ts) and [template](../packages/uhk-web/src/app/components/device/host-connections/host-connections.component.html) | Existing entries can be renamed, reordered and assigned automatic switchover. Trailing empty slots are folded. There is no USB Left creation action; deletion is shown only for addressed Bluetooth/dongle entries. |
| [Configuration reducer](../packages/uhk-web/src/app/store/reducers/user-configuration.ts) | Reordering intentionally preserves positional key bindings. Deletion empties a slot without compacting the array. Rename rejects duplicate names. New USB setup must preserve these semantics. |
| [Connection key editor](../packages/uhk-web/src/app/components/popover/tab/device/device-tab.component.ts) | Already lists configured slots plus Next, Previous and Last connection actions. [ConnectionsAction](../packages/uhk-common/src/config-serializer/config-items/key-action/connections-action.ts) stores zero-based slot IDs. UI slot labels are one-based. A saved left entry should reuse this path. |
| [Deletion effect](../packages/uhk-web/src/app/store/effects/dongle-pairing.effect.ts) and [Electron device service](../packages/uhk-agent/src/services/device.service.ts) | The existing delete request invokes Bluetooth bond deletion. Making its button visible for USB would be insufficient and incorrect; USB removal needs a configuration-only path. |
| [Local-edit effect](../packages/uhk-web/src/app/store/effects/user-config.ts) and [Save effect](../packages/uhk-web/src/app/store/effects/device.ts) | An explicit editing-action list recalculates length, persists local edits and shows Save to Keyboard. Ordinary device save checks configuration versions and capacity. New USB controls must join this edit/save flow. No new relay-support check is added to saving, import, update or recovery. |
| [USB properties](../packages/uhk-usb/src/constants.ts) and [device-state reader](../packages/uhk-usb/src/uhk-hid-device.ts) | Version, Git, checksum and half-presence data exist. Existing version helpers already gate other features; no left-host support gate exists yet. Half presence and USB enumeration do not prove relay support. |

These are source-review findings. No Agent build, configuration write or hardware
exercise was performed for this PRD.

## Scope

Include persistent USB Left creation/reuse/removal, new-entry names, compatibility
checks for Add USB Left, existing connection-key integration, and focused
validation and regression tests for these changes.
Keep this first implementation in the current Connections screen and save flow.

Exclude firmware transport changes, automatic firmware installation, a new host
configuration format, new key-action opcodes, a live connection dashboard, and
configuration forwarding through left USB. Initial setup uses Agent's existing
right-half configuration connection; left USB input remains independent of that
control connection. Preserve the existing discovery filters: the left half is not
a second configuration device. Both cables on one computer must still select the
right half for configuration; left-only attachment keeps the existing missing-device
behavior. Existing update/vendor access to left USB does not change that contract.

Do not repair the separate [official-firmware USB-C reconnect issue](https://github.com/leadZERO/UHK-Firmware/issues/1),
redesign Bluetooth pairing, or automatically rewrite user macros. Supporting
normal release firmware requires a coordinated firmware/Agent release decision before upstream enablement.

Existing Agent edge cases are outside this work: oversized-list recovery, malformed
import handling, uncertain persistence after a failed save, general device-poller
hardening, and changes to updater/recovery write ordering. Do not add persistent
configuration-baseline tracking or global validation across every write path.
Prevent the new controls from introducing duplicates or overflow, and test that
they preserve established behavior. Existing issues may be recorded separately;
fixing them is not a prerequisite for this feature.

## User flow

1. Connect the right half to the computer running Agent. Agent loads configuration
   through its existing flow and checks the firmware support contract below.
2. On Connections, offer **Add USB Left** when support is confirmed and the current
   editable configuration has no left entry. Explain: "Connect a second computer
   to the left USB port.
   Use a connection key to choose which computer receives input."
3. Add the entry to the first eligible empty slot without moving other entries.
   Reveal the new row even if it was previously in the folded suffix. The name
   defaults to `USB Left`; if that name is already used, request a unique name
   before adding anything. Automatic switchover defaults to false. Use the normal
   unsaved-change and Save to Keyboard behavior; opening this screen never writes
   or adds a connection automatically.
4. If a unique left entry already exists, show that row and its configured name;
   repeated setup is a no-op. Offer the existing rename and switchover controls.
5. After a successful save and reload, the entry remains in its slot. Existing
   connection-key choices include it automatically. Setup must not switch the
   active input host, change a key binding, or require a new "next connection" key.
6. Offer **Remove USB Left** with the existing confirmation style. Empty that
   slot through a configuration-only action and the normal save flow. Do not call
   any Bluetooth unpair operation or remove USB Right. Permit removal of existing
   USB Left entries even when relay support is unavailable. Explain that keys
   bound to the removed slot still refer to that now-empty slot.

Show actionable outcomes for no free slot, duplicate left entries, name conflicts,
unsupported firmware, an unverifiable peer, and save failures. Do not silently
choose which duplicate to keep, replace an occupied slot, or discard edits.

## Configuration and selection requirements

| ID | Requirement |
| --- | --- |
| CFG-1 | Persist USB Left as existing type 2 and USB Right as type 1. USB entries contain type, name and switchover; they have no Bluetooth address. Retain the existing configuration version. |
| CFG-2 | Add USB Left must not create a second left entry. Reuse a unique existing entry; if duplicates already exist, explain the conflict and leave the configuration unchanged. Otherwise fill the first eligible empty slot within 0-21. Refuse addition when full or already oversized. Preserve occupied/paired slots and runtime-only data; do not compact, append overflow or truncate the list. |
| CFG-3 | Preserve all existing connection names. Use `USB Left` and `USB Right` as the defaults for newly created UHK80 USB connections. These are UHK80 product defaults; naming alone does not enable relay, and creation of a left entry still requires the Add gate. Explicitly distinguish products in the defaults generator so UHK60 defaults remain unchanged. Leave the product-agnostic `defaultHostConnections()` legacy migration behavior unchanged. Firmware display/fallback naming scope is a separate tracked review item. |
| CFG-4 | Preserve existing switchover preferences. A new left entry starts with switchover false. If another connection already uses the proposed default name, require a unique name before addition; do not rename the existing connection. |
| CFG-5 | Addition/removal never compacts or reorders the list. Preserve all key-action slot IDs. An explicit user reorder keeps the existing positional semantics: a key still selects its numbered slot, whose occupant may change. |
| CFG-6 | Add an explicit zero-based `index` when creating the entry. Keep one-based UI labels and macro indices separate from serialized zero-based connection IDs; cover the first and last slot in tests. |
| CFG-7 | Preserve unrelated settings and pending edits. Add/remove actions are atomic and join `saveUserConfig$` in `store/effects/user-config.ts` for length recalculation, local persistence and Save to Keyboard visibility. Do not copy the paired-device auto-save effect. Save errors retain edits and do not report success; verify saved data by readback. |
| CFG-8 | Preserve existing load/import/restore/export behavior and schema/capacity checks, including round trips of existing USB Left entries on unsupported firmware. Add no global import validation or oversized-list correction workflow. Apply CFG-2 to the new Add action; keep unrelated editing and normal saves available. |

Reuse `ConnectionsAction` for slot-based selection and the existing Next,
Previous and Last choices. Do not add a second representation of a host or bind
connections by display name. Existing macro commands remain firmware-owned:
`switchHost 'USB Left'`, quoted custom names, one-based indices, and
`switchHost nextActive`. Document quoting and verify parser-compatible escaping
for names containing quotes/backslashes before generating any macro text.
Do not introduce a new `nextActive` key-action opcode for this feature.

Offline selection and automatic switching remain firmware behavior: an explicitly
selected unavailable left host may wait for readiness; `nextActive` skips it.
An enum value or saved entry alone must never be displayed as proof that the left
host is connected, active, or able to receive input.

## Normal editing and save flow

Add/remove participates in Agent's ordinary local editing and Save to Keyboard
flow. Check relay support only for Add USB Left, using current device/module
information. Removal and editing of existing entries remain available without
relay support. Validate the Add action against the current editable list.

After addition, the entry follows ordinary configuration rules. Do not track its
creation history, add a pending-addition save check, or introduce another model of
persisted configuration. Loss of the left peer does not add a save restriction;
existing device availability, schema, capacity and error checks still apply.
Equivalent configurations follow the same save rules whether an entry was added,
loaded, imported or restored. Import, history, shared configurations, firmware
updates and recovery retain their existing behavior.

Before accepting this policy, verify that a valid saved USB Left record is harmless
on supported-schema firmware without relay: configuration and unrelated host input
remain usable, and selection cannot misroute input or leave keys held. Cover older,
relay-disabled and mixed-half images in A12. This is required evidence, not an
assumption that enum support establishes safe runtime behavior. If it fails,
resolve the specific compatibility defect before acceptance; do not silently add
a general save-validation system.

Use existing save results and error handling. Failures retain edits. Verify actual
persistence through save, reload and power-cycle tests with the development slot
disabled. Partial-save repairs and storage-baseline tracking remain out of scope.

## Firmware compatibility: reuse existing version gates

Agent and firmware already have compatibility mechanisms. This feature must reuse
them; it does not require a new general-purpose capability-negotiation framework.

| Existing mechanism | Verified example / behavior |
| --- | --- |
| Device protocol feature gates | [Pairing-with-slot support](../packages/uhk-common/src/util/is-device-protocol-support-new-pairings-with-slots.ts) requires device protocol >=4.19.0. `UhkHidDevice.getPairedDevices()` selects the new API or the older fallback accordingly. Firmware checksum queries are similarly gated at protocol 4.10.0. |
| Firmware/configuration feature selectors | [Store selectors](../packages/uhk-web/src/app/store/index.ts) enable macro playback and layer double-tap from firmware 8.4.3, and macro commands from user configuration version 5.0.0. |
| Require a newer Agent | [shouldUpgradeAgent](../packages/uhk-common/src/util/should-upgrade-agent.ts) checks whether incoming configuration requires a newer supported major/minor schema. Import handling displays an update-Agent error rather than parsing unsupported data. This is primarily schema compatibility, not a firmware-to-Agent application-version lookup. |
| Require newer firmware | [shouldUpgradeFirmware](../packages/uhk-common/src/util/should-upgrade-firmware.ts), save/import checks and the firmware parser reject incompatible configuration versions. Keep these protections; do not bump the configuration format merely to expose this UI. |
| Read both halves' versions | [getHardwareModules](../packages/uhk-agent/src/services/device.service.ts) already reads right-half versions and, when connected, left/module version information. Reuse that data before adding USB queries or IPC fields. |

**Chosen development contract:** relay-enabled UHK80 halves report firmware
**18.0.65535**, defined once in firmware's `shared/versioning.h` as
`UHK_USB_LEFT_RELAY_FIRMWARE_VERSION`. The generator selects that value only for
UHK80 builds with `CONFIG_UHK_USB_LEFT_RELAY=y`; disabled builds retain the normal
`scripts/package.json` version (currently 18.0.4). Device-protocol and configuration
versions are unchanged. This version identifies compiled support, not current
USB readiness or successful relay negotiation.

Implement an ordinary support helper/selector using the existing firmware and
module version data. Define a named Agent constant for the provisional version
and require **exact equality on both halves for Add USB Left**. Do not use `>=18.0.65535`: a later
ordinary release does not guarantee relay support. Older experimental images
reporting 18.0.4 are unverified and must be updated as a pair for new setup.
No new USB property, capability query or global minimum-Agent-version registry is
required. Agent still cannot enable a compile-time firmware option.

The provisional marker and exact-match exception are development-only decisions
for this fork. Before upstream enablement, agree on the official support version,
enabled release builds and Agent gate with the maintainers through
[firmware issue #6](https://github.com/leadZERO/UHK-Firmware/issues/6). Update the
firmware constant and Agent gate together; use a minimum-version gate only when
the corresponding official builds guarantee support. Explicitly decide whether
to retire the provisional exception. A new capability framework would require
separate justification and agreement.

Firmware runtime negotiation determines whether reports can be delivered to left
USB. The Agent gate only establishes eligibility to add the configuration entry.
Do not infer live support from a saved entry, USB enumeration, configuration schema,
repository name or diagnostic text.

| Support evaluation | Agent behavior |
| --- | --- |
| Both UHK80 halves report the provisional relay version | Enable Add USB Left. Left host USB may be unplugged; readiness is separate from feature support. Saving retains ordinary rules. |
| Right supports relay; peer absent or cannot be verified | Explain how to connect/verify the left half and disable Add USB Left until verified. Preserve entries and ordinary editing/saving during temporary link loss. |
| Explicitly incompatible peer or disabled/unsupported firmware | Explain the requirement for compatible firmware on both halves. Disable Add USB Left and preserve existing configuration and ordinary saving. |
| Unknown versions or failure of the feature support read | Keep Add USB Left unverified and the rest of Agent usable. Do not create an endless error loop or erase data. |
| Desktop without a live configuration device | Preserve local viewing/editing and export; retain existing physical-write restrictions and disable unverified Add USB Left. |
| Browser demo | Allow simulated USB Left add/remove for the UHK80 model using explicitly mocked support. Use the same configuration operations; do not claim live hardware support or send physical writes. |

Derive Add eligibility from existing device/module information and invalidate it
on disconnect, device replacement, changed firmware/peer data or a failed support
read. On Add, evaluate the current device's data and refresh through the existing
serialized query path when needed. Reject stale responses from another connection.
Keep unrelated device information, edits, polling and save behavior unchanged;
this requires no new background setup service.

The browser demo may use the established synthetic-device path with an explicit
UHK80 relay-support fixture. Keep that mock outside physical support evaluation;
do not change bundled firmware metadata or global version constants to enable it.

## Fork development setup and release boundary

Development uses a manually installed, verified pair of signed relay images.
Confirm both reported versions before Agent setup. This manual workflow is a
fork development procedure, not the eventual supported installation flow.
Keep Agent's bundled firmware
selection unchanged: it downloads official releases, and the provisional marker
does not identify an official archive. No automatic installation, updater redesign
or changes to configuration-write ordering are included in this work.

Experimental archive packaging is separate release work. Before offering an
Agent-uploadable archive, its manifest version, image paths and checksums must
match its contents and pass the existing custom-update flow. Manual paired
installation does not establish archive support.

Verify that the new controls leave ordinary firmware update/recovery usable and
refresh their support state after an update. With official or mixed firmware, new
USB Left setup is unavailable; existing entries retain the established configuration
behavior. Baseline update/recovery failures are separate issues, not repair work
required by this PRD. The upstream enablement proposal must identify the
maintainer-approved firmware release and supported installation path.

## Development-slot boundary

`CONFIG_UHK_USB_LEFT_DEV_SLOT` inserts a left entry in RAM when no saved entry
exists. It does not persist configuration and yields to a unique saved left entry.
Agent must use its normal configuration model rather than runtime/OLED labels.

The supported configuration flow and acceptance images use relay-enabled firmware
with the development-slot flag **disabled**. Creation, removal and persistence
must work without that helper. An older development build may recreate a RAM-only
slot after removal; record this only in development notes. Preserve user-authored
macros and existing index/cycling bindings.

## Implementation map and sequencing

| Work | Primary locations |
| --- | --- |
| Feature support checks | Existing version helpers in `packages/uhk-common/src/util/`, web selectors and existing version/module reads in `packages/uhk-agent/src/services/device.service.ts`. Gate Add USB Left only; reuse existing device/module state and query paths. Leave saving, general polling and updater/recovery writes unchanged. |
| Add/reuse/remove actions and persistence | `packages/uhk-web/src/app/store/actions/user-config.ts`, `store/reducers/user-configuration.ts`, `store/effects/user-config.ts` and the existing Save flow in `store/effects/device.ts`. Register USB actions in the normal local-edit effect; keep removal out of Bluetooth bond deletion and add no relay-specific save gate. Import/history/shared-config behavior remains unchanged. |
| Support selector and Connections controls | `packages/uhk-web/src/app/store/`, `components/device/host-connections/`, and `services/device-renderer.service.ts`. Derive UI state in selectors/container logic. |
| Product-specific defaults and serialization | `packages/uhk-common/scripts/generate-user-configs.ts`, `user-config-80.json` and generated 60/80 outputs; retain legacy behavior in `config-serializer/config-items/host-connection.ts` and `user-configuration.ts`. Assert UHK60 generated defaults and migrated/loaded names remain unchanged. |
| Connection-key regression coverage and help | `components/popover/tab/device/device-tab.component.ts`, `connections-action.ts`, and focused tests; reuse existing selection semantics. |

Keep models and serialization in `uhk-common`, transport in `uhk-usb`/Electron,
and presentation in `uhk-web`, following the repository's package-boundary rules.
Do not move UI helpers to `uhk-common` merely to make them easier to test.

1. Build/update both firmware halves with the provisional relay version, then implement the support selector using the existing version/module data.
2. Implement Add/remove validation, local-edit persistence and the Add-only
   support gate using existing IPC/query paths. Integrate focused tests into the
   existing Agent CI while firmware review work proceeds.
3. Build the final paired candidate with the development slot disabled and
   complete targeted hardware acceptance plus the extended local trial below.
4. Review the recorded evidence before proposing the implementation upstream.
   General availability also requires the agreed release/version contract.

## Acceptance and validation plan

All items below are planned requirements, not completed validation.

| ID | Required evidence |
| --- | --- |
| A1 | JSON and binary round trips preserve USB types, custom names, switchover, all 22 slots, unrelated settings and key-action IDs. USB entries never gain an address field. |
| A2 | The new Add action handles no-left, unique existing-left, existing duplicates, first/last empty slot, full or oversized lists, folded rows and name conflicts without unintended mutation. It creates neither duplicates nor overflow and is idempotent. General import correction is outside this test requirement. |
| A3 | Rename, explicit reorder and removal preserve positional binding semantics. USB removal performs zero Bluetooth unpair calls and preserves the right entry and bonds. |
| A4 | Regeneration produces UHK80 USB Right defaults while leaving UHK60 generated defaults unchanged. New UHK80 left entries use USB Left. Legacy migration behavior remains unchanged, and all existing connection names survive import, save and restore unchanged. |
| A5 | The Add gate handles exact supported pairs, disabled/old/newer-unsupported versions, mixed pairs, missing peers, failed reads, reconnect and device swap. Reject stale support data. After Add, losing the peer or restarting Agent does not introduce a new save restriction or discard edits; existing editing/saving rules apply. |
| A6 | Add/remove uses local persistence, length recalculation and Save to Keyboard visibility without an immediate physical save. Save failure retains edits. Save/readback and a full power cycle retain one left entry at the same index with the development slot disabled; removing it persists too. |
| A7 | Existing Next/Previous/Last and direct-slot bindings keep working. Name/index/nextActive macros work with configured names; spaces and parser-supported quoting/escaping are covered without rewriting user macros. |
| A8 | On separate Windows/macOS computers, both halves and modules send keyboard, mouse/scroll and controls only to the selected USB host. Include held keys/modifiers/buttons across switches, an unavailable left host, and repeat/release checks. |
| A9 | Repeat selection and sustained input with the bridge attached and absent, then restore it. Check independent sleep/wake and saved configuration after restart; correlate failures with the firmware acceptance issue. Record Linux coverage or its absence explicitly. |
| A10 | Right-only use, both host cables on one computer, left-only attachment, Bluetooth/dongle hosts, backup/restore and other models retain existing discovery/configuration behavior. Browser demo add/remove operates on simulated UHK80 data only. Baseline USB-C enumeration failures stay separate and use the known-working cable path for retesting. |
| A11 | Run focused configuration, Add-gate and local-edit/save regression tests in existing Agent CI alongside applicable builds and lint. Firmware CI must run retained regressions and build both halves with relay enabled and disabled. Identify mocked boundaries, retain practical negative controls and prefer complete production units where feasible. Record UI verification and pre-existing failures separately. |
| A12 | Existing load/import/restore/export and ordinary save round-trip valid USB Left entries under established schema/capacity rules. On older, relay-disabled and mixed-half supported-schema firmware, test saved-entry reload/reboot, unrelated host input and selection for misrouting/stuck keys. Configurations created by Add and import have the same save behavior. Record actual compatibility evidence; do not infer harmlessness from successful serialization. |
| A13 | After manual paired installation, Add recognizes support. Ordinary firmware update/recovery remains usable, with support refreshed for the resulting firmware and existing configuration behavior preserved. Custom-archive packaging and repair of existing update/recovery faults are separate work. |
| A14 | Before proposing the implementation upstream, complete the targeted matrix and extended local daily-use trial on identified Agent/firmware candidates. Record duration, active use, scenarios, failures and retesting; a quiet period alone is not acceptance. |

Follow CI's build-before-lint order because lint needs generated files: use the
existing build, lint, web build and test commands. The web tests use Node's test
runner; `uhk-usb` exposes `test:skip`, and the Electron Agent package currently has
no test script. Do not count those gaps as passing USB/IPC coverage or introduce
a repository-wide Angular/Electron test-framework migration for this feature.
Keep new tests focused and make remaining manual integration checks explicit.
The reviewed checkout specifies Node `>=24.18.1 <25` and npm `>=11.13.0 <12`;
recheck these before implementation.
Acceptance targets the new feature and regressions it causes; existing Agent edge
cases are recorded separately without expanding this implementation.
Hardware results must identify Agent commit/build, both firmware image hashes,
OS/cable paths, bridge/Bluetooth state and pass/fail/untested outcomes. Host tests
must not be presented as proof of physical key release or two-computer isolation.

## Final-candidate validation and extended local use

Before proposing this implementation as an upstream PR, build and exercise the
identified Agent revision and both final firmware images. Disable the RAM-only
development slot and configure the saved entry through Agent. Complete targeted
tests A1-A13 and an extended period of normal local work with the pair. Record the
planned duration and usage coverage before starting, then log actual active
days/hours; an idle keyboard does not establish daily-use coverage.

Keep a concise log of OS/cable paths, selected host, bridge/Bluetooth use,
switches, sleep/wake, pointer behavior, latency, stuck/dropped input, resets and
recovery steps. Cover right-only use as well as two-computer isolation. Record
hardware or scenarios not exercised. Agent should be used for normal configuration
work during the trial, while input must also work when Agent is closed.

Attribute every result to the Agent commit/build and both firmware image hashes.
After behavior-affecting changes, repeat affected targeted tests and a documented
period of daily use on the new candidate; do not carry forward an older candidate's
pass as proof. Investigate failures and record the human tester's acceptance of
the evidence before upstream submission. Extended use supplements fault-focused
tests and does not establish unmeasured memory, stack, power or radio behavior.

## Firmware follow-up and evidence ownership

The [firmware follow-up](https://github.com/leadZERO/UHK-Firmware/issues/11) tracks alignment with this PRD:
named routing states and permitted transitions, callback/buffer ownership across
reconnects, timeout rationale, and behavior when drain cannot be proven. Preserve
the release and stale-completion protections while improving readability.

It also tracks explicit product/build scope for firmware fallback names and a
concise authoritative behavior/validation record. Remove personal/development
name examples from current requirements; keep investigation history separately
with its actual tested revisions. Agent UHK80 defaults do not authorize changing
UHK60 defaults or unrelated firmware build behavior.

Use [firmware CI #3](https://github.com/leadZERO/UHK-Firmware/issues/3) for retained
regressions, the version-marker check, paired relay-enabled/disabled builds and
existing other-device coverage. Record mocks and practical negative controls;
prefer complete production units when feasible without a test-framework rewrite.
[Hardware acceptance #5](https://github.com/leadZERO/UHK-Firmware/issues/5) owns
the physical matrix and resource/latency measurements, including memory/stack
headroom and power impact. The [Bluetooth rollover investigation #4](https://github.com/leadZERO/UHK-Firmware/issues/4)
remains a separate feature regression check.

As recorded on 2026-10-12, integration `19b8a2483` passed host regressions but had
not been physically retested; the newer local version-marker change also needs
paired builds and hardware validation. This PRD records requirements, not their
completion. Link final results from the issues and the eventual PR.

## Eventual upstream contribution

Follow Agent's [CONTRIBUTING.md](../CONTRIBUTING.md): verify its CLA signing step,
seek early maintainer feedback on scope/UI/release decisions through an issue.
Submit the implementation PR after the targeted and extended local validation
above. The fork issue and this PRD do not establish upstream agreement; no
upstream outreach is part of this revision.
Keep implementation in the existing Connections controls, state/effect conventions
and package boundaries. Avoid a redundant relay-enable switch: the saved entry and
firmware support already determine whether setup is available. Keep the technical
PR focused on behavior, compatibility and validation; this document is the fork's
planning record, not a requirement to add a new upstream documentation process.

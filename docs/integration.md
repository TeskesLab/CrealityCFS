# Integration guide

An integration needs two cooperating controllers: a protocol/state engine for the CFS and a printer-side coordinator for the local filament path. Sending a feed command is not a complete material change.

This guide separates **recovered native behavior** from **recommended host policy**. It covers the main CFS applications 1.1.3 and 1.5.2, not an assumption that every Creality image or fitted hardware variant is identical. Start with [architecture](architecture.md), [transport](transport.md), [addressing](addressing.md), [commands](commands.md), [movement modes](modes-and-workflows.md) and [compatibility](compatibility.md).

All wire command, status, action, selector and mask values below are hexadecimal. Time, temperature and mechanical quantities are decimal. Request payload lengths exclude the six framing bytes. Firmware delays describe requested scheduling, not hard wall-clock guarantees.

## 1. Define ownership before opening the bus

| CFS application owns | Printer/host integration owns |
|---|---|
| Four spool feeders; shared hub motor; material and connection input banks; two buffer inputs; wheel/encoder; tag readers/cache; environment cache; local indicators; background preload and motor protection | UART/bus access and correlation; address identity/profile selection; local toolhead extruder and heater; cutter and its feedback; local/five-way filament sensor; axis homing/parking; purge/wipe; printer pause/resume; tool-to-slot mapping; persistent host metadata; operation recovery |

`0x07` controls the **CFS hub**, not hotend extrusion. The native host’s method named `extruder_extrude` merely clears a bookkeeping flag; actual local extrusion occurs in printer motion calls. Neither that method name nor CFS encoder progress establishes filament delivery through the nozzle.

A standalone CFS controller can inspect identity, raw sensors and cached material data, and can coordinate CFS-only feed/retract mechanics. It cannot infer heater readiness, cutter success, local extruder engagement, or nozzle flow from CFS replies alone. Full printer integration must supply these inputs and controls independently. A connected CFS is not authorization to heat, cut, move axes or enable automatic spool motion.

Recommended host policy is to retain separate records for:

- **Device:** opaque UUID, current address, device type, APP/LOADER mode, application version/SN, supported payload profile and connectivity observations.
- **Slot:** physical A–D index, wire one-hot selector, material/connection observations, insertion/preload event, raw tag/cache status, live metadata, saved metadata, selected/loaded ownership.
- **Operation:** device and slot, workflow phase, last transmitted request, expected response shapes, acceptance, telemetry, terminal result, deadlines, local sensor handoff and unresolved outcome.
- **Printer:** local sensor, cutter feedback, heater readiness, homing/positioning, extrusion mode/speed, paused/printing state and explicitly saved restoration values.

Do not copy the native host’s single upper-level `buffer` cache into a new multi-device design: keep each `0x05` observation associated with its reply address. Do not identify a physical slot solely by the most recently loaded tool token.

## 2. Read-only bring-up

“Read-only” here means no requested motor, persistent-write, reset, factory or heater operation. A polling command can still consume a firmware event/error latch or refresh online/indicator bookkeeping; it does not freeze independently running preload/PRINT workers. Observe an already-running system accordingly.

1. **Give one process ownership of serial I/O.** Use the framing, serial parameters and incremental parser in [transport](transport.md). Do not let a printer plugin, address manager and diagnostics tool independently transmit on the same port.
2. **Identify without changing addresses.** Query known assigned material-box addresses with `0xA2`/`0xA3` and validate the complete type/mode/UUID reply. To look for an unassigned box use the group discovery path in [addressing](addressing.md); it does not enumerate every assigned box. A timeout is not proof of absence, and one discovery reply is not proof of complete enumeration.
3. **Separate APP from LOADER.** The identity mode field is authoritative for that domain. Do not infer loader mode from an address such as `0x81`. Do not automatically jump out of LOADER, assign an address, update flash, or reset as part of read-only discovery.
4. **Select the application profile.** In APP, read `0x14` version/SN and `0x0C` persistent factory mode. Preserve raw identity bytes. The version bytes are ASCII characters, not numeric components. Refuse ordinary movement if identity/profile is inconsistent or factory mode is nonzero; inspect rather than rewriting it to “fix startup.”
5. **Take nonmoving observations.** Read `0x08 [0x00]` material, `0x08 [0x01]` connections, `0x05` buffer, `0x0E [0x01]` encoder, and `0x0A` status. Tag queries `0x02 [mask]` and remaining-state queries `0x03 [mask]` access caches. `0x15` is startup hardware-diagnostic data, not a live arbitrary six-sensor mask; interpret its category encoding through [commands](commands.md) and [sensors](sensors.md).
6. **Publish uncertainty, not defaults.** Associate every sample with address, receive time and validity. An absent value is not a clear sensor; a `busy`, `unknown`, `none` or `0xFF` cache value is not interchangeable with valid stock metadata. Preserve the special event/status branches described below.

Do **not** add `0x0D` action `0x02/0x03`, `0x04`, `0x06/0x07`, `0x09`–movement stages, `0x50`, `0x56`, `0x5C`, or `0x5D` to a startup probe list. Some move motors, change persistent state, reset, or run fixture operations. In particular, `0x01` is an indicator refresh/connection ACK, not a full hardware initialization command, and `0x5D` is not a normal initialization substitute.

## 3. Initialization, addressing and profile admission

### Native startup is not a mandatory sequence

The CFS initializes motor/sensor state, storage, encoder/environment handling and workers locally. It reads persistent factory mode before enabling the ordinary extended preload/RFID flag on the normal returning path. The host does not need to issue raw driver initialization or a factory test to make ordinary protocol queries work.

The recovered native printer host creates state and saved-state objects before deferred device lookups. At printer-ready it resolves serial, toolhead, print-state, rack/material and configured local sensor objects, emits its ready event, then queues its heartbeat callback. The local filament sensor emits the separate feed-handoff event. These are printer framework lifecycles, not required wire commands; a new host should implement equivalent ownership, not require the original Python modules.

The native connection reconciler admits address-manager entries that are online and in APP mode. For a newly admitted address it marks the host cache connected, polls state, usually requests IDLE **unless PRELOAD is reported**, reads version/SN, saves UUID, and sends `0x0D` OPEN/CLOSE according to host preference. First-ever admission also synchronizes material inputs and schedules metadata reconciliation. Reconnect and first admission are not identical branches.

There are important reasons not to blindly replay it:

- The native cached “last box mode” local can survive across address iterations; a missing response can reuse another address’s mode. A new integration should use explicit per-device observations.
- Native connection startup may send IDLE and enable background workflows: it is **not** the read-only procedure above.
- Optional native power-on preload can issue forced action `0x02`, then tag/length queries while toggling the printer sensor. That is explicit motor-moving host policy, not an inherent discovery requirement. Ordinary cleanup calls are not exception-proof.
- A transport-level address/command match does not establish version compatibility. The legacy host builders do not universally match either mapped application: its old two-byte `0x10` form and two-byte `0x13` form must not replace the exact application layouts. `0x12` is 1.5.2 only.

Recommended admission order is therefore **identity → APP/LOADER distinction → exact profile → nonmoving observations → deliberate workflow preferences → operation eligibility**. Restore saved address reservations only through the controlled assignment procedure in [addressing](addressing.md), and verify UUID after assignment. Persisting a mapping does not prove that the assigned device acknowledged it.

## 4. A receiver that can distinguish replies, events and progress

### Parse first, interpret second

Maintain one incremental byte-stream parser. Keep partial frames across reads; validate length and CRC before decoding business fields. At the protocol layer, preserve actual reply address, request-mode/status byte, command, payload length, payload and arrival order. See [transport](transport.md) for malformed-frame handling and reply exceptions.

There is no on-wire transaction number. Keep a serialized request lane and explicit operation phase rather than allowing arbitrary concurrent waiters for the same address/command. Correlate to the outstanding operation only after frame validation. A request timeout leaves an ambiguous outcome; it does not make a late frame belong to the next command automatically. Quarantine/reconcile late replies before reusing that correlation key.

The decoder must branch on **status, command, actual payload shape and operation phase**, not merely the command byte:

| Received form | Decode/route |
|---|---|
| `0x0A`, normal status, profile-sized payload | 1.1.3: four bytes; 1.5.2: six bytes. Decode temperature/humidity/event-motor field/operation state, then version-specific trailing fields. Normal status can be a latched error rather than `0x00`. |
| `0x0A`, status `0x30`, four data bytes | Four separate A–D slot event codes. **Not** temperature/humidity/status, even though its length equals the 1.1.3 normal payload length. |
| `0x10`, status `0x00`, empty during action `0x04` admission | Accepted, not completed. |
| `0x10`, four data bytes after action `0x05` | BE binary32 encoder plus status. During the private action-`0x04` loop this is telemetry; after the worker exits it may consume its latched result. |
| `0x10`, empty after action `0x06` | Direct buffer result, including possible status `0x0D`. It may be private completion or standalone fill after an early action-`0x04` failure; require the serialized normal action-`0x05` result check before advancing. |
| `0x10`/`0x12`, one state byte after prepare | Completion result plus post-finalization operation state; not a sensor or encoder. |
| `0x52` request, response command `0x53` | Documented environment-query exception. Validate the expected two-byte reply; do not require identical command codes universally. |

The normal `0x0A` motor/event field is not the four-slot event array. Its high bit can identify a latched motor block mask; normal polling consumes relevant latches. Temperature is signed; the firmware humidity byte is unsigned despite native-host signed conversion. The 1.5.2 last byte is RFID/workflow busy/pending information, not a raw material-presence mask. See [sensors](sensors.md) for the exact layouts.

The native PRINT consistency response `0x50` sets a separate low-progress-suppression flag and indicator, not ERROR or the operation error latch; it neither stops motors nor masks buffer interrupts. That flag suppresses the PRINT worker’s low-progress `0x52` path. Keep these outcomes distinct; see [PRINT protection](modes-and-workflows.md#timing-and-protection-are-not-a-safety-channel).

Recommended host policy is to record frame decoding and update state first, then publish events or wake operation waiters. Do not let an event callback run arbitrary serial requests inline inside the byte parser, hold the parser’s lock during printer work, or mark an operation successful because an unrelated event was processed normally.

### Per-slot event transitions

Status `0x30` carries **exactly four event bytes**, in A/B/C/D order. Values are `0x00` unchanged, `0x01` insert, `0x02` extract, `0x03` completed. These are byte values, not packed two-bit fields. Reject a malformed short event payload as a whole in a new implementation; do not apply a prefix and then silently default missing slots. Preserve unsupported values for diagnostics without fabricating transitions.

Events are notifications from an internal scheduler, not a durable queue or proof of current physical state. Their source latches can be cleared or coalesced. Reconcile current material/connection/cache observations after reconnect, missed polls or ambiguous events. A completed event is not a full printer-load completion.

The native host applies the following policy. Here “preloading enabled” is the **host preference used by event handling**, not evidence that both firmware control bytes were accepted.

| Event/context | Native host behavior |
|---|---|
| `0x00` or unsupported | No per-slot metadata action. |
| `0x02` extract, any print/preload preference | Set live vendor to literal `none`; retain saved metadata and live color/type/length. It is not a full saved-record deletion. |
| `0x01` insert, preloading enabled, neither printing nor paused | Set live vendor `unknown`; do not immediately request RFID or clear every other field. |
| `0x01` insert, preloading enabled, printing/paused | Check the separate five-way/local sensor. Exact `False` triggers a connection-update script before active saved-metadata restoration. Other falsey objects are not the same native branch. |
| `0x03` completed, preloading enabled, neither printing nor paused | Defer a slot-specific cache/RFID callback, selected by the event’s address and one-hot slot. No immediate metadata rewrite. |
| `0x03` completed, preloading enabled, printing/paused | Active saved-metadata restoration; no five-way sensor check and no idle auto-RFID callback. |
| `0x01` or `0x03`, preloading disabled, printing/paused | Active saved-metadata restoration, without the enabled-insert sensor procedure. |
| `0x01` or `0x03`, preloading disabled, inactive printer | Set vendor `unknown`; restore manual color/type only for saved vendor `unknown` and a valid saved color; otherwise explicitly clear color/type to `unknown`. Length is left unchanged. |

Active restoration has a separate decision tree:

1. Saved vendor literal `-1`: set vendor/color/type to `unknown`.
2. Saved vendor `unknown`: set vendor `unknown`; restore saved color/type only if saved color is neither `unknown` nor `-1`. Otherwise **retain existing live color/type**, unlike the inactive explicit-clear branch.
3. Saved concrete record: restore vendor, saved length, color and type. Host B additionally checks slicer-used-slot membership for enabled active inserts and disabled active inserts/completions; a failed check clears vendor/color/type. That check is not physical presence. It does **not** gate the enabled completed-event branch.
4. Saved length equal to **numeric** zero triggers a remaining-length query, but the native path discards its return and later stores the **original zero**, potentially overwriting receive-side updates. String `"0"` does not take that equality branch. A new state engine should keep query results and event snapshots ordered, not reproduce the overwrite accidentally.

After all four events, the native host schedules saved-state and same-material reconciliation in that order. Its field writes/persistence are nontransactional: failure after a prefix does not roll it back; a deferred callback is not an additional wire ACK. RFID `busy` has another native quirk: it is accepted as a sentinel but then treated as concrete metadata in a downstream branch, yielding empty color/type slices. New integrations should preserve cache status separately from validated material identity. Full material/storage formats are in [RFID and material](rfid-and-material.md) and [storage](storage.md).

## 5. Polling and scheduling

Recommended policy is to make polling cooperative with a single operation scheduler:

- Poll admitted devices while no incompatible long handler owns the bus. Schedule material/cache refresh separately from rapid operation telemetry.
- Use profile-sized `0x0A` decoding and retain both ordinary state and event paths. Track transport reachability, application error, busy operation and observed physical inputs separately.
- Suspend ordinary background traffic **bus-wide** for the private action-`0x04` exchange. Its reader consumes from the shared UART, ignores normal dispatch, and can drain unrelated frames. Even polling another address can interfere.
- Continue the receive loop and printer event processing while waiting for a command. A blocking “send then wait on the same thread that must dispatch the local sensor event” cannot implement the handoff.
- Resume background polls only after terminal completion or a reconciled early-worker exit. Do not use a fixed sleep to assume normal dispatch resumed.
- Give the operation coordinator sole ownership of consumptive result-latch reads throughout action `0x04` and its post-action-`0x06` normal action-`0x05` check. Background diagnostics/status readers must not consume an early failure before that check; a broken ownership history makes the result unknown.

For comparison, native heartbeat behavior is:

- Wait up to ten half-second pauses for the power-on preload preference to become non-`None`.
- Rotate through addresses `0x01`–`0x04`, refreshing connectivity each pass. Poll `0x0A` only for connected, per-address-enabled entries; after the work, check aggregate connectivity and pause until a fresh monotonic time plus 1 s.
- Each address starts with a five-miss budget. Only an exact `None` result decrements it; native `False` resets it. Repeated misses call timeout handling and mark disconnected. This is native return-object policy, not a wire guarantee that an error reply is healthy.
- Host B has one local successful-poll counter shared among addresses. At ten non-`None` results it rechecks the selected address’s enable flag and requests `0x03 [0x0F]`; if newly disabled it retains the counter. Host A lacks this added cadence.
- Per-address heartbeat disable does not cancel the callback, connectivity reconciliation or its pauses. Shutdown is checked at loop boundaries; transport/parser exceptions can terminate the callback without an automatic restart.

The original timeout arguments vary sharply: ordinary queries commonly use 2 s, `0x10` uses 15 s, `0x11` uses 150 s, forced `0x0D` can use 300 s per selected slot, and `0x0A` uses 3600 s. These are host call arguments, **not required protocol timers**. Firmware stage windows and blocking waits differ from them. A replacement host should distinguish acceptance timeout, telemetry interval, expected mechanical window and operator recovery deadline rather than applying one short ACK timeout everywhere.

## 6. Loading: preparation → handoff → buffer → PRINT

Before authorizing motion, establish target device/slot, material presence, current connected paths and buffer state. Ensure the printer-specific local sensor, heater, cutter and axis policy is ready; CFS cannot establish these prerequisites. Decide explicitly whether automatic preload is allowed (`0x0F`) and whether extended tag/margin workflow is enabled (`0x0D`). They are not a coupled enable bit. Forced `0x0D/0x03` is mechanical pretension with its temporary extended flag cleared, not an RFID scan; `0x02` permits the extended phase. Both forced actions first check/retract all slots independently of the mask, even mask zero, require IDLE or PRELOAD (otherwise `0x03`), and report all-slot absence as `0x16` in 1.1.3 versus `0x18` in 1.5.2. See [background controls](modes-and-workflows.md#0x0d-extended-rfidpreload-workflow).

A practical staged load follows this order:

1. **Resolve ownership.** Map the requested tool to a physical device and A–D slot; verify UUID/profile. Preserve the prior loaded slot and relevant printer state for recovery. Do not overwrite “currently loaded” merely because a target was requested.
2. **Prepare the printer path.** If changing away from existing filament, cut/unload through the printer-specific flow below. Home/park as needed and establish heater readiness before local extrusion. Material target/flush temperature, fan behavior, purge volume and axis positions are printer/material policy, not CFS constants.
3. **Leave buffer operation deliberately.** If PRINT is active and the next operation requires IDLE, use `0x04` action `0x01` and require its actual result. PRELOAD/RELOAD rejects this command; do not treat that rejection as a successful stop.
4. **Run feed preparation.** Send `0x10 [slot, 0x00, 0x00]` and decode result plus state. It can evict other connected slots and has a hold/detooth wait; error is not permission to issue the next phase. In 1.5.2 `0x12` mode `0x00` offers preparation without that hold, but it is a distinct profile-specific choice, not a byte-compatible alias.
5. **Arm the local sensor handoff before starting action `0x04`.** Clear old host handoff/result records, associate a new operation generation with this slot, enable the event gate, then start `0x10 [slot, 0x04, 0x00]`. Record the empty acceptance separately from completion.
6. **Poll progress through action `0x05`.** While the private worker owns the UART, send only the scheduled `0x10` private exchange. A four-byte position/status response is not terminal success. Detect early failure through a later normal action-`0x05` result when the worker has exited; retain nonzero result immediately because the latch is consumptive.
7. **Perform the local handoff on a real sensor event.** The native callback first queues a local 5-mm extrusion at 60 mm/min, then sends action `0x06` to enter the CFS buffer-full check. That local command sequence has no inserted motion-completion barrier. A new integration must deliberately choose its local-motion barrier/heater policy; do not claim the sensor alone proves hotend engagement or silently assume all queued motion completed.
8. **Retain the direct action-`0x06` result, then check the normal result latch.** An empty `0x00` reply can come from standalone fill if action `0x04` already failed, so it is not sufficient proof of private completion. Keep the bus reserved and have the coordinator, as sole owner of consumptive latch reads throughout this operation, serialize a normal `0x10 [slot, 0x05, 0x00]` check after the action-`0x06` reply. Require status zero before engagement or PRINT; any nonzero result or unknown/lost check blocks advancement. Retain direct `0x0D` as failure regardless of a later zero. If the direct completion was lost, a later zero latch/IDLE cannot recover it; the outcome remains unknown.
9. **Coordinate short local/CFS feed and inspect buffer.** Native post-handoff loops queue local extrusion, pause, use action `0x07` with a byte distance, then wait/refresh `0x05`; their purpose is to move from “buffer still full” into local extruder engagement. The native default loop uses six passes, 9-mm local moves at 180 mm/min and a 3 firmware-mm CFS request. These are **host configuration defaults**, not universal hardware requirements. If action `0x07` returns `0x0E`, the native host can try up to five local-only 6-mm corrections, rechecking buffer each time. This is targeted recovery, not a generic replay of the failed CFS move.
10. **Enter PRINT only after handoff succeeds.** Require the retained successful direct action-`0x06` frame, the serialized zero normal action-`0x05` result under uninterrupted coordinator latch ownership, and successful engagement prerequisites. Send `0x04 [slot, 0x00]`; now buffer IRQs own ordinary hub replenishment. Verify the result and observe buffer/state. Follow with printer-local purge/wipe and chosen fan/heater/position restoration. Record loaded-slot ownership only at the defined successful handoff point.

The native wrapper temporarily disables the selected address’s heartbeat and enables its local sensor gate around part loading. On an **ordinary return** it reenables heartbeat then disables the sensor gate, before testing the returned failure object. These are not `finally` blocks. An exception can leave either setting altered. Native stage-event production also sets its “stage 7 available” flag **before** sending action `0x06` and before storing the result mapping; a callback error can expose a flag with stale data. In a new design, publish a fully populated handoff record atomically and attach it to the operation generation rather than copying this race.

The native part loop has a 90-count polling window with a one-second pause on the continuing path and action-`0x05` checks on even counts. Count checks precede some event handling. A handoff arriving during the last pause can lose to timeout reporting, and a stage-`0x05` error can take precedence over a simultaneously queued handoff. Use explicit ordered events/deadlines; do not interpret those native counters as a guaranteed 90-second handoff service.

Host B increments/persists its `change_color_num` **after stage-`0x00` or its recovery succeeds, before part loading and PRINT**. It is not a completed-material-change counter. Final native success updates last command, filament position and global filament flag only later. Keep requested, attempted, handed-off and completed-change accounting distinct.

## 7. Cutting and retracting

The useful high-level change order recovered from the native host is:

**snapshot printer context → prepare Z/park → cut if filament is known or locally detected → move to extrusion position → unload old path → load target → purge/wipe → restore selected printer context → resume only when eligible.**

Each stage can fail after earlier effects. Cutter motion, heater checks and axes are not CFS protocol operations. Cutting can use a switch-feedback retry burst, motor-current changes, local priming/retraction and special XY limits. Those settings depend on the printer. The native `SET_LIMITS`/`RESTORE_LIMITS` commands change **XY travel bounds only**; they do not restore acceleration, pressure advance, fans, input shaping or heater state. Do not transplant their coordinates or assume one restoration call repairs every temporary setting.

For a known loaded slot, a normal unload should distinguish two destinations:

1. **Leave PRINT with `0x04` action `0x01`.** Require actual acceptance; do not use action `0x10/0x03` as a physical-stop substitute.
2. **Release the buffer when filament reaches the printer.** `0x11 [slot, 0x00]` reverses toward buffer-free raw `0x02`. Coordinate this with local extruder withdrawal and cutter state. Buffer-free is not material-path-clear.
3. **Withdraw local filament appropriately.** Native code combines a half-buffer local retract with a further local retract after heater checks on specified branches. Its local sensor is re-read after failure; it does not blindly advance because the remote helper returned. Distances and extrusion mode are printer configuration, not a CFS recipe.
4. **Retract to the material path with `0x11 [slot, 0x01]`.** Require propagated status and inspect the selected connection input clear. Material still being present at the spool is expected; this is not an extraction event.
5. **Reconcile ownership.** Update the old/current slot records only after the required sensor/result conditions. Preserve prior identity and partial progress when failure leaves the path ambiguous.

For unknown loaded ownership, do not arbitrarily select the target slot and retract it. The native host scans devices and intersects material and connection masks, distinguishes local sensor detected/clear, and may use `0x11 [0x00, 0x00]` before choosing a connected candidate. The all-slot sentinel processes connected slots in A–D order and stops on the first failure; no processed-slot mask is returned. A new integration needs a deliberate unknown-path recovery procedure, not an assumption that an empty ACK emptied every path.

The native pre-cut helper also has a known mode-name mismatch: it maps the operation name to `PRINTING` but compares the restore string with `PRINT`, so the apparent reentry test does not match. It runs the whole nested pre-cut retraction only while the printer reports printing, and some ordinary `False` results are discarded by its parent. These quirks explain native behavior; they are not recommendations to recreate a missed restore or ignore cutter/pre-cut failure.

## 8. Failure, retry and recovery policy

### Classify before acting

Recommended result categories are **rejected without starting**, **accepted/in progress**, **completed with documented result**, and **outcome unknown**. Retain the raw frame with the category. Do not turn every nonzero status into the same error, or every status-`0x00` into physical completion.

| Situation | Appropriate next step | What not to do |
|---|---|---|
| Malformed frame or CRC failure on a nonmoving query | Discard malformed bytes, preserve other valid events, and consider a bounded query retry. | Retry every command through one generic CRC policy. |
| Timeout/lost reply after a moving request | Mark ambiguous, inspect/reconcile when the protocol can service queries, and use an explicit recovery workflow. | Replay feed/retract/forced scan assuming the first request never ran. |
| Busy/state rejection `0x03` | Preserve current state and inspect why its guard failed. | Treat it as success; clear state merely to force a stop through. |
| Action-`0x04` acceptance | Continue its private scheduled exchange and printer-sensor handling. | Release the bus or record material loaded. |
| Private action-`0x06` result lost | Preserve unknown buffer outcome; reconcile physical inputs and operator recovery policy. | Assume a later zero action-`0x05` latch recovered the completion. |
| Result `0x0E` from action `0x07` | Preserve the failed wire result even if subsequent state is IDLE. | Infer success from IDLE; blindly repeat a short motor move. |
| Parameter error / unsupported layout | Fix profile/length/selector before another attempt. | Send a longer `0x09` payload to repair the stale-byte bug or probe unsupported moving modes. |
| Reset, flash/write or factory operation loses response | Follow the operation-specific reconciliation in [maintenance](maintenance.md). | Apply normal command retry or reset to hide uncertainty. |

Native `send_data` retries CRC and LENGTH statuses through a shared counter, and its transport defaults can also retry. That is historical behavior, **not recommended universal safety policy**. Retry eligibility must be decided before sending and account for movement, persistence, consumptive reads and reply loss. Even “stop” is not uniformly idempotent or universally admitted: `0x07/0x00` rejects non-IDLE, `0x06` is defective and feeder release can detooth, `0x04` disable rejects PRELOAD/RELOAD, and `0x10/0x03` does not stop motors.

### Targeted native recovery patterns

These provide useful stage boundaries, not a promise that an automatic retry is safe for every printer:

| Failure context | Native higher-level order |
|---|---|
| Feed preparation/connection | IDLE → retract to MATERIAL (`0x11/0x01`) → feed action `0x00`; stop on the tested failure result. |
| Initial local sensor handoff retry | Loosen material, advance a bounded sensor-retry record, reenter part loading. |
| Second sensor retry | IDLE → retract to MATERIAL → feed action `0x00` → part loading. |
| Extruder engagement/gear retry | Cut → park → unload → feed action `0x00` → part loading. Some native parent calls discard earlier ordinary failures; a new integration should not confuse that with verified prerequisites. |
| Buffer/empty-print fault | Native recovery can request IDLE then PRINT for the saved slot, recheck local sensor and heat. It does not necessarily perform the entire load/purge chain. |
| Purge-only failure | Reenter the purge/cleaning stage with saved material context rather than automatically feeding a second spool. |

Native error dispatch preserves the first error and saved device/slot context on many paths. It can pause or cool based on printer state, but an error report is not necessarily a raised exception or stop command. Some status returns are even counterintuitive: native `0x10` state-error reporting can return `True`, and several material-absence statuses return `True` when no last-command token exists. A new protocol layer should return the raw documented result independently of UI/reporting policy.

Native retry dispatch also waits on cached PRELOAD in selected fault cases. Host B’s filament-error path makes one state query and then checks cached mode with up to thirty one-second pauses; it does **not** query anew every iteration. If count exceeds thirty it uses its fallback even when the last check just left PRELOAD. Missing saved slot can clear error and return without the common resume tail. A timed wait on a cache is not proof that preload stopped or that a slot became usable.

### Watchdog limitations and abort handling

Do not advertise heartbeat as a stop channel. The application heartbeat worker skips timeout handling while command-busy; offline behavior primarily changes online/indicator state. Long synchronous handlers and the private reader do not service general cancellation. Buffer IRQs can issue hub drive changes independently; motor-fault cleanup does not itself mask them. Firmware motor/block and PRINT progress checks use soft scheduling and version-specific behavior, not a measured safety deadline.

On host cancel or communication loss, stop issuing new workflow steps, retain pending/result ambiguity, and invoke only printer/CFS stop mechanisms actually available in the current state and separately validated for the hardware. If no reliable stop path is established, report that limitation and require the system’s independent physical safety procedure. Do not claim that closing serial, losing heartbeat, clearing ERROR, disabling `0x0D/0x0F`, or setting an internal host flag halted the motors.

For host resource cleanup, distinguish safe software cleanup (release locks, disarm stale callbacks, invalidate an operation generation) from hardware actions requiring current knowledge (restore heat/fans/axes, resume print, reenable a feeder workflow). There is **no blanket native `finally` restoration contract**. Decide each effect explicitly and preserve failures rather than masking them with a generic success/cleanup tail.

## 9. Implementation acceptance criteria

A complete integration should be able to demonstrate, without relying on motor ACKs as sensors:

- Exact APP/LOADER identity handling and per-version payload admission; no movement on unsupported/ambiguous profiles.
- Incremental framing/CRC validation, address correlation, heterogeneous `0x0A` decoding, `0x52`→`0x53` handling and late-frame isolation.
- Separate action-`0x04` acceptance, action-`0x05` telemetry/consumptive result, action-`0x06` direct result and action-`0x07` failed-result/IDLE cases; block engagement/PRINT until the normal post-`0x06` result check succeeds under sole coordinator latch ownership.
- Bus-wide private-exchange scheduling with a receiver and local sensor event consumer that remain runnable.
- Four independent slot event transitions; raw/live/saved metadata separation; reconnect reconciliation rather than an assumption of lossless events.
- Successful-path feed → local handoff → buffer check → engagement → PRINT and unload → buffer-free → material-path retract, with every guard/result retained.
- Explicit local extruder/heater/cutter/sensor/axis policy, rather than treating `0x07` as a hotend command.
- Bounded, operation-specific retry eligibility; unresolved motion outcome after reply loss; no generic moving/flash-command retries.
- Version-specific PRINT protection expectations, realistic heartbeat limitations, and no implicit stop/resume/finally behavior.

These are design/validation targets, not claims that hardware tests have been performed here. [Wire examples](wire-examples.md) provide small concrete frames; they are not an executable automation recipe.

## Implementation limits

The documented behavior does not establish measured travel, slip, actual fitted peripherals, safe cutter coordinates, heater limits, scheduling under load or live interoperability. The first retract odometer delta has the register-initialization limitation described in [movement modes](modes-and-workflows.md). Slicer flush-metadata producer syntax is not established by its consumers alone. Internal reader/applet implementation is outside this communication guide’s scope, not a blocker to the documented mainboard wire contract.

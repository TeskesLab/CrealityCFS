# Modes and movement workflows

This chapter describes the application’s state machine and mechanical stages. It is not a statement that an acknowledged command produced the intended physical result. Read [transport](transport.md), [commands](commands.md) and [sensors](sensors.md) before implementing these flows; [integration](integration.md) explains host scheduling and printer coordination.

Unless stated otherwise, the detailed movement behavior below is the **1.5.2 application**. Both application versions have been mapped; differences are called out rather than treating 1.1.3 as an unknown protocol. All command, action, selector, state and result values use hexadecimal. Time and distance quantities are ordinary decimal quantities, not encoded wire values.

## Distinct mode domains

Do not use one enum for all of these domains.

| Domain | Values and source | Meaning |
|---|---|---|
| Device execution mode | Address identity: `0x00` APP, `0x01` LOADER | Which image/protocol is running. An assigned address is not evidence of APP or LOADER. See [addressing](addressing.md) and [maintenance](maintenance.md). |
| Application operation state | `0x0A` normal status payload | IDLE, PRELOAD, PRINT, RELOAD, ERROR or TEST, listed below. |
| Persistent factory operating mode | One byte from `0x0C`; changed by `0x50` | Selects a startup fixture/aging path. This is internal-flash parameter ID `0x06`, **not a six-byte EEPROM record**. |
| Buffer inputs | One packed byte from `0x05` | Two independent physical input levels; not an operation state. |
| Background workflow controls | `0x0D` and `0x0F` | Independent flags; neither is an emergency stop. |
| Per-slot insertion/preload events | Status `0x30`, four payload bytes | Transitions reported separately for A–D; not a replacement for the operation state. |
| Printer state | Host `printing`, `paused`, etc. | Host policy. APP PRINT can exist without a printing printer, and a paused printer can leave the CFS in PRINT. |

The native host calls operation `0x01` **PRELOADING**, `0x02` **PRINTING**, and `0x03` **WRAPPERING** in some display maps. The application’s own names below are less ambiguous. Host stage/log labels are also not additional wire actions: the host callback called “stage 7” sends action `0x06`, and its “stage 8” procedure uses action `0x07`.

## Application operation states

| State | Firmware name | Entry and important effects |
|---|---|---|
| `0x00` | IDLE | Initialization and successful operation finalization. Clearing this state does **not** itself stop motors. |
| `0x01` | PRELOAD | Background insertion/preload workflow. Feed preparation rejects it; the retract guard accepts it. |
| `0x02` | PRINT | `0x04` action `0x00` enables buffer-controlled delivery for one slot. Buffer interrupts can start/stop the hub independently of normal command dispatch. |
| `0x03` | RELOAD | Accepted movement preparation, retract, and several direct movement actions. Some actions overwrite the state without the feed-readiness guard. |
| `0x04` | ERROR | Most nonzero operation completions latch their result. Normal `0x0A` responses use that latch as response status. |
| `0x05` | TEST | Factory/diagnostic domain. Do not enter it as normal initialization. |

The shared finalizer has three materially different paths:

- Result `0x00`: clear latched operation error, select IDLE, refresh the idle indicator from material inputs.
- Result `0x01` or `0x03`: restore the previous current-slot selection; do not generally change the operation state or error latch.
- Other nonzero results: select ERROR and latch the result for later `0x0A` polling.

These are bookkeeping/indicator effects, not a universal motor-cleanup routine. Action `0x10/0x07` deliberately finalizes with `0x00` even when its wire result is `0x0E`. Private action `0x06` during action `0x04` sends a possible `0x0D` result that is subsequently cleared by its caller. Neither later IDLE nor a zero result latch proves that these earlier stages succeeded.

### Persistent factory modes

`0x0C` reads the stored byte without changing it. `0x50 [mode]` writes a changed byte and causes delayed reset; unchanged data returns status `0x03`. That status is a command-specific “unchanged” outcome, not the movement busy guard.

At 1.5.2 startup, mode `0x00` takes the normal return path. **Every nonzero value enters TEST before case selection**, including unsupported values. Recognized selectors include `0x01`, `0x02`, `0x06`, `0x07`, and `0x0B`–`0x26`: they select fixture, reader/authentication, calibration or aging helpers, sometimes chained or nonreturning. Mode `0x01` enters the four-channel cycle/retry workflow and ultimately a button-monitor loop; mode `0x02` selects another aging path. An unknown nonzero value reports unknown and returns with TEST still selected. Do not probe these selectors to discover capabilities.

The **1.5.2** selector entry paths are below. These are diagnostic entry paths, not supported printing modes or safe capability probes. Many first helpers never return; later calls in a branch are not a promise that a sequence will complete.

| Persistent selector | First diagnostic behavior |
|---|---|
| `0x00` | Normal startup return; does not itself perform runtime TEST-exit cleanup |
| `0x01` | Four-slot endurance/cycle engine, followed by a nonreturning button-monitor phase |
| `0x02` | RFID aging coordinator |
| `0x06` | Start the RFID-authentication diagnostic thread |
| `0x07` | Clear TEST flag, then loop per-slot feeder/hub forward/reverse exercise; clearing TEST does not stop it |
| `0x0B` | RFID aging coordinator with its alternate cycle argument |
| `0x0C` | Connection/encoder calibration, then the repeated calibration exercise if the first helper returns |
| `0x0D` | Repeated buffer-motion cycles, including a terminal diagnostic wait |
| `0x0E` | Buffer transition timing/count diagnostic |
| `0x0F` | Nonreturning insertion/removal counting for the slot range |
| `0x10` | Motherboard diagnostic wrapper, including drive/FG and GPIO checks |
| `0x11` | RFID production stress test; subsequent exercise steps require its return |
| `0x12` | Repeated automatic feeder exercise |
| `0x13` | Encoder-assisted hub feed/retract exercise |
| `0x14` | Nonreturning buffer-state/indicator display loop |
| `0x15` | Nonreturning material-input display loop |
| `0x16` | Drive feeders and hub in native direction 0 |
| `0x17` | Drive feeders and hub in native direction 1 |
| `0x18` | Drive feeders only in native direction 0 |
| `0x19` | Drive feeders only in native direction 1 |
| `0x1A` | Drive hub only in native direction 0 |
| `0x1B` | Drive hub only in native direction 1 |
| `0x1C` | Drive feeder A in direction 0, then spin without cleanup |
| `0x1D` | Drive feeder A in direction 1, then spin without cleanup |
| `0x1E` | Nonreturning material-selected feeder/hub exercise |
| `0x1F` | Nonreturning four-feeder direction-cycle test |
| `0x20` | Environmental fixture comparison, including waits for external reference data |
| `0x21` | Nonreturning hub direction-cycle test |
| `0x22` | Finite two-reader lane-0 detection/read test, followed by the buffer diagnostic if it returns |
| `0x23` | Nonreturning buffer-driven hub diagnostic |
| `0x24` | Alternate slot-range endurance engine |
| `0x25` | Explicit TEST-exit cleanup followed by standalone automatic-feed mode, not just an idle return |
| `0x26` | Whole fixture test, including the raw JSON report |

Direction numbers refer to native drive settings, not an independently verified harness polarity.

In **1.1.3**, the recognized selector set is `0x00`, `0x01`, `0x02`, `0x06`, `0x07`, and `0x0B`–`0x22`; 1.5.2 extends that range through `0x26`. Legacy `0x22` runs a two-reader endurance test and then a permanent summary loop, rather than the newer finite reader test followed by a buffer diagnostic. Legacy `0x23`–`0x26` are unsupported, but, like every unsupported nonzero selector, still enter TEST before case selection. Do not apply the complete 1.5.2 behavior table to the legacy selectors merely because their numbers match.

The RAM console selector is separate from persistent `0x50`. Calling the runtime mode-`0x00` dispatch path does not itself perform explicit TEST-exit cleanup. Do not model writing a mode byte as a generic state setter. 1.1.3 has the same one-byte `0x0C`/`0x50` setting, but not the 1.5.2 `0x5D` structured self-test. See [maintenance](maintenance.md), its [environment fixture exchange](maintenance.md#environment-fixture-exchange), and [storage](storage.md).

## Buffer mode and independent background controls

### Raw buffer state

`0x05` returns one active-low packed byte: bit `0x01` is PD8; bit `0x02` is PD9.

| Value | Inputs | Native host label | Consequence when entering PRINT |
|---|---|---|---|
| `0x00` | Neither low | middle | No immediate motor decision. |
| `0x01` | PD8 low only | full | Stop the hub. |
| `0x02` | PD9 low only | empty/free | Start the hub in feed direction at configured speed. |
| `0x03` | Both low | No normal label | No immediate motor decision; preserve this distinct observation. |

The inputs are read separately, not as an atomic snapshot. Movement waits require ten consecutive matching samples, normally separated by 1 ms. Target index `0x00` means raw `0x02` (free); target index `0x01` means raw `0x01` (full). The target index is internal, not another wire buffer enum.

`0x04 [slot, 0x00]` enters PRINT; `slot` must be one of `0x01`, `0x02`, `0x04`, `0x08`. It enables the two buffer falling-edge interrupts, resets the free-event count and applies the initial decision above. In 1.5.2 it also begins selected-slot usage accounting.

`0x04 [slot_or_0x00, 0x01]` disables buffer mode. The zero selector restores the previous slot selection; it is not a multi-slot enable. This path clears buffer-active bookkeeping, masks both interrupts, stops the hub, clears operation state/indicator, and finalizes usage accounting in 1.5.2. Both actions reject PRELOAD and RELOAD with status `0x03`.

Once enabled, a PD9 falling edge starts feed and records a free event; PD8 falling stops the hub and clears the active/count fields. The callbacks do not first validate the other input, current operation state, or a host heartbeat. A separate normal-UART watcher masks buffer interrupts when it **observes** a transition away from PRINT; writing a new state is not an atomic interrupt-disable operation, and this watcher itself is not a hub stop.

### `0x0D`: extended RFID/preload workflow

Payload is `[slot_mask, action]`; this is a low-four-bit mask, unlike movement’s one-hot selector.

| Action | Effect |
|---|---|
| `0x00` | **Disable** the extended background RFID workflow. A nonbusy scheduler is reset to its initial phase; ongoing work is not universally cancelled. |
| `0x01` | **Enable** the extended workflow; normal boot enables it after factory-mode dispatch returns. |
| `0x02` | Forced per-slot mechanical preload with temporary extended-workflow flag `0x01`, permitting the later RFID/margin phase. Can retract existing filament and move motors. |
| `0x03` | Forced mechanical pretension with temporary extended-workflow flag `0x00`; not an RFID scan. Also motor-moving. |

The native host labels these CLOSE, OPEN, RUN and TIGHT respectively. Disabling the extended path can skip later RFID/margin work after the current mechanical helper, not all insertion handling or every queued preload. **Both forced actions run the initial all-slot readiness/retract path independently of the requested mask, even mask `0x00`.** That path permits IDLE or PRELOAD and rejects other operation states with `0x03`; all-slot material absence returns `0x16` in 1.1.3 versus `0x18` in 1.5.2. Only the later per-slot work is mask-selected. It can wait for an existing preload, then process selected present slots. A per-slot wait has a nominal initial 5 s plus up to 180 one-second waits; expiration is not necessarily propagated as failure or used to force the preload phase inactive. Empty success therefore does not prove all forced background work finished. Passive metadata queries are `0x02` and `0x03`, not forced `0x0D` operations.

### `0x0F`: automatic preload inhibit

Payload `0x00` **enables automatic preload scheduling**; payload `0x01` **inhibits it**. The application initializes this byte to `0x00`. It is independent of the `0x0D` extended-workflow flag and of the separate print-fault suppression latch. Native “tighten up” terminology does not change these byte semantics. There is no immediate motor-stop or buffer-interrupt call in this handler.

## Movement contract and shared prerequisites

Movement selectors are one-hot `0x01/0x02/0x04/0x08` for A/B/C/D. Only `0x11` also accepts `0x00` as an all-connected-slot sentinel. A mask such as `0x03` is invalid for these commands. Length validation is exact; counts below are **payload bytes**, excluding framing. All movement requests still need the request-mode byte and CRC described in [transport](transport.md).

Feed preparation used by `0x10/0x00`, `0x12/0x00` and `0x13` requires IDLE and no pretension phase. It then selects RELOAD, resets debounce counters, and requires five selected-material active-low samples within up to thirty 1-ms iterations. Missing material yields `0x08`. It retracts other slots whose connection input is active, stopping on the first failure. Thus preparing one slot can move another slot and leave partial progress; it is not a read-only readiness check.

The common material-to-hub phase starts feeder and hub, waits for ten consecutive active connection samples within roughly 7 s, then requires about 20 firmware-mm of feed-direction movement within a roughly 2 s polling window. Relevant errors are `0x05` and `0x09`. Successful intermediate helpers can intentionally leave drives running for the next phase.

The encoder is signed **big-endian IEEE-754 binary32** in wire telemetry. Feed normally decreases position, so feed travel is start minus current; retract normally increases it. The firmware scale is approximately 0.000788641337 firmware-mm per count. “Firmware-mm” records its numerical convention, not independent calibration or proof against slip. See [sensors](sensors.md).

### `0x10`: eight actions, not eight identical ACKs

Request payload is exactly three bytes: `[onehot_slot, action, unsigned_arg]`. The last byte is an integer, not a float. Use `0x00` when a stage does not use it.

| Action | Operation and guard | Successful/terminal response |
|---|---|---|
| `0x00` | Full feed preparation above, then a hub hold/stop sequence. Enables movement-window recovery during the hold. | Status=result; **one byte containing post-finalization operation state**. |
| `0x01` | Set RELOAD; hub feed/settle. No common feed-readiness guard. | Status=result; empty payload after the routine returns. |
| `0x02` | Set RELOAD; wait for initial hub progress, then feed/settle. No common feed-readiness guard. | Status=result; empty payload. |
| `0x03` | Clear state/error/indicator through successful finalization. | Status `0x00`, empty. **No hub-stop or feeder-release call**; not cancellation. |
| `0x04` | Set RELOAD; acknowledge **before** starting the private feed worker. | First status `0x00`, empty = acceptance only. Subsequent action `0x05` telemetry and action `0x06` completion have different roles, below. |
| `0x05` | Result/encoder query, with different private and normal-dispatch behavior. | Four-byte BE binary32 position; status meaning depends on which path handled it. |
| `0x06` | Standalone: set RELOAD; check/fill to full buffer. During action `0x04`: end its travel loop and enter its buffer check. | Status `0x00` or `0x0D`, empty, after buffer checking. |
| `0x07` | Set RELOAD; short relative feed using the unsigned argument byte as distance target. | Actual result `0x00` or `0x0E`, empty; **operation state is cleared even on `0x0E`**. |

Unsupported actions `0x08`–`0xFF` log an argument error but have no guaranteed response. A normal command CRC error uses status `0x02`; this must not be confused with an operation-state value or a distance.

Details relevant to deadlines and completion:

- **Action `0x00` hold:** drive at configured hold speed, enable overload monitoring, request a 3.2 s delay, then wait in 10-ms increments while the reverse-detooth-active flag remains set. That final wait has no independent deadline. It is **not** a wait for PRELOAD state to clear. Reset overload history, disable that monitor and stop the hub only after the wait exits.
- **Action `0x01` settle:** up to 1250 samples at requested 20-ms intervals; 25 consecutive samples with less than 0.01 firmware-mm feed progress end it successfully. Feed travel strictly above 3000 firmware-mm yields `0x0B`. Exhausting the window also returns success, so success is not a target-distance guarantee.
- **Action `0x02` initial progress:** nominally three one-second samples, but only the first two can qualify the 5 firmware-mm threshold in the observed branch. Failure stops the hub and returns `0x0A`; qualifying progress enters action-`0x01`-style settle.
- **Standalone action `0x06`:** a fifteen-sample full-buffer precheck can succeed without issuing any start **or stop**. Otherwise drive at configured feed speed, wait up to 3000 full-buffer samples, stop and return `0x00` or `0x0D`.
- **Action `0x07`:** up to fifteen 20-ms samples at its configured speed. Travel strictly greater than the byte target succeeds early. At the final sample, travel at least target minus 0.5 firmware-mm succeeds; less yields `0x0E`. It is a short mechanical stage, not a long-distance guaranteed move.

#### The action `0x04` private exchange

This sequence requires the host receiver, printer filament sensor and operation coordinator to remain active concurrently.

```mermaid
sequenceDiagram
    participant H as Host operation coordinator
    participant C as CFS application
    participant P as Printer local filament sensor / extruder
    H->>C: 0x10 [slot, 0x04, 0x00]
    C-->>H: 0x10 status 0x00, empty (accepted)
    loop Until local sensor handoff or failure
        H->>C: 0x10 [slot, 0x05, 0x00]
        C-->>H: 0x10 status 0x00 + BE float position
    end
    P-->>H: Local filament-arrival event
    H->>P: Local handoff motion, with host-selected heater/interlock policy
    H->>C: 0x10 [slot, 0x06, 0x00]
    C-->>H: 0x10 status 0x00 or 0x0D, empty (buffer result)
    H->>C: 0x10 [slot, 0x05, 0x00] (serialized normal result check)
    C-->>H: 0x10 latched status + BE float position
    Note over H: Advance only with direct 0x06 success and zero checked result; unknown blocks
    H->>P: Coordinated local extrusion / buffer checks
    H->>C: 0x10 [slot, 0x07, distance_byte]
    C-->>H: 0x10 status 0x00 or 0x0E, empty
```

During action `0x04`, normal synchronous dispatch is occupied. Its private reader consumes the **same UART** and recognizes only fixed-size action-`0x05` and action-`0x06` candidates addressed to the actual device UID/address. It reads `0xF7`, delays about 2 ms, and requests exactly eight more bytes. It checks address, command `0x10`, and CRC, then recognizes the action byte; unlike normal dispatch it does not validate the selector or length byte. Continue to send valid complete nine-byte frames with the same slot and normal length. A short fragmented read can be discarded; arbitrary interleaved traffic is not a harmless concurrent poll.

The private reader does not execute other commands, including action `0x03`; rejected candidates can trigger a drain of up to three 128-byte reads. **Reserve the bus for this exchange**, rather than allowing background status/address/metadata requests to be consumed by it. The code contains a private status-`0x02` send branch, but the actual reader only forwards actions `0x05/0x06`; that branch is unreachable in these images. Do not wait for it as reliable rejection feedback.

Private action `0x05` returns the current sampled position with status `0x00` and keeps feeding. It does not read or clear the result latch. Before private action `0x06`, the worker stops and returns internally on:

| Result | Condition |
|---|---|
| `0x0A` | Private action `0x06` never arrived within 1250 iterations. |
| `0x0B` | Feed travel strictly exceeded 3000 firmware-mm. |
| `0x0C` | Twenty-five consecutive slow samples (nominally 500 ms). |

These early failures are **latched without an unsolicited final frame**. Once normal dispatch resumes, action `0x05` returns the latched status with a newly sampled position, then clears a nonzero result latch after its send attempt. The position is not a saved failure endpoint. A previous successful private telemetry response does not preclude a later failure; a lost normal result reply can lose the only nonzero latch observation.

Private action `0x06` is checked before that iteration’s travel/stall guards. It retains the existing hub drive while waiting for ten consecutive full-buffer samples, within 3000 samples; then it stops the hub and sends status `0x00` or `0x0D`, empty. **Both branches return zero internally**, so the caller subsequently clears ERROR and stores result zero. A later normal action `0x05` cannot recover a missed `0x0D`. The host must retain the direct completion frame separately from telemetry and later state.

An empty action-`0x06` reply alone cannot distinguish private completion from standalone buffer fill: action `0x04` may already have failed with `0x0A/0x0B/0x0C`, leaving normal dispatch to execute the incoming action `0x06`. Before further engagement or PRINT, the operation coordinator must serialize a normal action-`0x05` result check and exclusively own all consumptive result-latch reads throughout the exchange. A nonzero result or unknown/lost result blocks advancement. A zero result is usable only with that uninterrupted latch ownership and a retained successful direct action-`0x06` frame; it never recovers a lost completion or overrides a direct `0x0D`.

### `0x11`: release buffer or retract to material path

Request payload is exactly `[onehot_slot_or_0x00, action]`. The guard permits IDLE or PRELOAD and requires material presence. Busy returns `0x03` without finalization. Missing selected material returns `0x16`; all-slot absence in 1.5.2 returns `0x18` but latches `0x16` in the operation state. 1.1.3 returns and latches `0x16` for that all-slot case, while its indicator receives `0x18`.

| Action | Mechanics | Completion |
|---|---|---|
| `0x00` | Feeder inverse/detooth pulse, hub reverse, wait for free buffer raw `0x02` with ten consecutive samples within 5000 samples; stop hub. | Empty payload, status `0x00` or timeout `0x13`. This does not prove the filament cleared the connection sensor. |
| `0x01` | Pull back using encoder/block checks, then qualify connection input clear. In 1.5.2 a usage-accounting hook runs **before** the readiness guard. | Empty payload; propagated result. |

Action `0x01` rejects a processed slot whose material input is high (`0x17`). Its odometer phase has up to 2000 requested 20-ms iterations. It can finish on qualified connection-clear or a long low-progress/nonblocked interval; on a slow sample a latched motor mask yields `0x19`; exhaustion yields `0x1A`. Success in this internal phase can leave motion active for the following sensor-clear phase. That phase waits for ten consecutive raw connection-high samples within about 7 s, adds a 300-ms hold on its ordinary success path, then stops/releases. Timeout yields `0x14`. An already-clear fast path can omit the hold. The first retract odometer delta depends on an incoming floating-point register rather than a locally established initial sample; subsequent samples establish their local baseline. Do not turn this native defect into a promised first-sample measurement.

Selector `0x00` scans A–D with active connection inputs, in order, and stops at the first failing slot. Earlier slots are not rolled back. An internal processed-slot mask is not returned. Single-slot processing does not have that initial connection-presence skip. Unsupported actions are not usable: the aggregate no-active-slot case can return success without validating the action.

### `0x12`: prepare or relative-distance feed (1.5.2 only)

Exactly five payload bytes: `[onehot_slot, mode, speed_byte, distance_hi, distance_lo]`. Distance is unsigned **big-endian 16-bit**.

- Mode `0x00`: common readiness/material-to-hub preparation, **without** action-`0x00`’s hold sequence. Finalize and return status=result plus one post-finalization state byte.
- Mode `0x01`: select RELOAD and feed at the configured hold/feed speed. The wire `speed_byte` is logged but **ignored** by the movement body. Clamp target above 3000 firmware-mm to 3000. Check up to 1250 samples at requested 20 ms, stop when travel strictly exceeds target or after 100 slow samples, then stop. The observed function returns `0x00` even for slow/no-progress or full-window termination: empty success is not distance verification.
- Other modes log an error and do not guarantee a response.

There is no corresponding `0x12` dispatch in 1.1.3. Do not infer support from an address match or a host version setting.

### `0x13`: preparation plus settle

Exactly one payload byte: `[onehot_slot]`. Run readiness, material-to-hub preparation and hub feed/settle in order; short-circuit on error, finalize and send the propagated status with empty payload. This is not an encoder getter. The recovered native host’s legacy `extrude2_process` builder sends **two** bytes; that builder is not a valid template for the one-byte application contract in either mapped image. See [compatibility](compatibility.md).

### `0x07`: hub drive is not printer extrusion

Payload is one action byte: `0x00` stop hub, `0x01` feed-direction drive, `0x02` reverse-direction drive. There is no slot or speed in this request. Speed is taken from live firmware configuration. All actions, **including stop**, require IDLE; otherwise status `0x03` includes one current-state byte. Successful replies are empty ACKs to a drive-state change, not movement completion. These commands do not control the printer’s local extruder, hotend, cutter or local filament sensor.

The related feeder command `0x06` is not a usable arbitrary-slot control interface in either mapped image: it inherits the receive tick as selector, and accepted decoding is then overwritten by the current-slot mirror. `0x09` accepts two data bytes but consumes a stale third parser byte. Do not repair either defect by inventing a wrapper, widening the payload or describing them as safe fallback stops.

## Timing and protection are not a safety channel

Normal UART handlers are synchronous. A long command can delay normal polling and the PRINT-exit observer. Buffer IRQs and background workers can still act; periodic motor callbacks can recover, stop or detooth independently. There is no general wire cancellation token in these movement loops.

Both versions have motor-block/overload protection, but its timing is requested-delay/sample based, not a certified elapsed-time bound. A 100-ms service can attempt recovery after sustained blockage and later perform all-motor cleanup/write a fault snapshot. Recovery includes real reverse and detooth motion; nominal “release” can pulse a feeder. The motor cleanup does not itself mask buffer IRQs, so a later free edge can restart the hub.

PRINT also has a distinct buffer-progress worker. In 1.5.2 it starts with a 100-count window and requested 100-ms checks, adjusts the window for free events, and below count 50 can temporarily assist with the selected feeder and hub. After ten assist iterations it stops the feeder. At window end, insufficient progress can latch `0x52`; repeated/free-count or larger travel conditions can latch `0x51`; relevant faults stop/mask buffer hardware. A separate suppression latch can inhibit the low-progress fault. **1.1.3 has no 1.5.2 feeder-assist phase**, although related progress/count checks exist. Neither behavior is encoded by `0x0F`.

The native normal `0x0A` PRINT consistency check can also report `0x50`. This sets a separate low-progress-suppression flag and indicator; it does **not** select ERROR, populate the operation error latch, stop motors, or mask buffer interrupts. That flag suppresses the PRINT worker’s low-progress `0x52` path described above; do not treat `0x50` as equivalent to that worker fault or to a physical stop.

The application heartbeat worker is also not an emergency stop. While its command-busy flag is set it sleeps and does not perform the timeout path. Otherwise it waits on a semaphore with a 60000-tick argument and updates online/indicator bookkeeping; the offline path has state-specific indicator handling, not a universal stop. Heartbeat loss during a long command can therefore be deferred indefinitely. Host timeout, host disable, ERROR, IDLE and “no longer online” must never be used as substitutes for verified physical stop.

## Implementation limits

The documented branch behavior does not establish calibrated travel, fitted hardware, electrical stop performance, or live interoperability. Only the command paths described above are part of this movement contract; do not infer additional reachable actions from similar internal mechanics.

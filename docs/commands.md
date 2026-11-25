# Main-application command reference

This is the wire reference for the mapped CFS **1.5.2** application, with explicit **1.1.3** differences. It describes native behavior, including defects; it is not a promise that every recognized operation is suitable for an integration. Use [transport](transport.md) for framing, [addressing](addressing.md) for identity, [modes and workflows](modes-and-workflows.md) for movement sequencing, and [maintenance](maintenance.md) for high-risk persistent/factory/update operations.

**Count:** 1.5.2 recognizes **31 ordinary application function codes**: `0x01..0x15`, `0x30`, `0x50..0x53`, `0x55..0x56`, `0x5B..0x5D`. The four separately intercepted address codes `0xA0..0xA3` bring the combined application/address surface to **35**. Version 1.1.3 recognizes 27 of the ordinary codes; it lacks `0x12`, `0x5B`, `0x5C`, `0x5D`. `0x54` is unsupported in both, not a missing thirty-second command.

## Conventions and common failures

All numeric wire values, payload offsets, lengths `L` and selector values below are hexadecimal. Human byte counts are explicitly marked as counts. A request frame is:

```text
0xF7 | destination | L | request_mode | function_code | payload | CRC8
L = payload byte count + 0x03
```

Ordinary native application requests use request mode `0xFF`; addressing uses `0x00`. The reply's corresponding byte is a **status**, not an echo of request mode. Unless explicitly noted, the reply uses the same function code and the current local address. “Empty ACK” means status `0x00`, no payload, `L = 0x03`; it does not establish physical completion.

- Accepted destinations are current address, quiet `0xEA` and broadcast/fixture `0xEB`. `0xFE` works only while it is the current unassigned address. Quiet behavior and exceptions are in [addressing](addressing.md).
- Ordinary handlers require **exactly** their documented request `L`. A mismatch sends the supplied error function code, status `0x01`, empty payload, without invoking that handler. `0x53` has a wrong-code error exception described below.
- Invalid ordinary CRC sends the received function code, status `0x02`, empty payload, except for quiet `0xEA`.
- Unsupported ordinary codes, including `0x00`, `0x16..0x2F`, `0x31..0x4F`, `0x54`, `0x57..0x5A` and other unlisted values, produce status `0x01`, empty payload, except quiet `0xEA`. `0xA0..0xA3` are intercepted separately, not unsupported ordinary commands. Loader code `0xF0` is not a normal-application update alias.
- Status `0x03` is contextual: state/busy guards in motor paths, but **unchanged stored value** for `0x50`. Never globally convert it into success.
- Several operations intentionally send no success response. Silence is not interchangeable with a timeout failure or a successful ACK.
- Long handlers run synchronously in the normal UART worker. A command arriving during a long operation is not guaranteed to be processed, including a stop-looking command.

### Slots, masks and sentinel

| Logical slot | One-hot selector / mask bit |
|---|---|
| A | `0x01` |
| B | `0x02` |
| C | `0x04` |
| D | `0x08` |

A slot mask and a one-hot selector are different types:

- `0x02`, `0x03`, `0x0D` use low-four-bit **masks**; `0x0F` means all four slots.
- `0x04`, `0x06`, `0x0B`, `0x10`, `0x11`, `0x12`, `0x13` use the one-hot helper. Only `0x00`, `0x01`, `0x02`, `0x04`, `0x08` can pass that helper, with zero rejected where stated. It does not choose the first bit of a multi-bit mask.
- The helper maps `0x00` to an internal slot-four sentinel. That is not a fifth physical slot. `0x04` disable and `0x11` have specific zero-selector semantics; other command paths must not invent them.

## Complete ordinary command table

`N` is request payload byte count, not frame length. `N` and `L` are hexadecimal in this table. Reply descriptions assume a valid envelope and exact request length; common failures still apply.

| Code | Purpose | N / L | Request payload | Reply and operation-specific behavior |
|---|---|---|---|---|
| `0x01` | `CREATE_CONNECT` | `0x00 / 0x03` | Empty | Empty ACK, then refresh indicator/status selection from material inputs. No address assignment or full subsystem reinitialization. |
| `0x02` | Cached RFID records | `0x01 / 0x04` | `[slot_mask]` | Status `0x00`, selected `A:<value>;` through `D:<value>;` records in ascending slot order. Empty selection returns empty payload. No motor-driven acquisition is started. Grammar below. |
| `0x03` | Cached remaining percentage/state | `0x01 / 0x04` | `[slot_mask]` | Status `0x00`, exactly four bytes A–D. Unselected/busy/unavailable positions are `0xFF`; absent material is `0x00`. Present without valid RFID is `0xFF` in 1.5.2 but can be `0x64` in 1.1.3. |
| `0x04` | Buffer print control, native `SET_BOX_MODE` | `0x02 / 0x05` | `[onehot_slot_or_zero, action]` | Action `0x00` enables print-buffer control, requires a real slot and can start the hub. Action `0x01` disables IRQ control, stops hub and completes usage accounting; zero restores the previous slot. Empty ACK. Operation states `0x01`/`0x03` reject with status `0x03`, empty payload; bad selector/action gives `0x01`. |
| `0x05` | Buffer endpoint field | `0x00 / 0x03` | Empty | Status `0x00`, one byte: active-low PD8 at bit `0x00`, PD9 at bit `0x01`. No busy-state guard. Preserve all four codes `0x00..0x03`. |
| `0x06` | Intended feeder-drive control | `0x02 / 0x05` | Intended `[onehot_slot, action]` | **Broken selector path; not a usable arbitrary-slot API.** Lower actions `0x00` release all feeders, `0x01`/`0x02` drive selected feeder in configured directions/speeds. State guard rejects even release with status `0x03`, empty payload. Bad selector/action gives `0x01`; otherwise empty ACK. Both defects below apply before any promise about the requested slot. |
| `0x07` | Hub/connection motor control | `0x01 / 0x04` | `[action]` only | `0x00` stop hub, `0x01` direction zero, `0x02` direction one, using configured speeds. **No slot or speed byte.** Any nonzero operation state rejects before action decoding with status `0x03` and one current-state byte, even for stop. Invalid action gives `0x01`, empty; success empty ACK. |
| `0x08` | Slot input bank | `0x01 / 0x04` | `[bank]` | Bank `0x00` MATERIAL, `0x01` CONNECTIONS. Status `0x00`, one active-low A–D mask; upper four bits zero. Invalid bank gives `0x01`, empty. No operation-state guard. |
| `0x09` | Intended speed configuration | `0x02 / 0x05` | `[motor_kind, direction]` | **Defective:** accepts two payload bytes but consumes a stale third byte as speed. Valid selectors `0x00`/`0x01` each give status `0x00` plus one `0x00` byte; invalid selector gives `0x01`, empty. A three-byte request is rejected by exact-length checking. Do not expose as a working speed setter. |
| `0x0A` | Box state / event poll | `0x00 / 0x03` | Empty | Normally status `0x00` plus six bytes in 1.5.2, four in 1.1.3. Error status can accompany the normal layout. Status `0x30` instead returns four slot-event bytes. Reading consumes pending event information and can trigger sensor-consistency error handling. Full layouts below. |
| `0x0B` | Indicator pattern; requested slot is not honored | `0x02 / 0x05` | `[nonzero_onehot_slot, pattern]` | The slot byte is validated, but an alias defect replaces the decoded slot with CURRENT. Selected-slot patterns therefore act on CURRENT, not the slot in the request. Empty ACK; invalid/zero selector gives `0x01`, empty. Defined patterns and unsafe undefined values are listed below. Not the loader's one-byte `0x0B` start command. |
| `0x0C` | Read operating-mode parameter | `0x00 / 0x03` | Empty | Status `0x00`, **one RAM parameter byte** loaded from internal-flash parameter ID `0x06`. Refreshes its cached mode byte if different. Not EEPROM, a six-byte record, an identity mode, or flash readback of a preceding write. |
| `0x0D` | Extended-workflow gate / forced pretension | `0x02 / 0x05` | `[slot_mask, action]` | `0x00` disables extended background RFID work; `0x01` enables it. Forced `0x02` RUN permits extended RFID/margin work; `0x03` TIGHT performs mechanical pretension with that phase disabled while its temporary flag stays zero. The initial all-slot readiness/retract is independent of the mask, even mask zero. Outside IDLE/PRELOAD returns `0x03`; all-slot absence is `0x16` on 1.1.3, `0x18` on 1.5.2. Neither forced action guarantees a fresh read. |
| `0x0E` | Encoder reset/read | `0x01 / 0x04` | `[action]` | `0x00` resets baseline/cache and empty ACK; `0x01` returns status `0x00` and **four-byte big-endian IEEE-754 binary32 signed position**. Invalid action gives `0x01`, empty. No busy-state guard. |
| `0x0F` | Automatic preload inhibition, native `TIGHTEN_UP_ENABLE` | `0x01 / 0x04` | `[value]` | `0x00` ENABLE automatic preload, `0x01` INHIBIT it. Stores a separate automatic-preload gate; empty ACK. Other values give `0x01`, empty. Not the print-check fault-report-suppression flag. |
| `0x10` | Multistage feed | `0x03 / 0x06` | `[nonzero_onehot_slot, action, argument_byte]` | Actions `0x00..0x07` have distinct layouts and completion rules below. Invalid/zero selector gives `0x01`, empty. Action at least `0x08` logs an argument error and sends no guaranteed response. Argument is an unsigned byte, not a float. |
| `0x11` | Retract/release workflow | `0x02 / 0x05` | `[onehot_slot_or_zero, action]` | Action `0x00` releases toward buffer free target; `0x01` retracts using encoder and connection input, with a usage-accounting hook before the readiness guard. Zero scans active CONNECTIONS slots, not an unconditional all-motor stop. Result status with empty payload. State guard accepts only IDLE/PRELOAD; other states give `0x03`. Missing material can give `0x16`/`0x18`. Unsupported action gives `0x01` when a slot is processed, but all-slot/no-active-path can return `0x00`. |
| `0x12` | Prepare / distance feed, 1.5.2 only | `0x05 / 0x08` | `[nonzero_onehot_slot, mode, speed_byte, distance_hi, distance_lo]` | Mode `0x00`: preparation result plus one operation-state byte. Mode `0x01`: distance feed, result with empty payload. Distance is **unsigned big-endian 16-bit**; speed byte is ignored by movement code. Other modes log only, no guaranteed response. Invalid selector gives `0x01`, empty. |
| `0x13` | Prepare/feed/settle, native model 2 | `0x01 / 0x04` | `[nonzero_onehot_slot]` | Run readiness, material-to-hub preparation and hub feed/settle; stop at first failure, otherwise finish. Result status, empty payload. Invalid selector gives `0x01`, empty. May block for sensor/motion waits. |
| `0x14` | Version and serial | `0x00 / 0x03` | Empty | Status `0x00`, exactly 22 bytes: ASCII `152` (or `113`) followed by 19 serial-cache bytes. No delimiter or guaranteed printable/NUL-terminated serial. |
| `0x15` | Hardware boot-fault categories | `0x00 / 0x03` | Empty | Status `0x00`, 16 bytes: six category-prefixed fault bytes, then ten zero/reserved bytes. Decode category and fault separately; raw nonzero does not mean fault. Layout below. |
| `0x30` | Display backlight | `0x01 / 0x04` | `[value]` | PB0 high only for `0x01`; every other byte drives it low. Empty ACK. Not an RF power operation or display-content command. |
| `0x50` | Attempt mode commit and reset | `0x01 / 0x04` | `[mode]` | Same RAM value: status `0x03`, empty, no reset. Changed value: update parameter ID `0x06`, attempt an unchecked flash commit, send status `0x00`, empty, then reset. Neither ACK nor a pre-reset `0x0C` verifies persistence. `0xEA` suppresses its reply; `0xEB` temporarily makes reply source `0xEB`. After reset, rediscover the same UUID and verify loaded mode/state before motion. See [maintenance](maintenance.md). |
| `0x51` | Fixture direct-output flag | `0x01 / 0x04` | `[value]` | Only incoming destination `0xEB` stores the flag. No success ACK, including at other accepted destinations where it does not update the flag. Not a normal unsolicited fault message merely because its code equals a status value. |
| `0x52` | Cached environment query / fixture prompt | `0x00 / 0x03` | Empty for a host query | Query success is **code `0x53`**, status `0x00`, `[temperature_byte, humidity_byte]`; bad length uses code `0x52`, status `0x01`, empty. In the [environment fixture exchange](maintenance.md#environment-fixture-exchange), the MCU instead emits empty `0x52`/status `0xFF` prompts. A prompt must not satisfy a cache-query wait. |
| `0x53` | Fixture environment reference | `0x02 / 0x05` | `[temperature_byte, humidity_byte]` | Store fixture references, no success ACK; both bytes must be nonzero to release the fixture's unbounded reference wait. Does not update the live-sensor cache. Bad-length error incorrectly uses code `0x52`, status `0x01`, empty. See [environment fixture exchange](maintenance.md#environment-fixture-exchange). |
| `0x55` | Communication test | `0x01 / 0x04` | `[value]` | Status `0x00`, one byte `(value + 0x01) modulo 0x100`. No movement. Native host has a special receive shortcut; a replacement should still validate envelope, code and status. |
| `0x56` | MCU reset | `0x00 / 0x03` | Empty | Empty ACK, then nominal 100 ms delay and reset. Does not establish a permanent loader-hold flag. Reset loses the RAM address. |
| `0x5B` | Reduced hardware identifier, 1.5.2 only | `0x00 / 0x03` | Empty | Status `0x00`, eight bytes: little-endian UID word at `0x1FFFF7E8`, then little-endian `(UID_word_1 AND UID_word_2)`. Not the 12-byte address UUID. |
| `0x5C` | Secure credential write/read, 1.5.2 only | `0x10 / 0x13` | 16 caller-supplied credential bytes | Writes EEPROM logical area `0x60`, returns status `0x00` and 16 credential cache/readback bytes. Low-level write/read failures are not faithfully surfaced; ACK is not proof of durable correct programming. Not arbitrary spool-tag writing. High risk; see [maintenance](maintenance.md). |
| `0x5D` | Factory self-test, 1.5.2 only | `0x00 / 0x03` | Empty | Enters TEST and may move motors, change GPIO modes and wait for fixture input; no ordinary direct completion ACK. Reports include unframed diagnostic JSON on the UART. Not startup health polling. See [maintenance](maintenance.md). |

The separate `0xA0`, `0xA1`, `0xA2`, `0xA3` table in [addressing](addressing.md#address-command-reference) defines every address operation's payload and identity reply. These four are not aliases for application commands.

## Passive data layouts

### `0x02`: RFID cache records

Records are emitted in A–D order for the selected low-four mask bits:

```text
A:<value>;B:<value>;C:<value>;D:<value>;
```

Each value is **either exactly 40 raw bytes** or one of the exact ASCII literals `busy`, `none`, `unknown`. There is no terminal NUL. One ordinary 40-byte record occupies 43 bytes including its letter, colon and semicolon; all four occupy 172 payload bytes. This is an **outbound** response size, not permission to send an oversized request to the main parser.

| Value | Native cache condition |
|---|---|
| `busy` | Acquisition state machine busy for this slot |
| 40 raw bytes | Cached data passes presence/first-byte checks and is not inhibited by a valid secure UID7 record with zero saved life/count |
| `unknown` | Material input active but no available accepted cache record |
| `none` | Material input inactive |

Data bytes are not escaped. Prefer **one-slot queries** when exact attribution matters: total payload length then distinguishes forty-byte data from an exact sentinel. Multi-slot aggregates are not always uniquely parseable because opaque data can imitate sentinel endings and later slot labels. Require a unique full parse in the requested slot order; preserve/reject ambiguous data rather than choosing sentinel-first or record-first. Do not require hexadecimal or UTF-8 data. See [RFID and material](rfid-and-material.md).

The original printer host's delimiter scanner is less defensive. A truncated inner scan can return no parsed result and cause a later exception before any records are applied. It also accepts `busy` as a sentinel but later applies concrete-record color/type slices to it, producing empty strings. These are host bookkeeping quirks, not additional wire fields or recommended handling for a replacement.

### `0x03`: remaining values

The four payload bytes are always A, B, C, D, even for a one-slot mask:

- Unselected slot or acquisition busy for that slot: `0xFF`.
- Selected slot without active material: `0x00`.
- Present material with valid cached RFID: the native cached percentage/state byte.
- Present material without usable RFID: `0xFF` on 1.5.2; 1.1.3 can return `0x64`, indistinguishable here from a genuine 100% result.

This is not four raw byte lengths or four Boolean card-presence flags. Preserve `0xFF` as unavailable; do not silently interpret it as 255 percent or a fresh measurement. The original host stores decimal-string forms even for `0xFF`; that storage policy is not a wire conversion rule.

### `0x05` and `0x08`: active-low masks

`0x05` returns `(not PD8) | ((not PD9) << 1)`:

| Buffer code | Meaning |
|---|---|
| `0x00` | Neither endpoint low; native host calls it middle |
| `0x01` | PD8 low only; full endpoint |
| `0x02` | PD9 low only; free endpoint |
| `0x03` | Both low; retain explicitly, do not normalize to middle/free/full |

`0x08` returns one bit per slot A–D, with a set bit meaning the GPIO is low. Bank `0x00` is MATERIAL (PD7, PD6, PD5, PD4), bank `0x01` CONNECTIONS (PE10, PE9, PE8, PE7). The bank is in the **request**; the reply does not repeat it. No extra independent shared gate/presence sensor is encoded here. Board correlation and physical-placement limits are in [sensors](sensors.md).

### `0x0A`: normal and error-state layout

Branch on **reply status and exact payload shape before decoding**. Normal-layout payload:

| Offset | 1.5.2 field | Encoding / behavior |
|---|---|---|
| `0x00` | Temperature | Signed two's-complement 8-bit, cached integer degrees Celsius |
| `0x01` | Humidity | Unsigned 8-bit, cached integer percent |
| `0x02` | Pending event/stage byte | A high-priority pending byte is ORed with `0x80` and cleared; otherwise an ordinary pending byte is used. The ordinary source is cleared after the read. Preserve raw value; it is not the operation enum or the four-slot event layout. |
| `0x03` | Operation state | Runtime enum below |
| `0x04` | CONNECTIONS mask | Same active-low low-four mask as `0x08` bank `0x01` |
| `0x05` | Secure-tag depletion/cache-inhibit mask | A–D bits are set only with debounced presence, a valid usage record, saved UID length seven and saved life/count zero. Not scan-pending, raw presence, remaining percent or acquisition completion; a clear bit proves none of those. |

Version 1.1.3 has **only offsets `0x00..0x03`**, four payload bytes. Do not synthesize the two missing masks. Both versions have the separate four-byte status-`0x30` event layout.

| Operation value | Native state name / meaning |
|---|---|
| `0x00` | `STATE_IDLE` |
| `0x01` | `STATE_PRELOAD` |
| `0x02` | `STATE_PRINT`, buffer-controlled feed/accounting |
| `0x03` | `STATE_RELOAD` in the mapped state logger; feed/retract movement workflows also use it (some host/earlier descriptions say RETRACT) |
| `0x04` | `STATE_ERROR`, with a latched operation result |
| `0x05` | `STATE_TEST` |

The response status is normally `0x00`. In operation ERROR it is the latched result, **with the normal-size payload still present**. In PRINT a sensor-consistency failure can return `0x50`, set the separate low-progress-report suppression latch and update the indicator. That reachable path does **not** enter ERROR, latch `0x50` as the operation error, stop the hub or mask buffer IRQs. It can suppress a later worker `0x52` report. Polling is therefore not entirely side-effect-free, and status `0x50` is not stop evidence. The helper does not produce the `0x53` value tested by another branch; do not invent that error path.

The native host's names are not a complete wire decoder. Host A reads temperature, humidity and operation state; host B additionally stores offset `0x05` as `slot_rfid_scrap`. They ignore the event byte; B also ignores the CONNECTIONS mask in this handler. Both host decoders treat humidity as signed before range checking, although the firmware producer stores unsigned humidity. A replacement should preserve the native unsigned field and diagnose out-of-range values explicitly. B has no automatic four-byte compatibility fallback; filesystem A/B names do not prove which host/application combination ran.

Environment fields are a cache. The mapped 1.5.2 environment worker attempts updates once per second only outside TEST and after successful AHT startup. A boot sensor fault or TEST freezes the cache; later failures can retain prior values. Neither `0x0A` nor `0x52` guarantees a fresh I2C measurement or supplies a reliable freshness bit.

### `0x0A`, status `0x30`: four independent slot events

This replaces the normal layout. Payload is exactly `[event_A, event_B, event_C, event_D]`, each a whole byte:

| Event byte | Native host interpretation |
|---|---|
| `0x00` | Unchanged |
| `0x01` | Insert |
| `0x02` | Extract |
| `0x03` | Completed preload/workflow transition |

These are not packed two-bit fields, temperatures or an operation-state tuple. The poll consumes pending event flags/remembered transitions, so preserve each received event; a later query need not repeat it. `0x03` means the reported slot transition, not completion of every pending command or printer-side tool change. Unsupported event values should remain visible as unknown raw data, not be silently reclassified.

The original host routes status `0x30` outside its ordinary success parser, applies slot events in A–D order and schedules saved-state/same-material notifications. It can return a host-level success Boolean for this **nonzero wire status**. That Boolean does not change the wire layout.

### `0x0E`: binary32 position

Action `0x01` yields four bytes in **big-endian IEEE-754 binary32** order. Position is signed and can be negative. It is not an integer count, a little-endian float or a distance that should be silently negated. Firmware converts signed encoder-count delta using binary32 scale bits `0x3A4EBCD3` (approximately 0.000788641337 firmware-mm/count). Related motion logs use millimetres, but roller slip and physical calibration are not measured. Usage accounting has an opposite-sign coordinate internally; that does not change wire `0x0E`.

### `0x14`: version and serial

The first three bytes are ASCII digits, for example `[0x31, 0x35, 0x32]` for 1.5.2, not a binary semantic-version triple `[0x01, 0x05, 0x02]`. The next 19 bytes are the serial cache. Select a known application profile from identity/version evidence; do not auto-detect 1.1.3 solely because an arbitrary `0x0A` frame has four data bytes, since status `0x30` has four on both versions.

### `0x15`: category prefixes and fault values

The producer zeroes a 16-byte response, then fills the first six bytes with `category_prefix OR native_fault_byte`. The expected low-nibble fault values must be separated from the high-nibble category:

| Offset | Category prefix | Category |
|---|---|---|
| `0x00` | `0x10` | EEPROM |
| `0x01` | `0x20` | Temperature/humidity startup |
| `0x02` | `0x30` | Encoder/measuring wheel startup |
| `0x03` | `0x40` | Buffer startup |
| `0x04` | `0x50` | Left RFID reader startup |
| `0x05` | `0x60` | Right RFID reader startup |
| `0x06..0x0F` | `0x00` | Zero/reserved in both mapped producers |

Thus healthy category bytes can be `[0x10, 0x20, 0x30, 0x40, 0x50, 0x60]`; **raw nonzero is not a fault test**. Preserve category and fault separately; do not discard the reserved bytes or claim a separate LCD field is present here.

The earlier printer host tests each of the six raw bytes for nonzero. The later host masks only the first byte with `0x03`, still testing the other five raw bytes. Healthy category prefixes can therefore trigger its error-reporting calls. A replacement should decode the native category/fault fields rather than copy those raw-byte tests; the returned hardware data is the same regardless of whether a particular printer UI displays the alarm.

## Command-specific control and defects

### `0x04` is not the `0x0A` operation enum

The native host calls its two wire actions PRINT (`0x00`) and IDLE (`0x01`). The firmware interprets them as **enable buffer / disable buffer**:

- Enable requires a real selected slot, enters operation state `0x02`, enables PD8/PD9 IRQs, samples buffer endpoints and starts usage accounting. At buffer `0x02` it starts the hub; at `0x01` it stops the hub; the other codes do not establish a new start/stop decision in this branch.
- Disable can use zero to restore a previous slot, disables both IRQs, stops the hub, clears operation state/indicator and finalizes usage accounting.
- Both reject operation PRELOAD `0x01` and RELOAD `0x03` with status `0x03`.

It is therefore neither a pure mode-bit write nor a universal “gear open/close” operation. Its wire action `0x00` differs numerically and semantically from operation IDLE `0x00`.

### `0x06`: two independent selector defects

The requested slot byte is not reliably used in **either** mapped application:

1. The `0x06` branch passes the inherited receive timestamp to the one-hot decoder instead of the request's slot byte. Most timestamps fail validation; an acceptable one still is not the requested slot.
2. Its decoded temporary aliases the helper's CURRENT-selection mirror. The mirror overwrites the result before the command uses it, so dispatch consumes CURRENT.

The alias also affects `0x0B`, but not every other selector-helper caller. Timing a request to obtain a one-hot timestamp does not restore arbitrary-slot selection.

Even lower-layer action `0x00` traverses the broken gate. If reached, it releases **all** feeder drives and can include inverse-direction detooth pulses, not merely an instantaneous PWM-zero write. The lower state guard rejects every action when operation state is nonzero. There is no supported safe wire wrapper for selecting an arbitrary feeder through `0x06` in these builds.

### `0x0B`: CURRENT-slot alias and indicator patterns

Both versions validate the requested nonzero one-hot slot, then overwrite its decoded temporary with CURRENT. Slot-specific patterns use that current selection. A valid ACK does not mean the requested slot was selected.

| Pattern | Meaning |
|---|---|
| `0x00` | Red chase |
| `0x01` | White chase with material-input scan |
| `0x02` | Refresh material-presence white indications |
| `0x03` | CURRENT-slot white breathing |
| `0x04` | CURRENT-slot red, 500 ms on/off |
| `0x05` | CURRENT-slot red held; other slots restored |
| `0x06` | CURRENT-slot red double flash: 200/200/200/1000 ms phases |
| `0x07` | CURRENT-slot red breathing |
| `0x08` | All-red breathing |
| `0x09` | All-white/all-red alternate at 1200 ms |
| `0x0A` | Factory-specific blink with limited selector cases, not a general slot-control pattern |
| `0x0B` | All off |
| `0x0C` | All white |
| `0x0D` | All red |

The firmware does not reject patterns `0x0E` and above. They acknowledge without selecting a defined pattern and can enter a no-delay redispatch loop. Do not probe them or advertise them as additional modes. “White” is also called BLUE in factory labels; the table preserves the native pattern terminology, not a new RGB-setting API.

### `0x09`: accepted two bytes, consumed stale third byte

The dispatcher requires `L = 0x05`, meaning exactly two data bytes. The handler nevertheless reads normalized request offset `0x06`, a **third data position**. CRC is stored separately at normalized offset `0x68`; the extra value is not the transmitted CRC, but stale/uninitialized payload storage.

| `motor_kind` | `direction` | Configuration byte modified |
|---|---|---|
| `0x00` | `0x00` | Feeder direction-zero speed |
| `0x00` | `0x01` | Feeder direction-one speed |
| `0x01` | `0x00` | Hub direction-zero speed |
| `0x01` | `0x01` | Hub direction-one speed |

Any other selector returns status `0x01`. Valid selectors can mutate configuration using the stale byte, then return status `0x00` and a zero data byte. Sending the “obvious” three-byte packet instead fails the exact-length gate. Do not offer it as a working speed-control API or attempt to exploit previous traffic to seed the stale byte.

### `0x0D` and `0x0F` are distinct gates

`0x0D` action `0x00` disables the extended background RFID/margin path; action `0x01` enables it. Native CLOSE sends zero and OPEN sends one. Disabling does not cancel every already queued insertion operation or guarantee immediate motor stop: a running mechanical helper can finish before the scheduler skips the extended path.

Forced `0x0D` begins with all-slot readiness/retract handling regardless of its mask, so even mask zero can move connected filament. After that it processes selected present slots: action `0x02` temporarily enables the extended RFID/margin phase; action `0x03` keeps it disabled during mechanical pretension while the temporary flag stays zero. Per-slot waits can extend to minutes, and success does not prove fresh acquisition. Outside IDLE/PRELOAD returns `0x03`; all-slot absence is `0x16` on 1.1.3 versus `0x18` on 1.5.2. Passive cache queries remain `0x02`/`0x03`.

`0x0F` instead stores the **automatic-preload inhibition** byte: zero enables, one inhibits. It is distinct both from `0x0D`'s extended-workflow flag and from the print-check fault-report suppression byte. These gates are not emergency-stop controls.

### `0x10`: action-by-action request/reply shapes

Every action uses `[nonzero_onehot_slot, action, argument_byte]`; include the argument even where unused. All replies use code `0x10`. The normal dispatcher and the action-`0x04` private reader have different semantics for actions `0x05`/`0x06`.

| Action | Ordinary dispatcher behavior | Reply |
|---|---|---|
| `0x00` | Readiness/preparation and hub hold/stop sequence; applies finish result | Result status plus **one post-finish operation-state byte** |
| `0x01` | Enters RELOAD; hub feed/settle | Result status, empty |
| `0x02` | Enters RELOAD; initial bounded motion then feed/settle | Result status, empty |
| `0x03` | Clears operation state/indicator via successful finish | Status `0x00`, empty; **no hub-stop or feeder-release call in this action** |
| `0x04` | Enters RELOAD, sends initial ACK, then runs long private-reader feed worker; latches worker return and applies finish | **Immediate** status `0x00`, empty is acceptance only. Early worker failure can have no unsolicited final frame. Private action `0x06` produces its own final frame. |
| `0x05` | Samples live encoder; reports latched action-`0x04` return; clears a nonzero latch after send attempt | Latched-result status plus **four-byte big-endian float32 position**, even with nonzero status |
| `0x06` | Enters RELOAD; fill-to-full buffer test; applies result | Result status, empty |
| `0x07` | Enters RELOAD; relative move based on unsigned argument byte | Actual result status, empty, but operation state clears through finish-success even for result `0x0E` |
| `0x08..0xFF` | Logs illegal data after selector bookkeeping | No guaranteed response |

During action `0x04`, a private fixed-size reader recognizes only the same local address, code `0x10`, valid fixed-span CRC and action byte `0x05` or `0x06`. It does not validate the selector or `L` as the normal parser does. Action `0x05` then yields status `0x00` and the current loop sample while feeding continues; it does not read or clear the latched return. Action `0x06` advances into the buffer-full check, then stops and sends result `0x00` or `0x0D`, empty.

The private action-`0x06` buffer-timeout frame is not retained in the normal action-`0x05` latch: its worker returns zero afterward. Conversely early action-`0x04` errors `0x0A`/`0x0B`/`0x0C` are latched without a final unsolicited frame. Later normal action `0x05` reports the live position at query time, not a stored failure endpoint. A status-`0x00` telemetry frame during the worker does not prove that run will succeed. An apparent private status-`0x02` send branch is unreachable through the fixed reader in the mapped 1.5.2 code; ordinary parser CRC status `0x02` remains real.

An action-`0x06` reply does not identify which reader handled it. If action `0x04` failed just before handoff, normal dispatch can accept `0x06`, start a new buffer-fill operation and return the same empty `0x00`/`0x0D` shape without clearing the earlier result latch. Before engagement or PRINT, the operation coordinator must serialize a normal action-`0x05` result check and exclusively own that consumptive latch. Nonzero or unreconciled results block advancement. Retain any direct `0x0D`; a later zero cannot replace a lost completion.

Detailed stage scheduling, timing and completion ownership belong in [modes and workflows](modes-and-workflows.md); do not build a generic single-empty-ACK wrapper around this code.

### `0x11`, `0x12`, `0x13`: completion limits

`0x11` accepts runtime IDLE or PRELOAD. A selected slot without material returns `0x16`; all-slot no-material can return `0x18` while latching `0x16` into the runtime error structure. All-slot processing scans CONNECTIONS inputs; a no-active-slot case can return success even for an unsupported action. Action `0x00` waits for raw buffer free code `0x02`, not full `0x01`. Actions may detooth and move hub/feeders; zero selection is not a universal immediate stop.

`0x12` mode `0x01` uses configured speed, **not** its supplied speed byte. It clamps the unsigned requested distance above 3000 firmware-mm (`0x0BB8`) to that threshold. In the mapped 1.5.2 loop it stops on strict target exceedance, sustained slow samples or the iteration limit, and **all those exits return status `0x00`**. A success reply therefore does not certify that the requested distance was reached. The nominal full loop is 25 seconds, with scheduling/actuator overhead outside that nominal delay sum. Mode `0x00` and `0x13` preserve readiness/preparation failures rather than unconditionally acknowledging them.

## Native result codes versus original host labels

Status bytes are a separate namespace from function codes, actions, operation states, slot events and hardware fault categories. For example status `0x50` is a filament-consistency event, while **function code** `0x50` writes persistent factory mode.

The native conditions below come from the mapped 1.5.2 movement/buffer producers. Some are specific to one stage; do not use the host's symbolic name as a universal mechanical diagnosis. The 1.1.3 contracts independently map their stage producers; the common protocol does not imply every threshold/timing is identical.

| Status | Native meaning / exposure | Original host label |
|---|---|---|
| `0x00` | Success, no-op, initial acceptance or telemetry success depending on operation; not universal physical completion | `OK` |
| `0x01` | Ordinary invalid length/selector/action or unsupported code; `0x11` action validity can be deferred until a slot is processed | `PARAMS_ERR` |
| `0x02` | Ordinary receive CRC failure | `CRC_ERR` |
| `0x03` | Operation-state/pretension guard; or unchanged parameter for `0x50` | `STATE_ERR` |
| `0x04` | Host-recognized label; **ordinary exact-length validator sends `0x01`, not this value** | `LENGTH_ERR` |
| `0x05` | Connection-low qualification failed in material-to-hub feed | `EXTRUDE_ERR1` |
| `0x06` | Buffer-full timeout in a recovered unreferenced helper; no established normal wire caller for that helper | Unmapped |
| `0x08` | Selected material-low readiness samples failed | `EXTRUDE_ERR4` |
| `0x09` | Required initial hub-outside progress not seen | `EXTRUDE_ERR5` |
| `0x0A` | Initial progress timeout, or action `0x04` never received its private continuation before the loop limit | `EXTRUDE_ERR6` |
| `0x0B` | Direction-zero travel exceeded the firmware's 3000-mm bound | `EXTRUDE_ERR7` |
| `0x0C` | Action `0x04` consecutive slow-sample failure before continuation | `EXTRUDE_ERR8` |
| `0x0D` | Buffer-full qualification timeout; private completion versus normal latch differs as above | `EXTRUDE_ERR10` |
| `0x0E` | Action `0x07` distance criterion not met, even though state is subsequently cleared | `EXTRUDE_ERR9` |
| `0x13` | `0x11` action `0x00` buffer-free qualification timeout | `RETRUDE_ERR1` |
| `0x14` | Connection-high qualification failed during retract | `RETRUDE_ERR2` |
| `0x15` | Host recognizes this status; no corresponding producer in the mapped ordinary movement-stage contract | `RETRUDE_ERR3` |
| `0x16` | No material for selected retract slot; also all-slot error latch even when wire result is `0x18` | `RETRUDE_ERR4` |
| `0x17` | Material raw input high when per-slot retract starts | `RETRUDE_ERR5` |
| `0x18` | All-slot retract readiness found no material; distinguish wire result from `0x16` latch | Unmapped |
| `0x19` | Slow retract sample with a nonzero latched motor-fault mask | `RETRUDE_ERR6` |
| `0x1A` | Retract odometer loop exhausted its sample budget | `RETRUDE_ERR7` |
| `0x22` | Host recognizes motor-load failure; do not infer a specific main-stage producer or direct E-motor diagnosis from this label | `MOTOR_LOAD_ERR` |
| `0x30` | Four-slot update/event payload, not generic failure | `UPDATE_STATE` |
| `0x50` | PRINT sensor-consistency poll error; sets low-progress `0x52` suppression and indicator, but does not itself stop/mask hardware or leave PRINT | `FILAMENT_ERR` |
| `0x51` | Print-buffer check can set it after free-cycle/travel criteria; not a new speed setter command | `SPEED_ERR` |
| `0x52` | Print-buffer check insufficient encoder progress, subject to a separate suppression flag | `ENWIND_ERR` |
| Other | Preserve raw status and command-specific payload; no invented aliases or blanket success | Unmapped |

Native print-check `0x51`/`0x52` can stop/mask buffer hardware independently of a foreground movement return. The motor snapshot uses EEPROM; it is not an unsolicited UART “event `0x80`” packet.

### Why original host return values cannot be used as wire truth

The original host's status handling includes policy and recovery side effects, not just decoding:

- It retries its `CRC_ERR`/`LENGTH_ERR` labels through a separate application loop in addition to transport retries. A new integration must not automatically repeat moving, reset, provisioning or flash operations on those labels.
- For requested `0x10`, status `0x03` can be downgraded to a warning and host Boolean success when its reporting flag is true. That does **not** mean firmware accepted the stage. Unknown statuses normally reach host false, not a fabricated ACK.
- Status `0x30` is consumed as a four-slot update and returns host true on normal completion.
- Status `0x16`/`0x17` can return host true when its remembered last command is absent. Status `0x50` can also return true under action/last-command/automatic-refill gates, while latching or scheduling recovery work.
- Error handling can mutate live/saved material state, register deferred work, pause or invoke printer macros. It is not a command-level wire error-to-exception mapping.
- Code `0x55` has an early host shortcut returning the first data byte without the usual status path. That shortcut is not a reason for a replacement to ignore malformed/nonzero-status test replies.

Preserve actual status, payload, source and command for diagnosis. Separate native result interpretation from printer retry, pause, refill and user-reporting policies. [Integration](integration.md) describes those ownership boundaries.

## Version profile and unsupported behavior

| Feature | 1.1.3 | 1.5.2 |
|---|---|---|
| Recognized ordinary codes | 27 | 31 |
| Normal/error-layout `0x0A` data | Four bytes | Six bytes |
| `0x0A`, status `0x30` | Four slot-event bytes | Four slot-event bytes |
| `0x03` material present, no valid card | Can return `0x64`, ambiguous with genuine 100% | `0xFF` |
| `0x12` distance/preparation command | Unsupported, ordinary status `0x01` | Implemented, ignored speed byte |
| `0x5B`, `0x5C`, `0x5D` | Unsupported, ordinary status `0x01` | Implemented |
| `0x54` | Unsupported | Unsupported |
| `0x06` selector path | Inherited RX tick and CURRENT-mirror alias defects | Same two independent defects |
| `0x0B` indicator selection | Validates request slot but selected patterns use CURRENT | Same alias defect |
| `0x09` length mismatch | Accepts two, consumes stale third data position | Same defect |
| `0x0C` / `0x50` storage | One-byte mode, internal-flash parameter ID `0x06` | Same storage role |

A profile should reject version-inapplicable operations before sending. Do not probe support with a state-changing command. The normal/error/event distinction, native producer layouts and actual version identity take precedence over old host labels or an accidental parse of one short frame. [Compatibility](compatibility.md) covers application-package equivalence and the unproven running host/firmware pairing.


# Maintenance, updates, and factory diagnostics

These operations are optional maintenance interfaces, **not normal CFS initialization**. Keep them separate from connection, material selection, and print workflows. Reset, persistent mode changes, factory tests, credential writes, and flash updates require explicit operator intent. No device commands, physical tests, or flashing were performed to produce this reference.

See [transport](transport.md) for framing, [addressing](addressing.md) for identity and address assignment, [storage](storage.md) for persistent regions, and [compatibility](compatibility.md) for application differences. All wire values, byte offsets, and lengths below use hexadecimal; timing and descriptive counts may use decimal.

## Separate three interfaces

| Interface | Target and purpose | Important boundary |
|---|---|---|
| Application maintenance | Running CFS application, normally its assigned address | Application opcodes such as `0x50` and `0x56` are not bootloader transfer commands |
| Bootloader update | Verified public `cfs0_050_G32` loader; local opcode `0xF0` | Payload meaning depends on the current transfer phase |
| Bootloader factory EEPROM | Destination `0xF0`, **opcode `0xF2`** | Factory destination is shared; this is not a local opcode-`0xF0` transaction |

The public loader came from another unit whose application was 1.0.8. The packaged G30/G32 application aliases being identical does **not** establish identical bootloaders, flash geometry, hardware, or recovery behavior. The internal G30 loader, the external reader's loader, and the onboard reader/secure-applet implementation are outside this document's receiver-side proof. Their absence does not block normal application communication.

## Application maintenance

Requests use the ordinary application framing and request mode described in [transport](transport.md). `N` below is payload length, excluding frame fields and CRC. Unless stated otherwise, a successful framed reply has status `0x00`.

| Opcode | Request payload / `N` | Response and effect |
|---|---|---|
| `0x0C` | Empty / `0x00` | One byte: RAM copy of operating/factory mode, loaded from internal-flash parameter ID `0x06` at startup; not flash readback |
| `0x50` | `[mode:u8]` / `0x01` | Changed: set RAM mode, attempt flash commit, send empty status-`0x00` reply, then delay 100 firmware ticks and request MCU reset. Unchanged: status `0x03`, empty payload, no reset |
| `0x51` | `[fixture_status:u8]` / `0x01`, destination **`0xEB` only** | Updates the fixture-output/status flag; no ordinary success ACK. The same valid request at a local address does not perform this update |
| `0x52` | Empty / `0x00` | Environment query; successful reply uses **opcode `0x53`**, with `[temperature:i8, humidity:u8]`, length `0x02` |
| `0x53` | `[reference_temperature:i8, reference_humidity:u8]` / `0x02` | Sets fixture comparison references, not live sensor cache; no ordinary success ACK |
| `0x56` | Empty / `0x00` | Empty success ACK, then the same delayed MCU reset path |
| `0x5B` | Empty / `0x00` | UID-derived identifier of length `0x08`; 1.5.2 only |
| `0x5C` | `[credential:0x10 bytes]` / `0x10` | Persistent EEPROM write and a length-`0x10` returned credential buffer; 1.5.2 only |
| `0x5D` | Empty / `0x00` | Start fixture self-test; no ordinary framed completion ACK; 1.5.2 only |

The application length guards are exact. For example, `0x5C` requires `L=0x13`, since `L=N+0x03`. Invalid `0x53` length can report an error under opcode `0x52`; this does not turn a valid `0x53` write into an acknowledged transaction. A sender must not wait for an ACK that the valid write does not produce.

### Persistent mode is not app/boot identity

Parameter ID `0x06` is **one byte in internal flash**, not a six-byte EEPROM record. Its meaning is separate from the addressing identity's app/boot mode and from transient operation state:

- `0x00`: normal startup.
- Any nonzero byte: enter TEST before validating the specific factory selection.
- `0x01`: four-channel cycle/endurance workflow.
- `0x02`: another aging workflow.
- Other recognized selections invoke fixture/aging routines; an unknown nonzero value can still leave TEST selected.

A persistent nonzero value can therefore cause automatic motion after reboot. Do not probe supported values by writing them. Changing to `0x00` through `0x50` attempts a persistent write plus reset; it is not an instantaneous motor-stop operation. The RAM-only console test-mode switch is a different mechanism. See [modes and workflows](modes-and-workflows.md) for version-specific factory selector boundaries.

For `0x50`, destination `0xEA` suppresses the ordinary reply; `0xEB` temporarily makes the response address `0xEB`. Prefer a verified local address for intentional maintenance, rather than treating these group-address behaviors as request correlation. Status `0x00` does **not** verify the flash write: commit errors are not propagated, and a pre-reset `0x0C` reads the changed RAM value, not flash. The parameter store has no verified journal, rollback, or checked write-readback guarantee. After reset, rediscover the same UUID and verify the loaded mode and runtime state before permitting motion; do not infer persistence from the ACK.

### Reset is not an emergency stop or proven loader hold

`0x56` requests a Cortex-M system reset after acknowledging; it does not directly jump to a recovered loader, set a proven persistent loader-hold mailbox, or establish that mechanics have stopped. Reset also loses RAM address assignment and requires the host to re-establish application identity and state. Do not use loss of heartbeat or loss of a reset ACK as physical-stop evidence.

The vendor startup script sends `0xEB:0x56` before invoking its update utility. That is a description of vendor orchestration, not proof that every loader will stay available after application reset.

### `0x5B` is not UUID12

The length-`0x08` value is two little-endian words:

1. MCU UID word at `0x1FFFF7E8` unchanged.
2. UID word at `0x1FFFF7EC` bitwise-AND UID word at `0x1FFFF7F0`.

Address discovery and assignment instead use the full ordered UUID of length `0x0C`. Do not substitute `0x5B`'s reduced value for UUID matching, inventory deduplication, or loader selection.

### `0x5C` writes credentials

This operation writes EEPROM offsets `0x060..0x06F` and invalidates the credential-read cache. It is neither a spool-tag read nor a spool-tag write. An incompatible value can break authentication with the onboard secure device.

The original handler does not surface low-level EEPROM write failures reliably. Its read helper can return existing cache contents after a failed read; an ordinary status-`0x00` reply alone is therefore insufficient. A host may compare all returned bytes with the intended value, but even a match is not an independently proven power-loss persistence guarantee. Keep actual credential values out of logs, examples, and general-purpose integration configuration. No credential data or factory guard bytes are supplied here.

### Environment fixture exchange

Persistent mode `0x20` invokes this fixture comparison in both application 1.1.3 and 1.5.2. An environment startup fault skips the comparison. Otherwise the firmware requests a 2.5-second delay, then sends unsolicited **opcode `0x52`, status `0xFF`, empty payload** prompts at requested intervals of about 100 ms while either reference byte is zero. There is no deadline. These prompts are not the opcode-`0x53`/status-`0x00` response to a normal `0x52` cache query.

An authorized fixture host supplies `[reference_temperature:i8, reference_humidity:u8]` with `0x53`; this writes comparison references and produces no ordinary success ACK. **Both bytes must be nonzero**: a well-formed reference of `0` °C or `0`% RH never satisfies the wait gate. Supplying references does not refresh the normal environment cache.

Once the gate is satisfied, the firmware makes up to three comparison attempts using truncated temperature and humidity samples, accepting differences within ±2 °C and ±5 humidity percentage points, respectively. It stops attempting on a match. The sensor-read return is not checked, so these are not three guaranteed fresh measurements. Completion/failure is presented through local display/log output, not a framed completion ACK; do not treat silence or a reference write as a pass.

## Factory self-test and raw JSON

`0x5D` enters TEST and exercises the hub and four feeder drives, current/FG feedback, fixture GPIO pairs, boot-fault indications for environment/encoder, EEPROM status, and both onboard RFID readers. Each RFID probe uses lane `0x00`; this is **not** a fresh scan of all four filament slots. Tests can move motors, wait for external inputs, and leave GPIO/mode configuration changed. They are unsuitable as startup health probes on an assembled working printer.

### Report schema

Application 1.5.2 emits one unframed JSON object containing a `uuid` string and exactly these 24 numeric result fields. Indices are hexadecimal offsets within the result array, not transmitted JSON keys.

| Index | Exact JSON key | Index | Exact JSON key |
|---|---|---|---|
| `0x00` | `EEPROM` | `0x0C` | `BLUE3LIMIT7` |
| `0x01` | `BLDC` | `0x0D` | `BLUE4LIMIT8` |
| `0x02` | `BDC0` | `0x0E` | `BUF_LIMIT1/BUF_LIMIT2` |
| `0x03` | `BDC1` | `0x0F` | `CS/RD` |
| `0x04` | `BDC2` | `0x10` | `WR/DAT` |
| `0x05` | `BDC3` | `0x11` | `ADDR_A/BG_CTL` |
| `0x06` | `RED1LIMIT1` | `0x12` | `ADDR_B/IO_POWER` |
| `0x07` | `RED2LIMIT2` | `0x13` | `HUM/TEM` |
| `0x08` | `RED3LIMIT3` | `0x14` | `AHT30` |
| `0x09` | `RED4LIMIT4` | `0x15` | `ODOMETER` |
| `0x0A` | `BLUE1LIMIT5` | `0x16` | `RFID0` |
| `0x0B` | `BLUE2LIMIT6` | `0x17` | `RFID1` |

`uuid` encodes `0x0C` ordered bytes as `0x18` hexadecimal characters without spaces. Result values are unsigned bytes, serialized as ordinary JSON numbers. Preserve their raw values rather than normalizing every nonzero number to a Boolean.

The result convention is **`0x01` pass, `0x00` fail**, unlike the hardware-fault command's error convention. All result bytes start at `0x01`; an untested field can look successful. A confirmed firmware indexing bug makes the PA7/PA4 `HUM/TEM` test overwrite result `0x10` (`WR/DAT`) instead of `0x13` (`HUM/TEM`). Consequently:

- `HUM/TEM` can remain at its initialized pass value without holding that test's result.
- `WR/DAT` can lose its own earlier result.
- An all-ones report is not a reliable proof that all tests ran and passed.

The firmware prints the report three times with 500 ms between reports. These are repeated reports, not three independent test executions. Application 1.1.3 has human-readable fixture/aging paths but **does not implement `0x5D`, `0x5B`, `0x5C`, or this structured JSON report**. Do not wait for this schema on 1.1.3.

### Coexistence with binary traffic

The report is raw text on the same RS-485 UART as framed `0xF7` traffic, through the direct formatted-stdio callback. It has no `0xF7` header, frame CRC, response status byte, or binary completion ACK. The board's separate RT-Thread console is configured for `jlinkRtt`; that does not negate the independently present direct-stdio UART route.

A host that supports fixture reports needs persistent incremental parsing for both binary frames and JSON, including partial objects spanning reads and unsolicited frames arriving between reads. Recognize a report by the complete schema, not merely an opening brace. Preserve unrelated binary traffic and do not discard a partial JSON object while waiting for a binary response.

However, there is no recovered per-report UART lock. Text and framed sends can interleave **inside** each other's byte sequences; no receive parser can reconstruct bytes that the transmitter interleaves or loses. Avoid concurrent ordinary traffic during an authorized fixture run, and report malformed/incomplete output honestly rather than manufacturing a pass or completion ACK.

## Verified public `050_G32` loader

This section documents only the known public `cfs0_050_G32` loader, not an installed G30 or reader loader.

### Startup, identity, and framing

The recovered loader initializes UART1 at 230400 baud, 8N1, and waits up to 30 seconds for address assignment. A valid application can start when that window expires. Assignment instead enters the command loop; no persistent hold mailbox was recovered in this startup path.

It accepts its local address, group `0xFE`, broadcast `0xFF`, and factory destination `0xF0`. Its internal address-class enumeration is not the wire mode byte. The wire mode byte is stored and CRC-covered but is not the command-dispatch gate. The vendor updater sends mode `0x00`.

The frame is `0xF7 | address | L | mode/status | opcode | payload | CRC8`, total length `L+0x03`, with `L=N+0x03`. This loader accepts `L=0x03..0xFC`, so maximum payload is `0xF9` bytes. CRC-8 is MSB-first polynomial `0x07`, initial `0x00`, no reflection or final XOR, over length, mode/status, opcode, and payload; sync, address, and CRC are excluded.

Loader addressing differs from the application:

- Group `0xFE`, opcode `0xA0`: match UUID12, accept a nonzero new address, reply once from it.
- Group `0xFE`, opcode `0xA1`: inclusive address range, including already assigned boxes in range, with a nonzero pseudorandom delay of 1–999 ms.
- Local `0xA2`/`0xA3`: identity `[type=0x01, mode=0x01, UUID:0x0C bytes]`.

A host policy for this documented loader should select one known UUID, verify an unambiguous local address, and verify exact loader identity and geometry **before any write**. These gates are recommendations for a new implementation, not protections enforced by the vendor utility.

### Command phase: local opcode `0xF0`

The following one-byte payloads are subcommands **only in command phase**. Replies use opcode `0xF0`, outer status `0x00`, and the payload shown; `0x75` and `0x20` are payload markers, not frame-status values.

| Subcommand | Reply payload | Effect |
|---|---|---|
| `0x00` | `0x19` bytes: boot ID `0x0C`, ASCII `-` (`0x2D`), app ID `0x0C` | Recheck application CRC; invalid app suffix is `0x0C` zero bytes |
| `0x01` | `0x75` | Enter length phase |
| `0x02` | `0x75` if valid; otherwise `0x1F` | Recheck CRC; on success wait three seconds, then hand off through app vectors |
| `0x03` | `0xE0` | Geometry encoding for this loader's `0x80`-byte wire chunks |
| `0x06` | `0x75` success / `0x1F` flash error | Erase private internal-flash page at `0x0807F800` |
| `0x07` | Raw UUID, length `0x0C` | Read full hardware UUID |
| Other | No action/reply | No `0x04`/`0x05` protocol is established |

Require the boot identifier `cfs0_050_G32` and geometry `0xE0` for this contract. A matching filename is not an identity query. Do not infer arbitrary geometry units from `0xE0`, or generalize its decoding to an unknown loader.

Private-page erase is a separate destructive operation. It is **not application erase**, not required as an implicit preliminary step to upload, and not an EEPROM credential erase. Application-area pages are erased by the transfer writer as it programs them.

### Length and image phases

The payload widths below are **mandatory sender invariants**, not guarantees that the native receiver rejects other lengths. The host MUST enforce them before sending.

| Current phase | Mandatory host opcode `0xF0` payload | Required normal reply | Next phase |
|---|---|---|---|
| Command | Exactly `[0x01]` | Exactly `[0x75]` | Length |
| Length | Exactly `0x04` little-endian image-length bytes, nonzero; **no subcommand prefix** | Exactly `[0x75]` | Image |
| Image, nonfinal block | Exactly the next `0x80` raw image bytes | Exactly `[0x75]` | Image |
| Image, final block | Exactly the remaining `0x01..0x80` raw bytes | Exactly `[0x20]` (DONE) | Command |

The receive helper accepts a nonempty local opcode-`0xF0` payload, copies a caller-selected width (`0x04` in length phase, `0x80` in image phase), and returns the actual received payload length; it does not require that length to match the copy width. Length phase checks receive failure and whether the copied word is nonzero, not whether exactly four bytes arrived. Image phase passes the staged block to the writer and subtracts the actual received length from the remaining count without first enforcing the table's chunk widths. Malformed widths can therefore desynchronize phase arithmetic or programmed data rather than fail safely. Do not send malformed packets to probe this behavior.

The loader allows five failed length attempts before returning to commands. Transfer errors can return `0x1F` (NACK) or `0x21` (FAIL); neither is a safe instruction to blindly replay data.

There is no offset, sequence number, per-block subcommand, wire padding, digest trailer, or separate EOF packet. Send the file starting at offset `0x00`, including its vectors and metadata; there is no wrapper/header skip. For the archived 1.5.2 image of length `0x0002AE38`, the length payload is `38 AE 02 00`, followed by 1373 chunks: 1372 full `0x80`-byte chunks and a final `0x38`-byte chunk. Final `0x75` alone is **not** completion; require `0x20` in response to that final chunk.

**Do not automatically retry ambiguous transfers.** If a request was accepted but its reply was lost, the next packet may be consumed in a different phase or appended as new image bytes. Even an apparently harmless version, geometry, or start subcommand becomes image data while the receiver is in image phase. On timeout, unexpected marker, malformed reply, interrupted write, or uncertain local echo, treat receiver phase as unknown and stop sending; this reference establishes no universally safe in-band resynchronization sequence.

Local adapter echo must be an explicit transport property, not a frame-equality heuristic: a final one-byte data payload `0x20` can be identical to a valid DONE reply. Bind replies to the selected local address, opcode, outer status, expected payload length, and current phase, but recognize that the protocol still has no transaction identifier.

### Image metadata and CRC16

Application file offset `0x00` maps to physical flash `0x08003000`. Multi-byte image-header fields are little-endian.

| File offsets, inclusive | Length | Field |
|---|---|---|
| `0x000..0x003` | `0x04` | Initial stack pointer |
| `0x004..0x007` | `0x04` | Thumb reset vector |
| `0x200..0x20B` | `0x0C` | Application descriptor |
| `0x20C..0x20D` | `0x02` | Stored CRC16 |
| `0x20E..0x211` | `0x04` | Declared image byte length |

CRC16 is MSB-first, polynomial `0x8005`, initial `0x0000`, no reflection, no final XOR. Compute over the exact declared image span beginning at offset `0x00`, substituting zero for **all six bytes `0x20C..0x211`**. Do not omit that range or hash only the application body. The ASCII known-answer input `123456789` gives `0xFEE8`.

Offline validation MUST reject files shorter than the complete `0x212`-byte header and require exact file/declaration length agreement, matching CRC16, and the image-contained `0x0C`-byte descriptor at `0x200..0x20B` matching ASCII `cfs0_DDD_DDD`, where each `D` is an ASCII digit `0`–`9`. A matching filename is not a substitute. The recovered client policy also requires `0x20000000 < SP <= 0x20010000`, stack alignment `0x08`, and a Thumb reset vector (bit zero set) whose target, after clearing that bit, is inside `[0x08003000, 0x08003000 + image_length)`. Require the exclusive padded end `0x08003000 + programmed_extent <= 0x0807F800`, using the extent below. These are host checks; they are not proof that the loader validates the same boundaries before writing.

| Archived image | Image length | Stored/computed CRC16 |
|---|---:|---:|
| CFS 1.5.2 | `0x2AE38` | `0x7377` |
| Public CFS 1.0.8 | `0x24B28` | `0x172F` |
| External RFID 0.1.0 | `0x5230` | `0x3BC2` |

The reader's matching packaging calculation does not establish that its separate loader uses the same transfer or recovery policy.

### Programming extent is larger than wire data

The receiver stages two `0x80`-byte blocks, programs `0x100`-byte groups, and erases `0x800`-byte pages at aligned write starts. Its final zero padding is unusual. With image length `n`:

- Extra `0x80`-byte blocks: `(-(bool(n & 0x7F) + ((n >> 7) & 0x01))) & 0x0F`.
- Programmed extent: `(ceil(n / 0x80) + extra_blocks) × 0x80` bytes.

This is **not ordinary rounding to a `0x800`-byte boundary**; it can add `0x0E` or `0x0F` blocks. The padding is generated by the loader, not sent by the host and not included in the declared image CRC span. The host MUST require the exclusive padded end `0x08003000 + programmed_extent <= 0x0807F800`, so no programmed byte enters the private page; checking only the nominal file end is insufficient. Do not send extra zeros to imitate the writer's internal padding.

DONE does not recompute the application CRC. Version query and start do. Flash programming has internal retry handling, but no independent host readback operation or rollback slot is established. CRC16 is corruption detection, **not cryptographic authenticity**; no image-signature enforcement is recovered here. After a successful start, a host can separately reconnect and query the running application, but must not confuse that with byte-for-byte flash verification.

### Broadcast start is distinct

`0xFF:0x0B [0x01]`, mode `0x00`, requests immediate CRC-gated application start without a reply. Full frame: `F7 FF 04 00 0B 01 C8`. It has no local-`0xF0 [0x02]` three-second delay and can affect every listening loader.

The vendor utility's broadcast-only invocation instead sends `F7 FF 04 00 0B 00 CF`. On this verified loader, `[0x00]` is **not** the start command. On the separate reader application, `0x0B [0x00]` requests reset; see [external reader](external-reader.md). Never interpret one broadcast payload's meaning uniformly across app and boot targets.

## What the vendor updater does—and why not to copy it blindly

The statically inspected `mcu_util_485` performs discovery, assignment, version and geometry queries, private erase, image selection, transfer, application start, and report generation. It is evidence of vendor host behavior, not a safe reference implementation.

| Vendor behavior | Consequence for a new host |
|---|---|
| Discover groups `0xFD` motors, `0xFC` belts, `0xFB` RFID, `0xFE` CFS; assign CFS from `0x01`, RFID from `0x11` | Do not conflate external-reader runtime fixed address with a proven reader bootloader implementation |
| Private erase `0xF0 [0x06]` for every type-`0x01` record at `0x01..0xF0`, before image selection | It can erase settings on devices later skipped for upload; private erase must be independently authorized |
| Upload candidates require stored boot mode `0x01`, address `0x01..0xEF` | Later operations are less selective; do not infer a global mode/UUID safety gate |
| Filename-prefix and manifest-based selection; force bypasses comparison, not candidate lookup; UUID/file override uses a different manifest schema | Filename/version policy is not hardware, signature, image CRC, or geometry validation; non-CFS comparison can also accept downgrades |
| One shared semaphore for recognized callbacks, without request-token/address binding or per-request drain | A wakeup is not proof of a matching positive reply |
| Up to three exchanges with 500 ms waits; retries can resend data without a sequence number | These retries are unsafe to adopt as a generic CRC/timeout recovery policy |
| Worker can proceed after handshake retry exhaustion | Require actual phase acceptance rather than imitating fall-through |
| Signed geometry arithmetic and narrowed read sizes | For this loader `0xE0` yields `0x80`; positive or zero encodings have defective zero-read/skip paths and are not a general geometry contract |
| Start `0xF0 [0x02]` is attempted for nonfailed records, including skipped images | An update report can describe start success without a new image upload |
| Reported successful version can be copied from selected filename | It is not a post-start application-version query |

The utility has neither flash readback nor independent image validation, signature verification, rollback, alternate-image recovery, or post-start version verification. Its outer process status is not a trustworthy aggregate per-device success result. These facts explain vendor behavior; an independent integrator should not reproduce defects as successful no-ops.

## Factory EEPROM: destination `0xF0`, opcode `0xF2`

This is a privileged, shared-destination loader path, separate from local opcode `0xF0`. Every attached compatible loader hearing destination `0xF0` can receive it. Use only an isolated single-unit bus under explicit maintenance authorization; ordinary UUID/address matching does not isolate this destination. Both read and write replies use **source address `0xF0` and opcode `0xF2`**, not the loader's assigned local address.

| Operation | Payload | Reply |
|---|---|---|
| Read | `[0x01, count:u8]`, count `0x01..0xF9` | Status `0x00` plus exactly `count` EEPROM bytes from offset `0x000`; comparison failure returns status `0x01` with exactly `count` **untrusted bytes**, not an empty payload. Never consume them as a successful read |
| Write | `[0x00, guard:0x08 bytes, data:0x01..0xF0 bytes]` | Writes from offset `0x000`, verifies readback; status `0x00` on success or `0x01` on failure, with an empty payload for either status |

The safe write maximum follows the frame payload limit: `0x01 + 0x08 + 0xF0 = 0xF9`. There is no arbitrary EEPROM offset field in these requests. The read path compares two reads, with up to three pairs, but underlying EEPROM bus waits can remain indefinite.

The write guard is a fixed, loader-specific code-derived value of length `0x08`. It is **not** the EEPROM secure credential and is not a device-specific authentication secret. No actual guard or credential bytes are included in this repository. Do not infer a compatible guard from a version filename or an unverified loader image.

Writes can overwrite device identity and credentials. Private internal-flash erase, application transfer, application `0x5C` credential write, and factory `0xF2` EEPROM write are four different operations. None implies permission to perform the others, and none should be hidden inside normal integration startup.

This reference does not establish physical flash reliability, power-loss behavior, motion safety, electrical interoperability, or byte identity to an installed loader. No device access, physical validation, or flashing was performed.

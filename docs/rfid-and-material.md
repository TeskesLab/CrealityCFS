# RFID, material identity and remaining amount

RFID acquisition is a motor-moving workflow, not simply a memory read. [Modes and workflows](modes-and-workflows.md) describes scheduling and motion; [commands](commands.md) describes the cached host interface. Numeric protocol values, offsets and byte counts below are hexadecimal. Percentages, physical units and timing are decimal.

## Cached host interface

Opcode `0x02` returns selected slot entries labeled `A:` through `D:` and terminated by `;`. A data entry contains exactly `0x28` opaque bytes. `none` means absent, `unknown` means filament present without a publishable cache, and `busy` means the selected slot is busy. Presence is not successful identification.

Prefer **one-slot queries** when exact attribution matters: the complete payload length distinguishes a data record from the tokens. Multi-slot replies have no escaping or per-entry lengths; opaque data can contain token-like prefixes, labels and semicolons. Require a unique complete parse against the requested slot order. If multiple full parses fit, preserve/report the ambiguity or reject the reply; do not greedily prefer either a sentinel or a `0x28`-byte record. Preserve opaque bytes before attempting material decoding.

Native publishability requires presence, a first character in ASCII `1`–`9`, `A`–`C` or `a`–`c`, and no applicable inhibit; it does not validate the full record. In 1.5.2, debounced presence plus a valid saved usage record with UID length `0x07` and zero saved life/count inhibits publication. This is the [status depletion/cache-inhibit mask](sensors.md#environment-cache-and-status-0x0a), not pending scan work; a clear bit does not establish a completed read.

Opcode `0x03` returns per-slot cached percentage: in 1.5.2, `0xFF` represents unselected, busy or unavailable; a selected nonbusy absent slot returns `0x00`. The percentage publisher converts zero to one. Application 1.1.3 can report decimal 100 for failed acquisition or present-but-unidentified material, so that value does not prove a full identified spool. Cache replies do not authenticate a tag anew.

Opcode `0x0D` action `0x00` disables the extended automatic RFID workflow, and `0x01` enables it (default). Disable is not synchronous cancellation of an already-running scan. Forced RUN (`0x02`) temporarily enables the extended RFID/margin phase; forced TIGHT (`0x03`) runs mechanical pretension but skips that phase while its temporary flag remains zero. Neither guarantees a fresh tag read. Both can first retract connected slots outside the selected mask, **even with a zero mask**. See [workflows](modes-and-workflows.md) for guards and sequencing. Opcode `0x0F` is separately automatic-preload control: `0x00` enables, `0x01` inhibits; it is not a fault-report flag.

## The `0x28`-byte ASCII material record

Offsets are zero-based; end offsets are exclusive. There is no trailing NUL in the record.

| Start:end | Width | Field | Interpretation |
|---|---|---|---|
| `0x00:0x05` | `0x05` | Date | Month character, two day characters, two year characters; retain text |
| `0x05:0x09` | `0x04` | Vendor | Native schema spells this `venderId` |
| `0x09:0x0B` | `0x02` | Batch | Text |
| `0x0B:0x11` | `0x06` | Full material ID | Includes prefix; database key excludes first character |
| `0x11:0x18` | `0x07` | Color | Preserve all seven characters |
| `0x18:0x1C` | `0x04` | Nominal length | Decimal text fallback, not binary or hexadecimal |
| `0x1C:0x22` | `0x06` | Serial | Native `serialNum` |
| `0x22:0x28` | `0x06` | Reserved | Preserve unchanged |

Example, with separators added only for explanation:

`AB124 | 0276 | A2 | 100001 | 002972D | 0330 | 000001 | 000000`

The older host requires exactly `0x28` characters. Resolve `result.list[].base.id` against record bytes `0x0C:0x11`, i.e. full material ID without its first character. Preserve exact strings and original types. The database supplies `brand`, `name`, misspelled `meterialType`, `density`, `diameter`, `minTemp`, `maxTemp`, and engine values `pressure_advance` and `filament_max_volumetric_speed`. Native projections include `materialType`, `pressure`, `maxVSpeed`. **Temperatures and extrusion tuning are not encoded in this tag record.** An unknown database ID does not establish Generic PLA or any other material.

The seven-character color is not proven to be conventional RGB plus alpha. Native UI code prepends `#` without stripping the leading character. Do not silently normalize it to six characters, change case or infer channel semantics.

The 1.5.2 nominal fallback uses only characters at offsets `0x19`, `0x1A`, `0x1B`, with decimal weights 100, 10, 1, then multiplies metres by 1000 for millimetres. Each nondecimal character contributes zero; offset `0x18` is ignored. Thus text `0330` means 330000 mm, not hexadecimal `0x0330`. UID-length `0x04` accounting uses this fallback; another type keeps supplied capacity if at least 10000 mm, otherwise substitutes the fallback. This parser is not installed as ongoing usage accounting in 1.1.3.

## Reader/slot selection

Slots A/B use reader `0x00`, lanes `0x00`/`0x01`; C/D use reader `0x01`, lanes `0x00`/`0x01`. Paired neighbors are B/A/D/C. Direct reader buses are PB8/PB9 and PB10/PB11. Lane indexes RAM, not an observed antenna-select write. The moving scan uses neighboring filament motion and visibility to isolate the desired tag; a detected card alone does not prove its slot.

Application 1.1.3 always uses direct readers. In 1.5.2, PD10/PD11 straps select direct versus secure-board operation; either high selects the secure variant. Secure reader selectors are `0x28` for A/B and `0x29` for C/D. Failed secure authentication does not switch to a direct/default-key fallback.

## Classic block and key contract

The 1.5.2 direct Classic signature is ATQA `04 00`, final SAK `0x08`. The normal 1.1.3 reader uses the Classic layout without that newer type dispatch. Authenticate the sector containing blocks `0x04`, `0x05`, `0x06` using the UID-derived six-byte key. Its derivation is AES-128 encryption of UID4 repeated four times, taking the first `0x06` output bytes. Independently AES-decrypt each `0x10`-byte data block with a **different** fixed key. Concatenate all of plaintext blocks `0x04` and `0x05`, then the first `0x08` bytes of block `0x06`, yielding `0x28` bytes. There is no IV, chaining, padding or MAC in these block transforms. A sector-trailer formatter exists for `key6 | FF 07 80 69 | key6`; its existence is not proof of an installed host write command.

### Extracting the two fixed keys without publishing them

Use only an exact verified application artifact, not a boot package or a guessed firmware revision. File origin is physical `0x08003000`.

| Application | Complete-image SHA-256 | Physical source | File offset |
|---|---|---|---|
| 1.1.3 | `386a1106391a332e6a97803c5ce00b87b6d3ddc4c3d61f6fe07fb19fd3125b76` | `0x08026149` | `0x00023149` |
| 1.5.2 | `ed969a32b6f55faed0d41f42d94be6d4fc9ca1598208471dc6179dc0a75435af` | `0x0802BD59` | `0x00028D59` |

The source is `0x30` bytes, identical in both verified images, SHA-256 `061e7e6d7abdfecf4db0c36b0ae5fefb8e0c19560d61f922493cdf27edcccc36`. It consists of four interleaved rows of `0x0C` bytes. Concatenate the four-byte little-endian words at source-relative offsets `0x00`, `0x0C`, `0x18`, `0x24` to obtain the UID-authentication AES key; use offsets `0x04`, `0x10`, `0x1C`, `0x28` for the independent block AES key. Preserve each word's little-endian byte order; do not reverse it again. Each resulting key is `0x10` bytes. Their SHA-256 checks are respectively `e544d94feb16159bbd7bc227df1e283eca1f38f2bb2015dfcc6161b74473b5c2` and `acec2106007458579ba522b25610b2cf509ae59d7879cb975f65c45228e5c9a1`. No literal key or factory credential is reproduced here.

## Alternate route and version-specific defects

The 1.5.2 alternate direct path reads block `0x04` and `0x08`, takes the first `0x14` concatenated bytes and formats them as `0x28` uppercase hexadecimal characters. It is **not** Classic AES material decoding. The secure alternate path similarly reads five `0x04`-byte chunks with selectors `0xFF80`–`0xFF84`. Do not interpret the resulting representation automatically as the ASCII field table above.

Two 1.5.2 direct-route defects are closed static findings, not missing-producer assumptions:

* The raw detection UID-length byte has no installed in-bounds producer and remains cold-zero. It is copied into the tag record. UID snapshot returns length zero without filling output, and the two-hit comparison performs a zero-length comparison. Native direct scanning therefore does **not** establish same-UID identity. Supplying length `0x04` would change behavior, not reproduce it.
* The direct auxiliary source has no installed in-bounds producer and stays cold-zero; logging/copying it does not create a capacity or percentage measurement.

In 1.1.3 the two-hit check compares actual UID4 words. Its block-`0x08` path really can publish an auxiliary little-endian binary32 plus a validity byte; failed reads preserve prior auxiliary data. That auxiliary is not geometric percentage or the newer authenticated usage fields. The normal 1.1.3 scan does not automatically fall back to plaintext/default keys; such fallback wrappers belong to diagnostics.

The separate external reader has its own classification, default-key arm and forced material/color characters. Those overrides must not be applied to CFS application caches; see [external reader](external-reader.md).

## How the CFS accesses secure tags

Implementing the MCU-side interface does not require reconstructing chip firmware. The following is the observed 1.5.2 FM15L013-facing contract; it is absent from the installed 1.1.3 callbacks. It is a separate PB10/PB11 GPIO bus, seven-bit address `0x71` (write/read address bytes `0xE2`/`0xE3`), not RS-485.

This is optional implementation background, not a second setup protocol for an external RS-485 master. The CFS MCU performs this work. Normal integrations use its cached `0x02` reply and the documented acquisition controls; they do not need direct access to this internal bus, its credential, or the applet's firmware.

A transport frame is `LE16(payload length + 0x03) | header2 | operation/status | payload | XOR`. XOR covers preceding frame bytes, not the bus address. TX header2 is zero; operation `0x02` carries an APDU, `0x30` initializes. RX accepts at most `0x1D` payload bytes and does not validate header2. Transport success `0x00` and trailing **big-endian** APDU status `0x9000` are distinct. Bad length and XOR return `0x0D` and `0x0E`; address/ACK timeout is `0x0C`. The APDU exchange polls only transport `0x04` and `0x0C`; it does not resend the APDU on each poll. Mapped transport errors include `0xFFE2` timeout, `0xFFE3` length and `0xFFE4` checksum.

Secure body encryption is two-key TDEA ECB, **not the Classic AES keys**. The provisioned `0x10`-byte credential K authenticates the handshake below; encrypted setup and tag APDUs use the **derived `0x10`-byte session key**, not K directly. Append `0x80` and zero bytes through the next `0x08` boundary, including a full block for aligned plaintext. CLA/INS/P1/P2 and padded Lc stay clear. Response data before SW is decrypted separately with the session key. Native unpadding uses the last `0x80` without validating following zeros; malformed lengths/markers should be surfaced explicitly rather than exposing native undefined-memory behavior.

Initialization sends transport `0x30`, SELECT `00 A4 04 00 0C` with ASCII `Temporary.MF`, version query `80 CA 00 F1 0A`, then `00 A4 00 00 00`. All APDU steps require `0x9000`. Authentication is:

1. Form UID8 as `LE32(uid0) | LE32(uid1 AND uid2)` from the MCU identifier words. Supply an independent eight-byte challenge tail N.
2. Send clear `80 71 00 00 10`, body UID8 | N. Require SW `0x9000` and consume response C1 | R, each `0x08` bytes.
3. Require TDEA-decrypt with credential K of C1 equals **N**, not UID8. Mismatch is `0xFFCE` before initializer translation.
4. Let P = TDEA-encrypt K of R. Send `84 73 01 00 08`, body P; require `0x9000`.
5. Session material is `E_K(C1) | E_K(P)`. The second response body is **not** session-key input.

The native initializer translates authentication failures to `0xFB` and a non-`0x10`-byte derived session to `0x03`. It then sends session-encrypted `80 70 00 00` setup bodies for reader selectors `28 00` and `29 00`; each body has the same preceding `0x0F` configuration bytes. Setup failures are logged but do not prevent ready state. Obtain any required credential/configuration through authorized provisioning; do not invent a factory default.

All command bodies in the following table are padded and encrypted with the derived session key.

| APDU header/selector | Body/interface | Result |
|---|---|---|
| `80 77 FF 00` | Reader selector byte | Detection; Classic normalizes UID length to `0x04`, alternate to `0x07` |
| `80 85 FF 84/85/86` | Derived key6, UID4, reader selector (`0x0B` plaintext bytes) | TDEA unwrap, then Classic AES block decode |
| `80 79 FF 80` through `80 79 FF 84` | UID7 and reader selector | Alternate identity chunks |
| `80 75 FF 81` | UID7 and reader selector | Authenticate/select alternate tag |
| `80 79 00 10/11/12` | Padding-only encrypted body | Capacity, percent, used length; four packed-BCD bytes each |
| `80 7B 00 00` | Padding-only encrypted body; wire Lc `0x08` | Four-byte big-endian life/count |
| `80 7F 00 12/11` | Four packed-BCD bytes | Write used length, then percent |
| `80 7D 00 00` | The four bytes returned by the life read, before padding/encryption | Retirement operation; successful response requires four zero bytes |

Packed BCD is most-significant decimal digit first, with the earlier digit in each byte's high nibble. Writes retain only the lowest eight decimal digits in four most-significant-byte-first packed-BCD bytes. Capacity is multiplied by 1000 for mm. Native BCD decode does not reject nibbles above nine. Classic block failures return local `0x01`, `0x02`, `0x03` by block position; alternate chunk failure `0x10`, authentication failure `0x06`, subsequent field failure `0x03`. Detection maps `0x9000` to success, `0x9EF5` to the special result `0x02`, other SWs to `0x01`. On `0x6985` or `0x6A88`, native recovery clears session/credential-cache flags and reinitializes, but does not retry the failed operation itself. There is no direct-board fallback.

## Remaining amount and writes

The scan's published geometric percentage comes from encoder travel between visibility hits and the area formula below, not a float stored in Classic blocks. Both versions round in binary32, narrow to a byte **before** clamping, change zero to one and cap above decimal 100. Application 1.5.2 failure/unavailable is `0xFF`; 1.1.3 failure is decimal 100. Motion prerequisites are in [workflows](modes-and-workflows.md).

### Native geometric estimate

Let D be the absolute difference between the two encoder positions sampled at the tag-visibility hits. Both versions raise a smaller D to the binary32 baseline 301.44000244140625 firmware-mm. Only 1.5.2 also clears its geometry-valid flag when applying that floor; 1.1.3 does not. Evaluate the area estimate in this order, using binary32 at every step:

1. `q = D / 6.28000020980835`
2. `area = ((q × 3.140000104904175) × q − 7234.56005859375) / 15988.880859375`
3. `rounded = area × 100 + 0.5`

Convert `rounded` to an unsigned integer, narrow to the low byte, then map zero to one and values above decimal 100 to 100. Division precedes multiplication by 100; algebraic reassociation can change rounding. Narrowing occurs **before** clamping; full-integer saturation would not reproduce it. A separate circumference estimate is diagnostic output, not the published percentage. Failed measurement publishes the version-specific unavailable value described above. Hosts reading `0x03` do not need to recompute either estimate, and neither is a calibrated mass or length measurement.

### Authenticated usage accounting

Only the secure-board 1.5.2 worker installs ongoing usage accounting; the pure direct-board worker does not run it. For a valid UID-length `0x04` record in a running accounting mode, the worker clears update-pending and requires nonzero nominal capacity N and initial measured percentage at most 100. It computes `Q = uint8(floor(100 × accumulated_mm / N))`, then `P = max(0, initial_percent − Q)`. Narrowing Q to a byte is part of the calculation. Changes update the RAM percentage only; this does not write Classic blocks.

For valid UID7 records, let N be nominal capacity in mm, C tag-used mm and `B = floor(103 × N / 100)`. Remaining percent P is zero if C ≥ B, otherwise `floor(100 × (B − C) / B)`. A valid geometric estimate A ≤ 89 with A + 10 < P corrects P to A + 5 and schedules `pending_mm = max(0, floor(N × (100 − P) / 100) − C)`. This correction uses **N, not the 103% base B**. Unavailable geometry does not trigger it. Preserve native 32-bit integer products before division rather than widening them. Published zero is still changed to one.

Mode-1 travel accounting uses negative encoder distance as feed travel, committing increments of at least 10 mm and retaining subthreshold residual; stop adds only positive residual below 500 mm. A separate mode starts by adding 15 mm and runs for approximately 30 seconds. These are native accounting rules, not physical capacity calibration.

Before a pending UID7 write, detection must produce length `0x07` and the exact saved seven-byte UID of a valid saved record. Mode 1 clears update-pending before probing and makes up to two attempts on detection status `0x01`; other modes make one, and status `0x02` is not retried. Wrong current UID length clears pending; wrong UID does not write. Zero capacity returns `0x05`, authentication failure `0x06`, write/life errors `0x03`. The writer updates changed used length first, then percent, acknowledging each shadow field only after its own `0x9000`. If used exceeds floor(110 × N / 100) with a nonzero life shadow, it reads life/count and, when nonzero, performs the `0x7D` operation and requires four zero response bytes. An already-zero life read succeeds without necessarily clearing the shadow. After an overall failure, the worker restores the old saved used-length shadow; that RAM restoration **cannot undo an earlier physical write**. These operations are nontransactional across fields and power loss.

Diagnostic counters distinguish attempts, successes and errors, gated by an unsigned candidate-used minus last-counted-used difference of at least 280 mm. Only detection-status-`0x01` failures on calls that accounted new travel increment the miss count. At 21 or more misses, reacquisition additionally requires successful detection and no valid current saved record. A valid paired-slot record (B/A/D/C) with the same UID length and UID blocks reacquisition; otherwise the worker attempts a read up to three times. A successful read republishes text and initializes usage with measured percentage 100. The miss count is cleared at the end of that detected/no-valid-record branch. None of this proves a tag write completed. Preserve errors and reconcile by rereading instead of blindly replaying writes.

There is no generic RS-485 tag writer in this contract. A direct Classic writer exists as machine code without a located installed caller; secure usage writes are a different reachable interface. Opcode `0x5C` writes credential/configuration EEPROM, **not spool data**. Never turn a transport checksum failure into a generic retry policy for tag writes or motor-moving scans.


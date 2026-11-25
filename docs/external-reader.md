# Separate external material reader

The external reader is a separate UART device, not either of the two onboard CFS radio front ends. Its recovered application is `rfd0_010_G21-rfd0_000_010.bin`, version 0.1.0, length `0x5230`, SHA-256 `23eeaa282f70afebfe1a1f88d990276e324fad0c4fdc29fe082c6a6809dd43ca`.

It supports **three application commands** at fixed address **`0x11`**: cached material read, version, and reset. It has no filament-motion API or per-query fresh scan. No reader, radio, printer, or original firmware code was executed in recovering this contract.

See [transport](transport.md) for the common frame format, [RFID and material](rfid-and-material.md) for material interpretation, and [maintenance](maintenance.md) for the separate update boundary. All wire values, byte offsets, and lengths here use hexadecimal; timings use ordinary units.

## UART and addressing

- 230400 baud, 8 data bits, no parity, one stop bit.
- Frame: `0xF7 | address:u8 | L:u8 | flags/status:u8 | opcode:u8 | payload | CRC8`.
- `L=payload_length+0x03`; total wire size is `L+0x03`.
- CRC-8: MSB-first polynomial `0x07`, initial `0x00`, no reflection or final XOR. It covers exactly `L` bytes beginning with the length byte: length, flags/status, opcode, and payload. Sync, address, and CRC are excluded.
- Incoming address must be `0x11` or `0xFF`; the application has no address assignment or persisted-address lookup.
- **Every reply uses source address `0x11`**, even for a broadcast request at destination `0xFF`.
- Incoming flags are stored and CRC-covered, but are not a dispatch gate. Runtime vendor requests use `0xFF`; normal reply status is `0x00`.

Do not treat the received address byte as the destination the host originally used. In particular, a request sent to `0xFF` does not make a source-`0x11` reply an unrelated device. Conversely, broadcasts can reach other device classes; prefer local `0x11` for reader-specific requests and correlate address, opcode, status, and payload length together.

The vendor external-material wrapper reads a configurable `addr` setting but sends literal `0x11`; that setting is not evidence that this application can be readdressed. Do not allocate a CFS box to the reader's fixed address on a shared bus.

## Complete application command set

`N` is request payload length. Send only the defined minimal request, even where firmware ignores surplus data.

| Opcode | Request payload / `N` | Successful response | Side effect |
|---|---|---|---|
| `0x02` | Empty / `0x00` | Status `0x00`; either cached `0x28` bytes, or exactly the seven lowercase ASCII bytes `unknown` | Drives PA6 low; does not start scanning or clear cache validity |
| `0x14` | Empty / `0x00` | Status `0x00`; exactly `0x1B` bytes: ASCII `010` (`30 31 30`) followed by `0x18` zero bytes | None |
| `0x0B` | Exactly `[0x00]` / `0x01` | **No reply** | Disable interrupts and request MCU system reset |

Response sizes:

| Response | `N` | `L` | Total wire bytes |
|---|---:|---:|---:|
| Material cache | `0x28` | `0x2B` | `0x2E` |
| `unknown` | `0x07` | `0x0A` | `0x0D` |
| Version | `0x1B` | `0x1E` | `0x21` |

Neither material response includes a trailing NUL. Treat `unknown` as unavailable **only when the entire payload is exactly those seven bytes**. A length-`0x28` cache beginning with the same letters is still data. A CRC error, timeout, wrong opcode, or truncated body must not become a valid material record.

`0x14` is not the main CFS application's version-and-serial format. It contains no serial number or UUID. The file's descriptor `rfd0_000_010`, packaged filename prefix `rfd0_010_G21`, and runtime `010` response are three distinct identifiers; none proves what the separate bootloader reports.

`0x0B` reads the first payload storage byte without checking that one was supplied. A zero-length reset request can therefore act on stale storage. Always include the defined byte. A nonzero byte is ignored. The observed action is SYSRESETREQ, not a direct loader jump, proven loader hold, or flash operation; no ACK is expected and no automatic retry is justified by its absence.

### Unsupported and malformed traffic

CRC-valid opcodes other than the three above are ignored without a reply or command side effect. This explicitly includes `0xA0..0xA3`, `0xF0`, and `0x56`. The regular reader application therefore does **not** implement CFS discovery, readdressing, boot geometry, erase, upload, verify, or start commands.

For an assembled frame with bad CRC:

- Address `0x11`: reply from `0x11`, echo the opcode, status `0x02`, empty payload (`L=0x03`).
- Address `0xFF`: no reply.

The parser requires `L>=0x03` but does not enforce a safe maximum against its fixed `0x64`-byte payload storage. This is not permission to send oversized frames. Wrong address, malformed prefix, incomplete input, and dropped receive-ring bytes do not have a guaranteed error response. The prefix filter and frame assembler have separate timeout/reset layers; this is not a transaction timeout or freshness mechanism.

## Cache freshness and PA6

The scanning loop runs independently of host queries:

1. Select a card. On failure, wait nominally 100 ms and try again.
2. On success, wait nominally 50 ms and select again. There is no UID-equality check between the two selections.
3. Enter the authenticated read state; it performs another preparatory selection and the material read. Up to two read attempts are made, with 20 ms waits after failures.
4. Successful read publishes a valid cache, raises PA6, and enters presence monitoring.
5. Monitoring waits nominally 100 ms between successful selections. Two consecutive selection failures lead to twenty 100 ms waits before scanning resumes.

**Removal does not invalidate the last successful cache or lower PA6.** A subsequent failed read cycle or reset can invalidate it. A valid `0x02` command lowers PA6 before returning the cache, but does not clear its valid flag or request a new read. Repeating `0x02` can return old material indefinitely while no new read replaces it.

PA6 is therefore a **latched successful-read indication**, not a physical card-presence input. The vendor wrapper can schedule a read on a rising configured notification-pin event and also exposes an explicit external-material query. Firmware plus that caller does not establish the actual PA6-to-host wiring. Do not equate a configured host pin's level with a verified electrical connection.

There is no per-query freshness counter, refresh command, unsolicited UART tag announcement, encoder, usage-counter writer, feeder-position input, or filament interlock. No snapshot lock protects the cache against the parser interrupt while the scanning loop is filling it. A host should preserve raw bytes and mark freshness as unknown rather than claiming each query represents a newly presented tag.

## Two cache encodings

The returned length is the same for both formats; length alone does not identify their semantics.

### Exact ATQA bytes `04 00`: authenticated material path

The reader selects this path only for the ordered ATQA bytes `04 00`; it is a firmware predicate, not a general card-type detector. It reads blocks `0x04`, `0x05`, and `0x06` with MIFARE-style authentication:

- Repeat the four ordered UID bytes four times to form a length-`0x10` input.
- AES-128 encryption under the image's derivation key produces a block; its first `0x06` bytes are supplied as Key A to the RF front end.
- Decrypt each length-`0x10` data block independently under a separate AES-128 key. There is no IV, chaining, tag MAC, or plaintext-schema validation on this path.
- Cache block `0x04` in offsets `0x00..0x0F`, block `0x05` in `0x10..0x1F`, and only the first `0x08` bytes of block `0x06` in `0x20..0x27`.

The external application then makes two deliberate overrides:

| Cache offset | Forced value |
|---|---|
| `0x0B` | ASCII `1` (`0x31`) |
| `0x11` | ASCII `0` (`0x30`) |

These alter the apparent material and color fields. **Do not apply them to the onboard CFS cache**, and do not mistake this external response for unchanged tag plaintext. Onboard reader acquisition is a different implementation.

A decoded but unentered scanner state also supports a raw/default-key path with a length-`0x06` all-`0xFF` authentication key and no block decryption. No in-image writer of that state was established. It is not a user-selectable UART mode, a proven automatic fallback, or a reason to silently retry authentication with another key.

The exact reader image locates its key lanes at physical `0x08007E84`, in four length-`0x0C` rows. Concatenating the first `0x04` bytes from each row forms the derivation key; the next `0x04` bytes from each row form the separate decryption key, in stored order. This documents provenance and ordering without embedding key material. Never substitute onboard CFS key material simply because the record lengths resemble one another.

### Other ATQA values: uppercase hexadecimal path

Other ATQA values read blocks/pages `0x04` and `0x08` without that authentication/decryption wrapper. Take all `0x10` bytes of the first response and the first `0x04` bytes of the second response, then encode each byte high nibble first as uppercase hexadecimal. The resulting `0x14` raw bytes become exactly `0x28` ASCII characters.

That response is **not the fixed-width Classic material plaintext layout**. Do not split it into material/color/date fields merely because it has the same length. The remaining `0x0C` bytes of the second read are not included in the cache.

### Authenticated plaintext field boundaries

These boundaries come from the firmware's diagnostic label table and the vendor consumer. Ranges are half-open: the end offset is excluded. They describe the material-layout path, not the alternate hexadecimal encoding.

| Cache range | Length | Label / interpretation boundary |
|---|---:|---|
| `[0x00,0x01)` | `0x01` | Month |
| `[0x01,0x03)` | `0x02` | Day |
| `[0x03,0x05)` | `0x02` | Year |
| `[0x05,0x09)` | `0x04` | Supplier |
| `[0x09,0x0B)` | `0x02` | Batch |
| `[0x0B,0x11)` | `0x06` | Material identifier; first byte overridden |
| `[0x11,0x18)` | `0x07` | Color; first byte overridden |
| `[0x18,0x1C)` | `0x04` | Length field |
| `[0x1C,0x22)` | `0x06` | Number field |
| `[0x22,0x28)` | `0x06` | Reserved field |

The application does not validate these as numeric dates, lengths, colors, or material IDs. Labels alone do not establish every field's unit or database interpretation. Consult [RFID and material](rfid-and-material.md) for host-side material semantics, and preserve invalid/unrecognized bytes instead of replacing them with plausible values.

## Persistence and update boundary

The application address and key inputs are fixed in the image. Tag cache, scan state, and authentication-sector cache are volatile. No application flash programming, emulated EEPROM, downloaded-key provisioning, or tag writer was recovered. During startup, the MCU copies a `0x0C`-byte factory ID from three system-memory words into volatile RAM. It is not a device identity embedded in the shared application package, and no application command returns it.

File offset zero maps to `0x08003000`; bytes below that address are not in this application package. Its CRC16/length metadata follows the same packaging calculation described in [maintenance](maintenance.md#image-metadata-and-crc16), but this does **not** establish its bootloader's transport, installation bounds, startup window, private erase, or recovery policy.

The vendor updater routes type `0x04` to RFID images, assigns addresses starting at `0x11`, and attempts its staged update exchange. That is host intent. It does not override the receiver evidence: the running reader application ignores `0xA0..0xA3` and `0xF0`, and `0x0B [0x00]` only requests reset. Do not send the public CFS `050_G32` loader protocol to this reader on the assumption that reset guarantees entry into an identical loader.


The firmware's diagnostic formatter routes characters to a stub that returns without UART output. Its diagnostic strings do not establish unsolicited application-UART material reports. The external reader's internal loader implementation is outside scope; physical RF interoperability, electrical signal behavior, tag freshness, and reset-to-loader timing were not tested.

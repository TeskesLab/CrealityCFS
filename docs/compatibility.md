# Compatibility and firmware profiles

This reference describes inspected builds, not every product sold under the CFS name. The primary profiles are the main applications whose version prefixes are ASCII `113` and `152`, conventionally displayed as **1.1.3** and **1.5.2**.

## Select a profile deliberately

1. Establish an addressed material-box identity with the [addressing protocol](addressing.md). Its type and application/loader mode identify the endpoint family, not its software revision.
2. For a main application, query command `0x14` and retain the full response. The first three bytes are the version prefix; the remaining nineteen bytes are its serial cache.
3. Select the matching main-application profile. Do not silently treat an unknown prefix as 1.5.2.
4. Branch on reply status before decoding any status payload. A four-byte `0x0A` data field is not enough to identify 1.1.3: status `0x30` events have four bytes on both versions.

A separate reader at `0x11` returns a different version layout. A loader's addressing mode and `0xF0` version information are another interface. Neither is a main application's `0x14` format.

## Main application differences

Numeric command/status values in this table are hexadecimal.

| Behavior | 1.1.3 | 1.5.2 |
|---|---|---|
| Normal successful `0x0A` data | Four bytes: signed temperature, humidity, event byte, operation state | Six bytes: the same first four plus connection-input and secure-tag depletion/cache-inhibit masks |
| `0x0A`, status `0x30` | Four independent slot-event codes | Same shape; do not decode as environment data |
| `0x03`, present filament but no usable card | Can return `100`, also a valid full-spool percentage | Unavailable state is `0xFF` |
| `0x03`, physically absent selected slot | `0x00` | `0x00` |
| `0x03`, unselected/busy slot | `0xFF` | `0xFF` |
| Command `0x12` distance feed | Unsupported | Implemented, but the speed byte is ignored by the observed routine |
| `0x5B` hardware identifier | Unsupported | Eight-byte identity derivative; not the addressing UUID |
| `0x5C` credential write | Unsupported | Sixteen-byte persistent credential write/readback |
| `0x5D` structured factory test | Unsupported | Motion-capable fixture sequence with unframed JSON output |
| Onboard RFID paths | Deployed direct-reader/Classic path; no installed secure-board session path | Direct-reader path plus optional FM15L013 secure-board path |
| Classic AES key source | Same forty-eight interleaved source bytes as 1.5.2, at a different image location | Same source bytes, different location |
| Direct-reader UID comparison | Actual four-byte UID comparison | Missing UID-length initialization makes the normal direct two-hit comparison ineffective |
| High-block auxiliary diagnostic cache | Concrete block-8 float/valid producer exists | No installed in-bounds producer; high-block diagnostic operations omit the work |
| Indicator `0x0B` | Slot byte is validated but slot-specific patterns act on CURRENT | Same alias defect; no arbitrary-slot indicator API |
| Direct commands `0x06` / `0x09` | Selector and length defects exist | Same classes of defects; no reliable arbitrary-slot/speed API is inferred |
| Factory mode `0x0C` / `0x50` | One-byte internal-flash parameter ID 6 | One-byte internal-flash parameter ID 6 |

The old no-card value `100` is deliberately retained. Do not convert every old `100` response to “card missing,” and do not manufacture connection/depletion-mask fields for the old four-byte status. The newer depletion mask is not a scan-progress indicator; a clear bit does not prove acquisition completed.

Both versions use the main application's normal frame shape, CRC-8 convention, A–D slot masks and application addressing family. Shared framing does not make every payload, diagnostic feature, error result or motion algorithm equivalent.

## Do not infer hardware from a filename

The supplied G30 and G32 **application** packages are byte-identical within version 1.1.3 and within version 1.5.2. Application byte equality does not establish:

- identical bootloader code or update/recovery policy;
- identical connectors, transceiver characteristics or board revision;
- the presence of a secure RFID chip on a particular unit;
- the active printer filesystem slot;
- identical mechanical calibration or behavior under load.

The main application files start at physical flash `0x08003000`. They omit the first twelve KiB of boot flash. See [Maintenance](maintenance.md) before implementing any update path.

## Host software is not the protocol specification

Where this guide compares **Host A** and **Host B**, it means the earlier and later printer-side integrations. These labels describe host bookkeeping, not a physical device, an active boot slot or a reliable way to select the CFS wire profile. Select the application profile from its own identity/version information. Relevant host differences include:

- The later host understands the six-byte status and stores the secure-tag depletion/cache-inhibit mask as `slot_rfid_scrap`; the earlier layout has no such byte.
- Both native main applications prefix six hardware-fault fields with category bits. The original host does not mask all of these correctly. Decode the native fields as specified in [Sensors](sensors.md), rather than copying a printer UI's false fault reports.
- A host can retain stale metadata or overwrite a newly refreshed remaining-length cache with an old saved zero. These are host bookkeeping effects, not additional on-wire tag fields.
- Printer IPC IDs, G-code registration lists and display object layouts can change while a bus opcode stays the same. Do not send an IPC message ID as a CFS command.

An independent implementation should reproduce the documented wire behavior and expose real errors. It does not need to reproduce unrelated host UI bugs or every Creality-specific macro.

## Support boundary

Read-only framing, identity, cache and sensor queries can be implemented without enabling factory mode or flashing a device. Movement needs the version-specific sequencing, guards and completion handling in [Modes and workflows](modes-and-workflows.md).

For unknown firmware, preserve raw bytes and report the unsupported profile. Do not probe by issuing motion, fixture, credential or update commands. New product names or newer firmware require new evidence; their compatibility is not implied here.


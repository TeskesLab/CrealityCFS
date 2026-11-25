# Storage and persistence

There are four independent persistence domains: application RAM caches, external EEPROM, the MCU internal-flash parameter page, and printer-host files. Tag-side authenticated usage is a fifth, separate domain described in [RFID and material](rfid-and-material.md). No single acknowledgment commits all of them. Numeric offsets, sizes, IDs and wire values in this chapter are hexadecimal; timing and physical units are decimal.

## Domain map

| Domain | Contents | Lifetime/commit boundary |
|---|---|---|
| CFS RAM | Debounced presence, pending scan FIFO, material/auxiliary caches, percentage, usage shadows, environment/encoder caches, session and credential cache flags | Reinitialized on application restart; not a persistent database |
| External EEPROM | Serial information, fault snapshot, 1.5.2 secure credential/configuration, destructive boot-test canary | Byte/page writes; partial completion possible |
| Internal flash | Parameter ID `0x05` version and ID `0x06` operating/factory mode | Whole-page erase followed by word programming |
| Printer-host filesystem | Material database, configuration, manual/cached slot descriptions, tool maps, saved last command | Independent host file writes |
| Tag | Secure UID7 used amount, percent and life/count | Separate authenticated operations; no transaction across fields |

A cache can be old even while a device is reachable. EEPROM failure can leave zero or previously cached credentials; tag write failure can leave one field updated; a host save failure can leave RAM newer than disk. Treat each result according to its own boundary.

## Volatile application state

In 1.5.2 the scheduler initializes a `0x11C`-byte RAM structure, seeds presence/debounce states from GPIO and fills its four-slot FIFO with `0xFF`. Its per-slot material cache is `0x28` bytes, auxiliary cache `0x08` bytes, with a separate percentage byte. Starting a measurement clears that slot's text/auxiliary cache and sets percent unavailable (`0xFF`). Ordinary host cache reads do not acquire a tag or make these values durable.

The per-slot secure usage record contains accumulated/pending used mm, selected nominal capacity, service timestamp, encoder baseline, miss count, measured percent, pending/mode/valid flags, a copied tag record and UID. It is RAM state, not the EEPROM fault record. Secure session material and the ready flag are likewise volatile; startup/recovery must authenticate again. The direct 1.5.2 auxiliary source and UID-length byte remain cold-zero through installed in-bounds routes, as detailed in [RFID](rfid-and-material.md); persistence must not be invented to explain them.

The environment cache can freeze in TEST or after a startup fault, and failed reads can republish old samples. Encoder distance is relative to a RAM reference. Neither implies durable sensor history; see [sensors](sensors.md).

## External EEPROM allocations

Both mapped applications use PE5 SCL and PE6 SDA, GPIO IDs `0x45`/`0x46`. The logical address space is `0x800` bytes, with `0x10`-byte write pages.

| Logical byte range, inclusive | Size | Use |
|---|---|---|
| `0x000–0x012` | `0x13` | Serial-information read at boot |
| `0x060–0x06F` | `0x10` | Secure-chip credential/configuration in 1.5.2; no installed secure allocation in 1.1.3 |
| `0x080–0x0AB` | `0x2C` | Motor/fault snapshot |
| `0x7F0–0x7F3` | `0x04` | Destructive presence/readback canary |

These are **storage offsets**, not RS-485 opcodes. In particular EEPROM offset `0x80` is unrelated to opcode numbering, and parameter ID `0x06` does not denote a six-byte EEPROM record.

The fault monitor writes the `0x2C`-byte motor snapshot after a persistent block condition, alongside stopping the hub and releasing feeders. This is a last recorded fault snapshot, not a journal of every error or a complete power-loss resume record. Its telemetry includes feeder voltage/threshold/request/correction and hub target/measured speed/output; these values have firmware engineering units, not independently calibrated current or motion measurements.

### EEPROM transaction behavior

Address selection uses control byte `0xA0 | (((address & 0x7FF) >> 0x08) << 1)` and the low address byte. Reads issue a repeated START but use fixed read control `0xA1`, including for nonzero banks. This is observed software behavior, not hardware confirmation that all banks are read as intended. Intermediate read bytes are ACKed, final byte NACKed. Writes split at page boundaries, STOP and address-poll up to 1000 times; data NACKs stop the operation. Successful read/write returns `0x00`, failure `0x01`. There is no normal readback, range guard, rollback or encompassing transaction mutex. Effective addressing wraps to the low eleven address bits.

The startup test reads the little-endian word at `0x7F0`, chooses `0xBAADF00D` if it was `0xDEADBEEF`, otherwise chooses `0xDEADBEEF`, writes and rereads it. It does not restore the original contents and ignores individual transfer results before comparison. Up to three initialization attempts are made. The recorded EEPROM boot status is `0x00` for success, `0x01` for missing/not-ready, `0x02` for mismatched readback. A successful boot therefore still writes this reserved page; there is no wear rotation.

### Secure credential cache and opcode `0x5C`

In 1.5.2, the credential getter reads `0x10` bytes at EEPROM `0x60` only when its cache-valid flag is clear. It sets that flag only on a successful read, but always returns length `0x10` and copies the existing cache when requested. A failed read therefore yields cold-zero bytes before the first successful population, or an old cache afterward. Returned length is not proof of successful EEPROM I/O.

The credential writer ignores the write return and invalidates the cache flag. Opcode `0x5C` writes then calls the getter and returns `0x10` bytes of readback. This is credential/configuration mutation, **not a generic RFID spool writer**, and can break secure authentication if EEPROM and chip provisioning no longer agree. Do not expose it as an ordinary scan or recovery operation. Factory credential contents are not supplied by these application images, and none are published here.

## Internal-flash parameter page

Both 1.1.3 and 1.5.2 install one `gd32_flash` page at physical `0x0807F800`, size `0x800`. Only the first `0x10` bytes belong to the installed parameter group:

| Page offset | Size | Meaning |
|---|---|---|
| `0x00` | `0x04` | Little-endian 32-bit stored CRC, containing zero-extended CRC-16 |
| `0x04` | `0x04` | Little-endian payload length, normally `0x00000008` |
| `0x08` | `0x04` | Parameter ID `0x05`: little-endian firmware/schema version |
| `0x0C` | `0x01` | Parameter ID `0x06`: operating/factory mode |
| `0x0D` | `0x03` | Unmapped padding included in CRC |

Defaults are version `0x00010103` for 1.1.3 and `0x00010502` for 1.5.2, mode `0x00`. CRC is CRC-16/XMODEM: polynomial `0x1021`, initial `0x0000`, nonreflected, xorout `0x0000`, over fixed bytes `0x08:0x10`. The validator compares against the entire stored little-endian 32-bit CRC and does not use the stored length to choose its checksum span. Do not assume padding is always zero after an invalid read; it is part of the CRC-covered RAM image.

Opcode `0x0C` reads the **one-byte** mode; opcode `0x50` changes it only if different and follows the native reset path. This parameter is neither the secure-chip strap nor an EEPROM field. The numerical generic mapper accepts wider ID ranges, but only IDs `0x05` and `0x06` have installed storage; the implementation lacks sufficient field/group/copy-size bounds checks to make those generic ranges a supported interface.

### Load, migration and commit failures

A missing backend or invalid CRC selects defaults and commits them. A valid page caches the prior version; if the version differs, only ID `0x05` is updated and committed, preserving stored mode and padding. A version upgrade is therefore not a factory-mode reset.

RAM mutation and durable commit are distinct. Commit computes the payload CRC and skips storage if it matches the RAM header CRC. Otherwise it updates the RAM CRC/length **before** erasing the page and programming four little-endian words. There is no journal, generation counter, wear leveling, readback, rollback or software mutex. Flash-program failures are logged but are not propagated as a checked commit failure. The advanced RAM header can make a later unchanged commit skip retrying failed physical storage. Loss of power between erase and final programming can invalidate the whole group; a subsequent boot can select defaults rather than recover the prior mode. No atomicity or successful persistence should be inferred from an ACK/reset request.

## Host configuration, material maps and saved state

The printer host owns transport selection, motion/purge configuration, material metadata and logical tool mapping. These are not CFS EEPROM parameters. Native files include `box.cfg`, `material_database.json`, `tn_data.json` and `material_box_info.json`. Filenames identify host artifacts, not firmware wire endpoints.

`tn_data.json` is an object whose `base_data` contains per-box `T1`–`T4` records. Both host revisions generate four common saved arrays: `vender`, `remain_len`, `color_value`, `material_type`. Each has four slot cells. The newer host (HOSTB) adds the fifth saved array `change_color_num`; the older host (HOSTA) does not generate it. Fresh cells are string `'-1'`, not numeric minus one. Per-box version/serial/mode and live state/environment/UUID fields are separate from those arrays. Root `enable`, `tnn_map` and `last_cmd` can coexist; surrounding orchestration may also own `remain_material`.

Wire replies project into host caches; manual/cached material assignments can survive without RFID identity. On cache synchronization, an absent presence bit sets live vendor to `none` without clearing every other cell. A present slot with saved vendor `'-1'` becomes `unknown`, but saved color/material may still be restored. A presence bit plus a saved description is therefore not evidence of a current tag read.

Same-material groups use exact full material ID and exact color, without case or `#` normalization. A usable manual/cached record with vendor `unknown` can still participate. The database lookup drops the material prefix; its displayed group label comes from native `meterialType`, not necessarily `name`. Unknown/unavailable database data leaves an empty label, not a fabricated generic identity. Auto-refill consumes these groups but applies additional enable/availability policy; grouping itself does not move filament.

The newer host publishes `Tn_data['map']` as the same mapping object, not a copy; the older host does not publish that key. Saved physical-to-logical mappings and `last_cmd` feed power-loss restoration, whereas the separate resume token is RAM-only. Do not conflate either with the CFS flash mode or assume native restore proves physical filament position after an outage. See [integration](integration.md).

### Host-file failure semantics

Accepted saves read the current document, replace the selected root key and write through ordinary `open(..., 'w')`. Successful reads preserve other root keys; caught read failure may start from an empty object. There is no temporary-file/rename transaction. RAM is changed before persistence and is not rolled back on failure; interruption can truncate or partially replace JSON.

A non-`none` slot-array change can persist `base_data`; assigning `none` changes only live state in the traced path. Some saved-data mutators themselves are RAM-only. HOSTA adopts existing legacy `base_data` unchanged. HOSTB alone uses a recursive `change_color_num` key test across the entire root to gate migration: a marker in another box or outside `base_data` can bypass migration, while missing/malformed original box data can cause indexing failures during migration. This is not general malformed-file repair or schema validation. An independent implementation may choose explicit schema validation and atomic saves, but must label that as **host policy**, not native behavior. Avoid silently replacing uncertain stored identity with a successful RFID claim.

Secure applet internal storage, device-specific EEPROM contents and real power-fail electrical behavior are not established by this interface contract.

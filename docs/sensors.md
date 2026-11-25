# Sensors and status

This chapter separates electrical observations from mechanical names. Command framing and request lengths are in [commands](commands.md); motion-dependent interpretation is in [modes and workflows](modes-and-workflows.md). Numeric wire values, byte offsets and lengths below are hexadecimal; physical quantities and time intervals are decimal.

## Eight slot inputs and two buffer inputs

All ten inputs are configured with pull-ups. A low electrical level means asserted/present. The two bank accessors return **raw** levels; wire presence masks instead identify asserted inputs with set bits. In each slot mask, bits `0x01`, `0x02`, `0x04`, `0x08` correspond to A, B, C, D.

| Slot | Bank 0: material/insertion | Bank 1: path/hub entry |
|---|---|---|
| A | PD7 | PE10 |
| B | PD6 | PE9 |
| C | PD5 | PE8 |
| D | PD4 | PE7 |

Bank 0 drives insertion/removal debounce, readiness and idle indicators. Bank 1 detects the selected filament approaching the hub: feeding waits for low and retracting waits for high. The fixture names bank 0 `LIMIT_1` through `LIMIT_4`, but bank 1 `LIMIT_8` through `LIMIT_5` in slot order. Human labels such as “gate,” “post-gears” and “hub sensor” do not establish extra channels. These paths contain eight slot inputs plus the following two buffer inputs, **not** an additional independent shared gate switch. Exact physical placement and local harness revision require hardware evidence.

The packed buffer value uses asserted-low PD8 as bit `0x01` and asserted-low PD9 as bit `0x02`:

| Packed code | PD8 | PD9 | Native interpretation |
|---|---|---|---|
| `0x00` | high | high | Neither asserted; preserve distinctly |
| `0x01` | low | high | Full |
| `0x02` | high | low | Free |
| `0x03` | low | low | Both asserted; preserve distinctly |

The wait helper requires ten consecutive matching samples, with requested 1 ms delays between nonterminal samples. It returns match or timeout, not a normalized two-state position. Both IRQ callbacks use **falling edges**, not both edges. When enabled, PD9 falling starts normal-feed hub motion at the configured buffer effort and increments the free-event count; PD8 falling clears active/free count and brakes/stops the hub. Neither callback rereads the other input. Both-low at boot records a buffer fault. See [integration](integration.md) for why these autonomous effects matter.

## Environment cache and status `0x0A`

Normal `0x0A` data is:

| Offset | Meaning |
|---|---|
| `0x00` | Signed temperature byte, degrees Celsius |
| `0x01` | Unsigned relative-humidity byte, percent |
| `0x02` | Event |
| `0x03` | Operation state |
| `0x04` | Bank-1 connections/path-presence mask, application 1.5.2 only |
| `0x05` | Secure-tag depletion/cache-inhibit mask, application 1.5.2 only |

Application 1.1.3 returns `0x04` data bytes; 1.5.2 returns `0x06`. The separate status-`0x30` reply has `0x04` slot-event bytes, **not** this layout. Neither reply supplies a sample timestamp or freshness flag. Opcode `0x52` also returns the same two cached environment bytes under response opcode `0x53`; it does not acquire a new sample. Incoming `0x53` sets separate fixture reference values and does not activate the diagnostic sensor override.

In byte `0x05`, bits `0x01`, `0x02`, `0x04`, `0x08` identify A, B, C, D. A bit is set only when the slot has debounced filament presence, a valid saved usage record, saved UID length `0x07`, and a zero saved life/count shadow. The same predicate [inhibits RFID cache publication](rfid-and-material.md#cached-host-interface). This is not the scan FIFO or update-pending state: a clear bit does **not** prove acquisition completed. Application 1.1.3 has no such field.

The environment worker requests a reading approximately once per second only outside TEST and with the startup environment-fault gate clear. TEST or a boot fault freezes the cache. Its persistent float locals start at zero; acquisition failures leave them unchanged, and the worker ignores the failure result before republishing their truncated low bytes. Thus a transient fault can repeat an old value, or zero before the first successful worker sample. A later successful diagnostic probe does not clear the boot gate automatically. Repeated plausible readings are not proof of a functioning sensor.

Both mapped applications use PC11/PC12 for the AHT-compatible `i2c0` bus, seven-bit address `0x38`. Acquisition writes `AC 33 00`, waits 200 ms, then reads `0x06` bytes. From bytes b0…b5, humidity raw is `(b1<<12)|(b2<<4)|(b3>>4)` and temperature raw is `((b3&0x0F)<<16)|(b4<<8)|b5`. Physical conversion is RH = raw × 100 / 2²⁰ and T = raw × 200 / 2²⁰ − 50. Byte b0, busy/status bits and CRC are not checked. Both transfers must return exactly one message before either output changes. The firmware's AHTxx/AHT10/AHT30 names do not prove the fitted part suffix.

## Encoder distance, polling and reset

Opcode `0x0E` action `0x01` returns `0x04` bytes: **big-endian IEEE-754 binary32 signed distance**. This differs from the little-endian integers used elsewhere. It reads a cache, not a new SPI transaction. Action `0x00` rebases the reference, immediately zeroes the float cache and acknowledges with no data; other actions return status `0x01`. Reset does not stop the sampling timer or disable the interface and retains angle/turn tracking. Do not treat it as a physical movement or power reset.

Both applications normally sample every 20 ms. Faster host polling produces repeated cache values and has no freshness guarantee. The sensor is bit-banged SPI mode 3, MSB first, PB12 CS, PB13 clock, PB14 MISO, PB15 MOSI. Angle request `30 03 00 00` decodes `BE16(RX[0x02:0x04]) >> 1`, giving a 15-bit angle. Turn tracking unwraps changes beyond half of the `0x8000`-count range; exact half-turn differences are not unwrapped. Signed position minus reference is converted to float and scaled by binary32 bits `0x3A4EBCD3`: approximately 0.000788641337 firmware-mm/count, or 25.8422 firmware-mm per nominal wheel revolution. Feed is negative and retract positive in the motion comparisons. Roller slip, effective circumference and real calibration tolerance were not measured.

Startup checks writable register `0x01` for `0x7E`, writes that value and rereads if necessary. This is not an immutable silicon-ID test. Setup is one-shot: allocation or communication failure sets the encoder boot fault, without clearing the latch or necessarily cancelling an already-started timer. Runtime sampling has no sample-validity/error check; a bad transfer can enter the unwrap/cache calculation. The public read has no separate readiness or fault field.

## Hardware status `0x15`

The response has `0x10` bytes. The first `0x06` are category prefixes ORed with the native raw fault value; the remaining `0x0A` are zero in these implementations.

| Offset | Prefix | Category |
|---|---|---|
| `0x00` | `0x10` | EEPROM |
| `0x01` | `0x20` | Environment |
| `0x02` | `0x30` | Encoder |
| `0x03` | `0x40` | Buffer |
| `0x04` | `0x50` | Left RFID reader |
| `0x05` | `0x60` | Right RFID reader |

For ordinary nibble-sized faults, high nibble identifies the category and low nibble is the fault; zero low nibble means no recorded fault. Preserve the raw bytes and reject unexpected category encodings rather than discarding high bits: the native builder ORs raw values without first masking them. A healthy initialization report is not continuous proof that readings, filament motion or authentication are succeeding. LCD initialization faults and the operation-context TEST flag are separate state, not interchangeable fields.


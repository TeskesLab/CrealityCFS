# Wire examples and decoding checks

These packets are **constructed examples**, not newly captured device traffic. Addresses, sensor values and the UUID are illustrative. No packet shown here was sent to a device during this work.

All bytes in packet blocks are hexadecimal. The layout and CRC are defined in [Transport](transport.md): `F7 | address | L | status/mode | command | data | CRC`. Total packet size is `L + 3`; payload size is `L - 3`. CRC covers `L` through the last payload byte, excluding `F7` and the address.

## 1. Query normal state

Main-application request to box address `0x01`:

```text
F7 01 03 FF 0A 5C
```

A successful **1.5.2** reply with temperature 25 °C, humidity 45%, no event, idle state, and clear connection and secure-tag depletion/cache-inhibit masks:

```text
F7 01 09 00 0A 19 2D 00 00 00 00 83
```

Its six data bytes are `19 2D 00 00 00 00`. `L=0x09` includes those six bytes plus status, command and CRC.

For **1.1.3**, the same first four values have a four-byte payload:

```text
F7 01 07 00 0A 19 2D 00 00 DD
```

Do not invent the newer two masks for this reply. Select the profile from the version information, not from length alone.

## 2. Distinguish an event from normal state

A reply to the state request can instead have status `0x30` and four slot-event values:

```text
F7 01 07 30 0A 01 00 00 00 7D
```

This example reports event `1` for slot A and no change for B–D. It is **not** temperature 1 °C followed by three more state fields. Both firmware profiles use this four-byte event shape. Event status must be processed before selecting the data decoder.

See [Commands](commands.md) for event meanings and [Modes and workflows](modes-and-workflows.md) for deferred completion.

## 3. Read the material-input bank

Request bank `0x00`:

```text
F7 01 04 FF 08 00 DB
```

Reply:

```text
F7 01 04 00 08 05 EB
```

Payload `0x05` has bits 0 and 2 set: the active-low inputs for slots A and C are active. The firmware has already inverted the GPIO levels. Do not invert this bitmask again. Selecting bank `0x01` reads the separate connection-input bank; it is not another slot mask.

## 4. Read the buffer endpoints

Request:

```text
F7 01 03 FF 05 71
```

The successful response contains one raw two-bit value. Preserve all four combinations `0`, `1`, `2`, `3`; a value is not a single generic “filament present” Boolean. [Sensors](sensors.md) defines the two bits and their effects on buffer control.

## 5. Read a scaled encoder position

Action `0x01` reads without resetting the baseline:

```text
F7 01 04 FF 0E 01 A2
```

A successful illustrative position of 12.5 is represented as big-endian binary32 `41 48 00 00`:

```text
F7 01 07 00 0E 41 48 00 00 1B
```

Decode the four payload bytes as a **big-endian IEEE-754 float**, not as a little-endian integer. This is the firmware's scaled position; it is not evidence of a mechanically calibrated 12.5 mm travel. Action `0x00` resets the baseline and must not be inserted automatically before polling.

## 6. Handle the environmental reply-code exception

Request command `0x52`:

```text
F7 01 03 FF 52 D3
```

Successful reply uses **command `0x53`**, not `0x52`:

```text
F7 01 05 00 53 19 2D 5D
```

The payload is temperature 25 °C and humidity 45%. Temperature is signed int8; humidity is unsigned. A length error can instead use the original request opcode. Do not discard a valid successful environmental reply merely because its command differs from the request. The values are cached, not a guaranteed fresh acquisition.

## 7. Address identity and discovery

Unicast identity request `0xA3` uses request mode `0x00` in the original addressing interface:

```text
F7 01 03 00 A3 DD
```

Illustrative identity response:

```text
F7 01 11 00 A3 01 00 00 11 22 33 44 55 66 77 88 99 AA BB 08
```

The fourteen data bytes are type `0x01`, application mode `0x00`, and the twelve-byte UUID `00 11 22 33 44 55 66 77 88 99 AA BB`. A leading zero does not invalidate the UUID. Do not replace this identity with the eight-byte hardware derivative returned by newer command `0x5B`.

Discovery of an **unassigned** main application at `0xFE` uses an inclusive range whose low and high endpoints are both `0xFE`:

```text
F7 FE 05 00 A1 FE FE F8
```

This is not a whole-bus enumeration request for an already addressed application. Assignment changes the address; see [Addressing](addressing.md) for the UUID match, new-address reply and bounded initialization sequence.

## 8. Communication-test wraparound

Request test value `0xFF`:

```text
F7 01 04 FF 55 FF CD
```

The successful reply contains `(0xFF + 1) modulo 256`, hence one **zero** data byte:

```text
F7 01 04 00 55 00 15
```

A zero data byte is not a timeout, missing packet or false acknowledgement.

## 9. Parse an RFID record without losing its boundaries

Select slot A with mask `0x01`:

```text
F7 01 04 FF 02 01 5E
```

Illustrative successful response:

```text
F7 01 2E 00 02 41 3A 41 42 31 32 34 30 32 37 36 41 32 31 30 30 30 30 31 30 30 32 39 37 32 44 30 33 33 30 30 30 30 30 30 31 30 30 30 30 30 30 3B 87
```

The payload is ASCII `A:` followed by forty material-record bytes and a semicolon. Those forty bytes are described in [RFID and material data](rfid-and-material.md). A mask is not a zero-based slot index: mask `0x01` selects A, while mask `0x00` selects nothing.

The alternative values `none`, `unknown` and `busy` are explicit sentinel records. This example deliberately requests one slot: the complete single-slot payload length distinguishes a forty-byte value from a sentinel, even if it begins with sentinel letters. Aggregate replies have no escaping or per-entry lengths and can be ambiguous when opaque data imitates delimiters or later slot labels. Require a unique complete parse against the requested slot order; if none or several exist, preserve/reject the ambiguous data and use separate one-slot cache queries rather than guessing.

## Offline checks for an implementation

A receiver should produce the same packet and payload when each example is delivered:

- in one serial read;
- one byte at a time;
- split at any packet boundary or payload byte;
- adjacent to another complete frame, with no gap;
- after noise or a packet with an invalid CRC;
- while a valid nonmatching frame must be retained for its own consumer.

Reject malformed lengths before indexing fields and retain partial packets until complete or expired. Do not use read-call boundaries as message boundaries. Use the command-specific decoder only after frame integrity, endpoint, command exception and status have been considered.

These constructed packets check framing and decoding only; they do not establish physical readings, motion completion or RF interoperability.

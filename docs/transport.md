# Transport and framing

This chapter specifies the shared byte framing and the main CFS application's receive behavior. [Addressing](addressing.md) and [commands](commands.md) define the payloads. Loader phases and the external reader are separate endpoints; see [maintenance](maintenance.md) and [external reader](external-reader.md). A frame accepted by a transport is not necessarily an accepted application operation, and an acknowledgement is not proof of physical completion.

## Physical interface

The inspected printer configures its CFS port, UART5 (`/dev/ttyS5`), for **230400 baud, eight data bits, no parity, one stop bit (8N1)** on a half-duplex multidrop RS-485-labelled connection. Use one master. Neither the address scheme nor the UUID exchange is evidence of CAN or CAN tunnelling.

The native host uses nonblocking serial I/O with no software flow control and no RTS/CTS flow control. It sets RTS when opening, but does not configure an RS-485 kernel mode or toggle RTS around each frame. Direction control therefore cannot be inferred from the host's serial setup. The serial wrapper's generic default baud is 460800; that is **not** the configured CFS baud.

### Published board correlation, not verified wiring instructions

A [published board description](https://github.com/fake-name/cfs-reverse-engineering/blob/HEAD/PCBs.md) identifies a GD32F303 MCU and a 3Peak TPT75176H interface on **another board**. Its J4 numbering is reproduced only as a qualified connector reference:

| Source's J4 pin number | Published signal |
|---:|---|
| 1, 2 | +24 V |
| 3, 4 | Ground |
| 5 | RS-485 B |
| 6 | RS-485 A |
| 7 | Buffer switch 1, PD8 |
| 8 | Buffer switch 2, PD9 |

The connector's mating-side orientation, cable reversal, revision and applicability to another unit are not established here. Do not wire an adapter from this table alone. In particular, this is not a TTL UART connector, and its power lines must not be treated as logic signals or connected to an adapter's supply by assumption.

The same author reports PA2 controlling transmit-enable/receive-disable through a 2N7002, a possibly grounded driver input, two 300-ohm bias resistors and an unpopulated 120-ohm termination position. Parts of that account are explicitly tentative. The resulting interface is described as **pseudo-differential/open-collector-style**, not an independently verified standards-compliant balanced RS-485 driver. A generic RS-485 adapter, arbitrary termination or an automatic-direction adapter is not thereby proven electrically suitable. Voltage, polarity, termination, direction timing and contention behavior require unit-specific hardware verification before connection.

The published buffer board contains two open-collector optical endpoint outputs, shared directly among CFS boxes, not another addressed protocol processor. Firmware corroborates PD8 and PD9 as two active-low buffer endpoints. Do not relabel these as an extra independent filament-presence switch. See [sensors](sensors.md) for the logical fields and remaining placement uncertainty.

## Frame layout

All numeric byte values and offsets below are hexadecimal. Lengths are unsigned byte counts. Multi-byte payload byte order is command-specific; do not apply the MCU's little-endian memory order globally.

```text
0xF7 | address | L | request_mode_or_reply_status | function_code | payload | CRC8
```

| Wire offset | Size | Meaning |
|---|---:|---|
| `0x00` | 1 byte | Sync, `0xF7` |
| `0x01` | 1 byte | Request destination; reply source address |
| `0x02` | 1 byte | `L = payload byte count + 0x03` |
| `0x03` | 1 byte | Caller-selected request mode, or reply status |
| `0x04` | 1 byte | Function code |
| `0x05` onward | `L - 0x03` bytes | Payload, possibly empty |
| `L + 0x02` | 1 byte | CRC-8 |

Total wire length is `L + 0x03`, or payload length plus `0x06`. The minimum frame is six bytes, with `L = 0x03`. There is no escaping, transaction number, sequence number or explicit end marker. An embedded `0xF7` is ordinary payload inside a valid length-delimited frame.

Ordinary application callers use request mode `0xFF`. Native address-management callers explicitly use `0x00`. The transport copies this byte unchanged; it neither chooses it from the function code nor converts one value to the other. The byte is not the identity APP/LOADER field, not the `0x04` buffer-mode action and not the operation state returned by `0x0A`.

### CRC-8

| Parameter | Value |
|---|---|
| Polynomial | `0x07` (implicit leading term omitted) |
| Initial accumulator | `0x00` |
| Processing | Most-significant bit first |
| Reflect input / output | Neither |
| Final XOR | `0x00` |
| Covered bytes | `L`, request mode/status, function code, payload |
| Excluded bytes | Sync, address, transmitted CRC |

Equivalently, hash **exactly `L` bytes starting at wire offset `0x02`**. For each byte, XOR it into the accumulator, then perform eight left shifts; XOR `0x07` when the pre-shift high bit was set, retaining eight bits after each step.

The address is not CRC-protected. A valid CRC therefore does not authenticate the source or prove that a frame belongs to the current transaction. There is no cryptographic frame authentication in this envelope.

## Length limits are endpoint-specific

| Layer | Recovered limit | Implementer consequence |
|---|---|---|
| Byte envelope / native host parser | `0x03 <= L <= 0xFF`; payload up to `0xFC` bytes; total up to `0x102` bytes | Representational limit only |
| Native host parser storage | Persistent 4096-byte input buffer; separate 512-byte staging/message storage | Storage sizes are not extra wire capacity or a flow-control guarantee |
| Main application parser, 1.1.3 and 1.5.2 | Checks `L >= 0x03`, but no upper-length rejection before copying into fixed normalized storage | **Not permission to send `0xFC` bytes to the application** |
| Main application command handlers | Exact per-command request length in [commands](commands.md) | Send only the documented shape; the largest recognized ordinary request is the 16-byte credential write |
| Address scheduler header helper | Separate `0x03..0x64` length predicate, with no caller-provided available-buffer bound | Does not replace the main receive gate or make oversized input safe |
| Public `050_G32` loader parser | `0x03 <= L <= 0xFC`, payload at most `0xF9` bytes | Loader transfer phases impose their own smaller layouts and stateful chunk rules |

The application's normalized request stores destination, length, request mode and code at offsets `0x00..0x03`, payload at `0x04`, and CRC at a **fixed** offset `0x68`. Its receive/parser object is 112 bytes. A large accepted length can overlap this fixed storage. The main parser is not a safe generic large-packet interface. Loader geometry and phase-specific limits must not be imported into normal application commands, or vice versa.

## Two different receive parsers

### Main MCU application

The application parser is a bytewise state machine:

1. Wait for sync `0xF7`.
2. Accept only the current local address, `0xEA`, or `0xEB`.
3. Require `L >= 0x03`.
4. Consume request mode, function code, exactly `L - 0x03` data bytes, then CRC.
5. Validate CRC; divert registered `0xA0..0xA3` operations to the address scheduler, otherwise call the ordinary dispatcher.
6. Clear receive/busy/quiet parser state after processing.

It refreshes a **120 ms interbyte timeout** on received bytes. This is not a total operation timeout: a long movement handler can run far longer. A payload's embedded sync byte does not restart an otherwise accepted partial frame. An invalid destination does not become a broadcast through a later address helper. A CRC failure produces the same function code with status `0x02` and empty payload, except the quiet `0xEA` destination suppresses that error response.

The normal worker executes a handler synchronously, then resumes receive processing. It is not a concurrent command server. In particular, the long `0x10` action `0x04` has a private, fixed-size reader for its action `0x05`/`0x06` exchange; it does not service arbitrary normal commands. See [modes and workflows](modes-and-workflows.md).

### Printer-side native host

The native host parser is incremental over the persistent input buffer. It waits until at least six bytes are available, then checks sync, minimum length, complete declared frame and CRC. It does **not** validate source address, mode/status, command or payload schema at this stage.

| Input condition | Native behavior |
|---|---|
| Fewer than six bytes | Retain input and await more |
| Head-aligned `0xF7`, valid `L`, incomplete declared frame | Retain the entire candidate; do not search inside it for a later sync |
| Bad head | Search from offset `0x00` for the next `0xF7` |
| Bad `L` or CRC after recognized sync | Search from offset `0x01`, including bytes inside the rejected candidate |
| A later sync is found | Discard only the prefix and retain that sync |
| No sync is found | Discard available input and enter synchronization-needed state |
| Synchronization-needed scan finds sync at the current head | Clear that state but return without parsing; a later positive read restarts the native loop |
| Transaction timeout | No parser flush; partial bytes survive |

That last synchronization quirk means a complete buffered candidate can be delayed until another read. An incomplete bogus long candidate can also hold later valid bytes until more input arrives; the host parser has no corresponding MCU-style 120 ms discard rule. A new implementation should define bounded partial-frame handling explicitly rather than assume a transaction timeout resets framing. Never discard valid asynchronous notifications merely because they did not match a pending request.

## Correlation, echo and bus ownership

A robust replacement should serialize transactions, validate complete frames, then correlate source, expected reply function, expected payload and status. Exceptions must be explicit: assignment replies come from the **new** address, and a successful environment request `0x52` replies as `0x53`. Identity-bearing assignment replies must additionally match the requested UUID and device type.

The **native host** instead keys ordinary replies by address plus function code only. It does not use request mode/status, UUID or a sequence number in that key. For `0xA0` sent to `0xFC..0xFE`, its transmit key omits the address; all received `0xA0` keys omit it. Thus any CRC-valid `0xA0` may satisfy that transport wait, and the address manager must perform the subsequent UUID/new-address checks. A unicast or `0xFF` assignment retains a transmit address prefix that the native receive rule cannot match. There is no native `0x52`-to-`0x53` remapping.

The native wrapper drops a CRC-valid frame without an active notification key **before** normal parsing and registered/default callbacks. Its TX and RX native slots are single overwriteable pointers, not lossless FIFOs. Separately valid replies can replace each other before consumption. These are implementation limitations, not rules requiring a new host to discard events.

The native transport performs no echo suppression and no collision arbitration. A CRC-valid adapter echo can satisfy a pending key; corrupt collisions follow ordinary CRC/resynchronization handling. Do not infer echo merely from equality: some request-mode-`0x00` replies can legitimately equal their requests. If an adapter guarantees one local echo per transmitted frame, track and remove that echo explicitly, once, in order. Without that guarantee, equality-based deletion can destroy real replies. Establish the adapter's behavior separately from protocol semantics.

## Timeouts and retries

There is no universal command timeout. Address discovery intentionally delays replies; movement and factory operations can block for extended periods. An accepted frame, transport write, initial ACK, operation-state change and final mechanical completion are distinct milestones.

Native host policy is not a recommendation to reproduce its retries:

- With retry enable true and a requested timeout below 3600 seconds, native transport permits five total attempts. Otherwise it permits one.
- A requested timeout of at least 3600 seconds means a **one-second attempt**, not a 3600-second wait.
- Each attempt deadline starts after enqueue, not after proven physical transmission; the public wait also includes time queued behind other requests.
- Any non-null reply ends transport retries, including nonzero status. The application wrapper has a separate retry loop for its `CRC_ERR`/`LENGTH_ERR` labels.
- There is no retry backoff, sequence number or parser reset. A late same-address/same-code reply can be mistaken for a later transaction.
- Native TX issues one nonblocking write without a short-write retry or transmit-drain guarantee. A write statistic is not a device ACK.

**Do not apply generic CRC, timeout or length-error retries to motor-moving, resetting, provisioning or flash-writing operations.** After an ambiguous timeout, retain and classify late replies and reconcile operation/identity state before deciding what to do. A repeated non-idempotent command can start another physical or persistent operation.

The MCU heartbeat worker waits 60000 ticks (60 seconds in the mapped 1.5.2 1 kHz configuration), but command-busy defers timeout processing. On timeout it clears online and, outside operation state `0x03`/`0x05`, updates indicators; it does not implement a reliable all-motor stop. Heartbeat loss is **not** an emergency-stop mechanism.


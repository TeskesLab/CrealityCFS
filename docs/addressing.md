# Addressing and device identity

Discover and identify a device before issuing application commands. Address management uses the [shared frame envelope](transport.md), but is dispatched separately from the normal application's [command table](commands.md). `0x01 CREATE_CONNECT` is an application indicator/status refresh; it does not discover a device, assign an address or establish UUID ownership.

All numeric wire values and offsets in this chapter are hexadecimal. Byte counts are stated separately.

## Address spaces

| Address | Role | Scope and qualification |
|---|---|---|
| `0x01..0x04` | Assigned material-box addresses | Enabled group in the inspected printer host |
| `0xFE` | Material-box group destination / reset-default unassigned address | The main application accepts it only while its current local address is `0xFE` |
| `0x81..0x84`, group `0xFD` | Closed-loop motor definitions | Present but disabled in the inspected address-manager group table; not CFS loader-address aliases |
| `0x91..0x92`, group `0xFC` | Belt-motor definitions | Present but disabled in that table |
| `0xEA` | Main-application quiet destination | Accepted independently of local address; not normal unicast |
| `0xEB` | Main-application broadcast/fixture destination | Accepted independently of local address; replies usually use the current local source address |
| `0xFF` | Host's all-device destination; accepted by some other firmware endpoints | **Not** accepted by the main application's normal parser |
| `0x41` | External material rack in host configuration | A separate logical device; not one of the four box addresses |
| `0x11` | Separate external RFID application | Fixed endpoint; see [external reader](external-reader.md) |

An address range is not a firmware-mode test. A motor-group address does not imply a loader, and a material-box address does not prove an application. Use the returned identity mode.

### Main-application filtering and reply source

Both mapped main applications accept **current local address, `0xEA`, or `0xEB`** before dispatch. A downstream scheduler helper's acceptance of `0xFF`/`0xFE` does not bypass this first gate. Once assigned to `0x01`, for example, a box no longer accepts ordinary `0xFE` traffic. A host cannot assume that another group assignment packet will readdress an already-assigned application through `0xFE` without a reset or a reachable accepted destination.

The normal reply builder uses the current local address, not the request destination. Consequently, a request to `0xEB` can cause multiple boxes to answer from their individual addresses. This is not collision-free enumeration. The persistent-mode setter `0x50` is an exception: it temporarily uses `0xEB` as the outgoing source when requested through that destination.

`0xEA` is implemented using a **send-once suppression flag** in the normal reply builder, not a separate harmless-command class. Quiet commands still execute their side effects. Ordinary single-reply handlers consume the flag instead of transmitting; CRC and unsupported-code errors to `0xEA` are suppressed. Do not assume that every multi-message workflow or separately built scheduler response is globally silent merely because the request used the quiet destination. Silence is not evidence of success or of no motion.

## Keep mode and state fields separate

| Field | Location | Meaning |
|---|---|---|
| Request mode / reply status | Frame offset `0x03` | Caller-selected request byte (`0x00` for native addressing, normally `0xFF` for application requests), or operation result in a reply |
| Identity mode | Identity payload offset `0x01` | `0x00` APP, `0x01` LOADER |
| Buffer-mode action | `0x04` payload offset `0x01` | `0x00` enable print-buffer control; `0x01` disable it |
| Operation state | Normal `0x0A` payload offset `0x03` | Runtime IDLE/PRELOAD/PRINT/RELOAD/ERROR/TEST state |
| Persistent operating mode | `0x0C` read / `0x50` write | One byte in internal-flash parameter ID `0x06`, used to select factory/standalone behavior at boot |

None of these fields is interchangeable. In particular, sending the `0x04` action `0x00` can enter operation state `0x02` (PRINT), whereas `0x04` action `0x01` disables buffer control. The host's historical `PRINT=0` / `IDLE=1` names are **wire actions**, not the runtime operation enum. Persistent factory mode is not the APP/LOADER identity flag.

## Identity record

Every successful `0xA0..0xA3` reply has status `0x00` and **14 payload bytes**, so `L = 0x11` and total frame size is 20 bytes:

```text
[ device_type, identity_mode, UUID byte 0, ... UUID byte 11 ]
```

| Payload offset | Size | Value |
|---|---:|---|
| `0x00` | 1 byte | Type: `0x01` material box, `0x02` closed-loop motor, `0x03` belt motor |
| `0x01` | 1 byte | Mode: `0x00` application, `0x01` loader |
| `0x02..0x0D` | 12 bytes | Opaque UUID in wire order |

The main applications report type `0x01`, mode `0x00`. Their UUID is a direct byte copy of three hardware UID words beginning at MCU address `0x1FFFF7E8`; the wire bytes follow the MCU's little-endian word storage. Preserve all 12 bytes in order. Do not convert through text, reverse the whole UUID or reject a UUID simply because its first byte is `0x00`.

The eight-byte value from application `0x5B` is a different, lossy hardware identifier: one UID word followed by the AND of the other two. It is not a substitute for this UUID.

## Address command reference

Native addressing requests use frame request mode `0x00`. The following request sizes exclude sync/address/length/mode/code/CRC. Commands with empty payload still have `L = 0x03`.

| Code and native name | Request destination | Request payload | Success and failure behavior |
|---|---|---|---|
| `0xA0 SET_SLAVE_ADDR` | Material-box group `0xFE` during startup/discovery | 13 bytes: `[new_address, UUID12]`; `L = 0x10` | A matching UUID installs the new address **before** replying, so the reply source is `new_address`. UUID mismatch produces no assignment response. The main callback does not independently validate an allowed assigned-address range; host policy must restrict allocations. An empty ACK is not assignment success. |
| `0xA1 GET_SLAVE_INFO` | `0xFE` for material boxes | 2 bytes: `[inclusive_low, inclusive_high]`; `L = 0x05` | Main application replies only while its local address is `0xFE` and the inclusive range contains `0xFE`. Native host sends `[0xFE, 0xFE]`. It returns one device identity, not a master announcement or a complete table. Silence can mean no responder or a lost/colliding reply. |
| `0xA2 ONLINE_CHECK` | Assigned unicast address | Empty | Returns that endpoint's identity. Confirm expected UUID, not just source address. It is not a sensor or filament-state query. |
| `0xA3 GET_ADDR_TABLE` | Assigned unicast address | Empty | Returns the identity of **one device at that address**. Despite its name, it does not return every address or a stored table. |

The scheduler compares the assignment's full UUID. A mismatched UUID or unmet discovery range is not the ordinary dispatcher's status-`0x01` response. Addressing does not acquire the normal application's per-command exact-length/error contract merely by sharing its envelope; malformed scheduler input is not a supported probing method.

Replies are scheduled, and callbacks can repeat a response according to their configured record. The independently recovered 1.1.3 records configure one discovery reply after a pseudorandom delay in a nominal 0–1000 ms range, three assignment sends, and one identity send. The 1.5.2 callbacks likewise use scheduler-controlled delay/repetition counts. The generator is an ordinary firmware PRNG, not cryptographic randomness or a collision-free bus-arbitration protocol. Accept duplicate identical identities without treating them as new devices, and retain later frames after the first match.

### Correlating assignment correctly

The native transport omits source-address matching for group `0xA0`; it does not verify UUID. A replacement must require all of:

- Valid framing and CRC.
- Reply code `0xA0`, status `0x00`, and exactly 14 identity bytes.
- Reply source equal to the requested new address.
- Expected device type and full UUID.
- Identity mode handled explicitly rather than assumed to be APP.

The CRC excludes the address. Neither a matching code nor a nonempty reply alone confirms assignment. `0xA1` replies from several reset-default boxes can collide; one valid response does not establish that discovery is complete. Assign a discovered UUID, then continue bounded discovery with a finite policy. Silence must not be promoted to proof that every possible box is known.

## Startup, reservations and persistence

### Device behavior

The main application's assigned address is **RAM-only**. UART/address initialization sets it to `0xFE`; reset loses assignment. The silicon UUID survives independently of that RAM address. The internal-flash parameter database used by `0x0C`/`0x50` does not persist this address, and application `0x01` does not restore it.

The original printer persists address-to-UUID reservations in its own configuration (`auto_addr`, `mb_addr_table_uniids`) and restores them after startup. The serial/material caches in [storage](storage.md) are separate data stores; do not search them for an MCU address table.

### Recovered native host sequence

1. Load saved UUID reservations.
2. Perform up to two `0xA0` passes for mapped entries still in initialization state, skipping confirmed devices.
3. Perform up to two unicast `0xA3` passes over addresses not already online.
4. Start periodic discovery (`0xA1`), assignment of waiting identities (`0xA0`), online checks (`0xA2`) and a separate loader-mode check.

Allocation preference is: a matching saved UUID at an offline/initializing address, then an unmapped address, then an offline/initializing reservation with a different UUID. If an already-online saved UUID suddenly answers unassigned discovery, the native host interprets that as a possible reset, marks it waiting for assignment and clears its acknowledgement rather than immediately allocating a second slot.

The native source marks a saved row mapped only when its first UUID byte is nonzero. That is a **host sentinel quirk**, not a hardware UUID validity rule. A new persistence format should represent “unassigned” separately from a 12-byte identity and reject duplicate UUID reservations explicitly. Keep absent-device reservations distinct from confirmed live devices; only an actual matching identity establishes current ownership.

The host writes changed UUID mappings through its configuration helper and `CXSAVE_CONFIG`. The bus command itself does not persist a reservation or make host writes atomic. A replacement must define when reservations are committed and how absent devices retain or relinquish them; do not silently replace a saved identity merely because a different box happened to answer first.

### Native timeout and online policy

| Operation | Native per-request timeout | Native transport retries |
|---|---:|---|
| `0xA0` assignment | 50 ms | Disabled |
| `0xA1` discovery | 1 s | Disabled |
| `0xA2` online check | 100 ms | Disabled |
| `0xA3` identity probe | 50 ms | Disabled |

These are recovered host choices, not guaranteed electrical response deadlines. Startup's two-pass policy is above the transport. Periodic online checking marks a mapped entry offline after its **fourth failed check** (`lost_count > 3`); a successful matching identity resets the counter.

The active combined periodic worker pauses nominally one second between passes, or ten seconds while the printer's state string is `printing` or `pause`. When any box heartbeat-enable flag is false, it defers the combined address work for ten seconds. Those are printer policy and scheduling intervals, not a mandatory protocol heartbeat. Application `0x0A` polling and the MCU's own heartbeat worker are separate mechanisms; see [integration](integration.md).

## Application versus loader

The public `050_G32` loader accepts local, `0xFE`, `0xFF` and its factory destination, unlike the main application. Its discovery can include an **assigned** device when that address falls within the requested range. Do not copy that behavior into the application profile.

The native address manager may send destination `0xFF`, request mode `0x00`, code `0x0B`, payload `[0x01]` after learning a LOADER identity. In the public loader this requests application start without a reply. In the main application, code `0x0B` instead takes a two-byte indicator request. Dispatch must be endpoint-specific: the same function byte does not imply the same operation. Loader start, reset, firmware verification and update flows belong in [maintenance](maintenance.md), not automatic discovery in a new integration.

The supplied G30 and G32 application packages are byte-identical for each mapped version. That does not prove identical hardware, bootloaders or which application was running on a particular unit. Read [compatibility](compatibility.md) before selecting a firmware profile.


# Creality CFS protocol reference

An unofficial, documentation-only guide for implementing an independent
integration with the Creality Color Filament System (CFS).

The inspected main applications use **half-duplex RS-485 at 230400 baud, 8N1**,
custom binary frames and CRC-8. This reference covers firmware **1.1.3 and 1.5.2**,
plus the separate external RFID reader and the recovered maintenance interfaces.
It does not contain a driver, firmware, keys or a drop-in printer controller.

## Start here

1. [System architecture and responsibilities](docs/architecture.md)
2. [Electrical link, framing and CRC](docs/transport.md)
3. [Addressing and initialization](docs/addressing.md)
4. [Integration guide](docs/integration.md)
5. [Checked wire examples](docs/wire-examples.md)

## Reference

- [Every application command](docs/commands.md) and [modes/workflows](docs/modes-and-workflows.md)
- [Sensors and status](docs/sensors.md)
- [RFID records, material profiles and authentication](docs/rfid-and-material.md)
- [Volatile and persistent storage](docs/storage.md)
- [External RFID reader](docs/external-reader.md)
- [Reset, firmware update and factory operations](docs/maintenance.md)
- [Version compatibility and support boundaries](docs/compatibility.md)

**Use one bus master.** Feed, buffer, RFID-acquisition, reset and factory commands
can move hardware or change persistent state. An acknowledgement is not always
completion, and heartbeat loss is not an emergency stop. Read the integration
and maintenance chapters before implementing those paths.

The documentation is based on static recovery and offline checks, not live
interoperability or unit-specific wiring approval. Known firmware defects and
unverified hardware assumptions are stated where they affect an implementation.
Chip-internal firmware reconstruction is not required for the documented
communication interface.

Documentation license: [Apache-2.0](LICENSE). Third-party firmware and external
research retain their respective ownership and are not redistributed here.

# Livt.FileSystems

`Livt.FileSystems` provides bounded, read-only filesystem access for Livt FPGA
applications. FAT32 files stream through a caller-driven byte interface, so a file
does not have to fit in FPGA memory. Storage is supplied through
`Livt.IO.IReadOnlyBlockDevice`; USB, SPI-backed adapters and simulated media can use
the same filesystem reader.

See [CHANGELOG.md](CHANGELOG.md) for release notes.

Version 0.0.1 provides:

- `Fat32Reader`: root-directory listing and short-name file streaming.
- `PartitionReader`: bounded GPT, primary MBR and unpartitioned-volume discovery.
- `Fat32Geometry`: checked FAT32 geometry and address calculations.
- `SectorBuffer`: configurable sector storage, using BRAM by default.
- `ShortFileName`: bounded ASCII 8.3 name validation and normalization.
- Retained results, consumer backpressure, cancellation and explicit failure status.

## 📦 Package

```toml
[dependencies]
"Livt.FileSystems" = "0.0.1"
```

The package depends on published `Livt.IO 1.7.0` and `Livt.Utils 1.1.0`.
Run `livt sync` to resolve them. Applications using `IReadOnlyBlockDevice` or
storage-provider types directly should also declare `Livt.IO`. The package has
no USB, UART, CLI or board dependency.

## 📚 Components

All public types belong to `Livt.FileSystems`.

| Component or type | Responsibility |
|---|---|
| `Fat32Reader` | Owns partition discovery, FAT32 traversal and retained file/listing state |
| `PartitionReader` | Selects a bounded volume through a borrowed sector buffer |
| `Fat32Geometry` | Parses the boot sector and calculates checked cluster addresses |
| `ISectorBuffer`, `SectorBuffer` | Copy and retain one 512-byte sector with configurable storage |
| `ShortFileName` | Normalize and validate supported ASCII 8.3 names |
| `FileResult` | Distinguish active work, successful completion and terminal failures |

## 🔌 API overview

Construct `Fat32Reader` with an exclusively borrowed block device. For listing,
call `TryStart(false)`, then call `Service()` regularly. When `entryReady` is true,
read its padded 8+3 name, size and attributes, then call `ConsumeEntry()`.

For file access, set all eleven padded uppercase alias bytes using `SetNameByte`,
then call `TryStart(true)`. Call `Service()` and consume available bytes with
`TryReadByte(value)`. The byte stream pauses when the caller stops consuming.
A terminal `result` remains available until the next accepted operation.
`ShortFileName` can prepare the alias: call `Reset`, append the input characters,
check `IsValid`, then copy its eleven `GetByte` values to the reader.

Only `FileResult.Success` makes a load complete. A consumer that receives bytes
before an error or cancellation must treat those bytes as an incomplete load.
Call `Cancel()` and continue `Service()` until the operation leaves `Active`;
then block-device ownership is released. Clock/reset comes from the component's
context. Functions are scheduled calls, with no fixed-cycle latency promise.

The application owns the byte consumer and destination. It may process bytes
immediately or write them into memory; the filesystem reader itself retains only
one sector. See [reader contracts](docs/fat32.md) for scheduling, media changes
and completion rules.

## ⚙️ Storage configuration

The default sector store is `SequentialRam<byte, 512, MemoryStyle.Block>`.
Choose a different provider through the `STORAGE` generic, for example:

```livt
using Livt.IO
using Livt.FileSystems

// Fields in an owning component.
reader: Fat32Reader
arrayReader: Fat32Reader<ArrayRam<byte, 512>>
```

In the owner's constructor, pass the exclusively borrowed block device to the
selected reader type:

```livt
reader = new Fat32Reader(device)
// Alternative for a design selecting array storage:
arrayReader = new Fat32Reader<ArrayRam<byte, 512>>(device)
```

These are alternative configurations. Readers sharing a device must take turns;
only one may own an active operation. `STORAGE` must provide at least 512
addressable bytes through serialized `IRam<byte>` calls. Compatible providers
may trade storage resources for additional clock cycles. Block style is a
synthesis hint; actual memory mapping depends on the target and toolchain.

Generic limits bound file size (16 MiB by default), directory traversal (256
clusters) and GPT scanning (128 entries). They do not allocate file-sized or
partition-table-sized buffers. See [storage and limits](docs/storage.md) for
configuration examples, ownership and reset behavior.

## 🧪 Build and Test

```sh
livt sync
livt validate
livt test --release -W all,error
livt build --release -W all,error
```

The [tests](tests/) use a simulated block device independently of USB. They cover
partition layouts, fragmented and empty files, retained listings, backpressure,
malformed metadata, cancellation, disconnects, filename handling and alternative
sector storage.

## 🚧 Supported profile

- 512-byte logical sectors, FAT32 clusters from 512 bytes through 32 KiB.
- FAT32 superfloppy, first primary MBR partition of type 0x0B/0x0C, or first GPT
  Basic Data/EFI partition. GPT addresses must fit the 32-bit block interface.
- GPT revision 1.0, 92–512-byte primary header, 128-byte entries, bounded table,
  and verified header/table CRC32. Corrupt primary GPT fails; no backup recovery.
- Root-directory entries and file access by short alias; LFN entries and volume
  labels are skipped. Subdirectories can be listed but are not traversed.
- Fragmented files, empty files with cluster zero, and short final sectors.
- Active FAT selection when mirroring is disabled; no comparison or repair of
  mirrored FAT copies. No filesystem writes, allocation, journal or repair.

See [reader contracts and limits](docs/fat32.md) for lifecycle and integrity rules.

## 📄 License

This project is licensed under the MIT License. See [LICENSE](LICENSE).

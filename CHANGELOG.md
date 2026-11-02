# Changelog

All notable changes to this package are documented here.

## 0.0.1

### Added

- Transport-independent, read-only FAT32 root-directory listing and file streaming
  through `Livt.IO.IReadOnlyBlockDevice`.
- `PartitionReader` for unpartitioned volumes, primary MBR FAT32 partitions and
  GPT Basic Data/EFI partitions, with primary-header and entry-array CRC32 checks.
- `Fat32Geometry` for bounded boot-sector parsing, active-FAT selection and
  cluster-address calculations.
- `ISectorBuffer` and `SectorBuffer` for one retained 512-byte sector, with
  configurable `IRam<byte>` storage and `SequentialRam` block-memory defaults.
- `Fat32Reader` with configurable file-size, directory-traversal and GPT-entry
  limits; fragmented-file chain validation before streaming; retained directory
  entries; byte-level backpressure; cancellation and explicit terminal results.
- `ShortFileName` for ASCII 8.3 filename normalization without truncation.
- Tests for partition discovery, file/listing behavior, corrupt media, disconnects,
  cancellation, filename bounds and both block-memory and array storage providers.
- Public documentation for component ownership, scheduled calls, completion,
  reset, storage substitution and supported format limits.

### Scope

- 512-byte logical sectors and power-of-two FAT32 clusters up to 32 KiB.
- Root-directory access by short alias; subdirectories can be listed but are not
  traversed. Long-name records and volume labels are skipped.
- GPT revision 1.0 with 128-byte entries and addresses within the 32-bit block API.
- No filesystem writes, allocation, repair, backup-GPT recovery or whole-file
  destination memory. Applications own file consumers and memory writers.

### Dependencies

- Published `Livt.IO 1.7.0` for memory providers and `IReadOnlyBlockDevice`.
- Published `Livt.Utils 1.1.0` for IEEE CRC32 calculation.
- No USB, UART, CLI, board-specific clock or platform dependency.

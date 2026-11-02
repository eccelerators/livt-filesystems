# FAT32 reader contracts

`Livt.FileSystems 0.0.1` provides scheduled, read-only FAT32 access independently
of the block transport. See [storage and resource limits](storage.md) for provider
selection and generic bounds.

## Ownership and scheduling

`Fat32Reader` owns one `SectorBuffer`, `PartitionReader` and `Fat32Geometry`, and
borrows the device's only scheduled client. The caller serializes every method,
including `Service`, byte access and cancellation. The device itself may have an
autonomous transport worker. Other block consumers must not access that device
until the reader has reached a terminal result.

`TryStart` returns acceptance, not successful mounting or opening. Every operation
remounts; no cached geometry survives between jobs. Media must remain unchanged
while an operation is active. A disappearing device invalidates buffered data.
A provider must expose loss of readiness or a failed read on media replacement;
the block contract does not carry a media-generation identifier.

Each `Service` call advances a bounded step or copies at most one sector byte.
Parsing a geometry or directory entry uses multiple scheduled RAM reads. Provider
latency is preserved; this API makes no one-cycle or throughput guarantee. There
is no wall-clock timeout above the block provider, which is responsible for
bounded transport completion or cancellation. Consumers may pause indefinitely;
call `Cancel` to end an operation deliberately.

Reset discards the job and its metadata. Payload RAM need not be cleared. The
borrowed provider must share the reset context so ownership is reset coherently.

## Streaming and completion

`TryReadByte` transfers one byte only on true and yields zero otherwise. It never
reads beyond the directory entry's file size. The caller observes `fileSize` after
opening and `bytesRead` as bytes are delivered. Payload is available only in the
file-data phase. The final consumed byte is followed by terminal completion on a
subsequent `Service` call.

Before streaming, the reader follows the entire file chain. It requires exactly
`ceil(fileSize / clusterBytes)` clusters, with a final end-of-chain marker. Free,
reserved, bad, out-of-range, truncated, cyclic and overlong chains fail this check.
This deliberately strict profile rejects files with excess allocated clusters.
No payload is delivered before this validation completes; an I/O error can still
occur later while streaming. Only `Success` commits the whole load. A consumer
with irreversible per-byte effects must account for partial failure explicitly.

Listings retain one entry until `ConsumeEntry`. Deleted entries, volume labels
and long-name records are skipped. Names are returned as their eleven raw short
alias bytes; callers must sanitize non-ASCII/control bytes before terminal output.
`ShortFileName` accepts ASCII letters, digits, underscore and hyphen, with a single
optional dot. It normalizes lowercase input and rejects paths or truncation.
Other valid FAT alias characters can be supplied directly through `SetNameByte`.

## Bounds and integrity

All filesystem and partition addresses stay within the block device and selected
partition. FAT type is determined by cluster count, not the decorative FAT32 text
in the boot sector. Sector geometry, FAT capacity, active FAT number, root cluster
and data ranges are checked before following them. Directory traversal is bounded
by both the medium's cluster count and `MAX_DIRECTORY_CLUSTERS`; exceeding that
limit reports `LimitExceeded`, including a directory cycle.

GPT parsing verifies the primary header and all declared entry bytes, including
unused entries. It selects the first Basic Data or EFI System partition and then
requires a valid FAT32 boot sector. It does not search later partitions when that
selected partition contains a different filesystem. It does not repair GPT or
fall back to its backup. Bounds and CRC validation cover the metadata used by the
reader; this is not a whole-disk consistency checker.

`MAX_FILE_BYTES` defaults to 16,777,216 bytes, `MAX_DIRECTORY_CLUSTERS` to 256,
and `MAX_GPT_ENTRIES` to 128 (maximum configurable 4096). These are caller policy
limits, not allocated buffer sizes. The block interface limits addresses to 32 bits.
One or two FAT copies are supported, with the active copy selected from BPB flags.
FSInfo free-space counters are unnecessary for read-only access and are ignored.

## Results and retained data

| Result | Meaning |
|---|---|
| `Idle` | No operation has been started since reset |
| `Active` | Accepted operation needs more service or consumer progress |
| `Success` | Listing finished, or every file byte was delivered successfully |
| `NotFound` | The configured short alias was not found as a root file |
| `Unsupported` | Block size or partition profile is outside the supported scope |
| `Corrupt` | Required geometry, checksums or file-chain invariants failed |
| `IoError` | The device became unavailable or a required sector operation failed |
| `Cancelled` | Cancellation completed and outstanding device access was drained |
| `LimitExceeded` | File size or directory traversal exceeded the configured bound |

Malformed or unsupported FAT boot geometry reports `Corrupt`; unsupported
partition profiles report `Unsupported`. A GPT table exceeding `MAX_GPT_ENTRIES`
also reports `Unsupported`. Results remain stable until the next accepted
`TryStart` or reset. After an error, `bytesRead` describes partial delivery and
must not be interpreted as a successfully loaded file.

## Lower-level components

Applications normally use `Fat32Reader`, which coordinates the following pieces:

- `SectorBuffer.Start(lba)` accepts a read; call `Service` until `busy` clears,
  then require `valid` before reading. `ReadByte` accepts offsets 0–511;
  little-endian `Read16` and `Read32` require offsets 0–510 and 0–508 respectively.
  Calls are scheduled and serialized. `Cancel` invalidates contents and requires
  continued service until device ownership is released.
- `PartitionReader.Start(capacity)` starts fresh discovery over an exclusively
  borrowed sector buffer; capacity counts 512-byte blocks. Service the sector
  buffer before each parser `Service` call. Use `start` and `length` only after
  `Success`. A caller cancelling standalone discovery must cancel and drain the
  shared sector buffer before reusing it.
- `Fat32Geometry.Parse(sector, start, length)` requires a valid retained boot
  sector and a partition range already bounded within the device and 32-bit
  addresses. Use its fields and `IsCluster` only after it returns true; a failed
  parse may leave partial fields. `ClusterLba` requires a valid data cluster.

All components share the owning clock/reset context. They create no PLL, select
no fixed frequency and make no bus-speed assumption. Their scheduled arithmetic
and storage accesses can span clock cycles.

## Format references

The implementation follows the FAT32 BPB, cluster-type determination, FAT entry
and directory-entry layouts in the [Microsoft FAT specification](https://www.scs.stanford.edu/~zyedidia/docs/_other/fat.pdf),
with the explicit profile restrictions above. GPT fields and CRC coverage follow
[UEFI 2.10, chapter 5](https://uefi.org/specs/UEFI/2.10/05_GUID_Partition_Table_Format.html).

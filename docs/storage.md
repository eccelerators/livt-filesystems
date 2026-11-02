# Storage and resource limits

## Sector provider

`Fat32Reader` forwards its `STORAGE` type to its owned `SectorBuffer`:

```livt
component Fat32Reader<STORAGE: IRam<byte> & new() = SequentialRam<byte, 512, MemoryStyle.Block>,
	MAX_FILE_BYTES: uint = 16777216, MAX_DIRECTORY_CLUSTERS: uint = 256,
	MAX_GPT_ENTRIES: int = 128>
```

The provider must implement `Livt.IO.IRam<byte>`, have a parameterless constructor
and support addresses 0 through 511. It inherits the reader's clock/reset context.
The sector buffer owns it exclusively and serializes reads and writes. Provider
calls may take multiple cycles; there is no fixed one-edge RAM requirement.
An undersized store cannot accept sector reads and will fail the reader operation.

| Provider | Storage intent |
|---|---|
| `SequentialRam<byte, 512, MemoryStyle.Block>` | Default block-style RAM with serialized scheduled access |
| `SequentialRam<byte, 512, MemoryStyle.Distributed>` | Scheduled RAM with distributed-memory intent |
| `ArrayRam<byte, 512>` | Explicit logic-array implementation, typically mapped to registers |

For example, `Fat32Reader<ArrayRam<byte, 512>, 1048576, 64, 128>` selects an array
sector store, a 1 MiB file limit, at most 64 directory clusters and at most 128 GPT
entries. Construct exactly that type with the borrowed block device. The same
storage choice is available on standalone `SectorBuffer<STORAGE>` instances.

Memory style is a synthesis hint. Resource use, implementation timing and total
scheduled-call latency depend on the backend, compiler and target. Changing the
provider preserves acceptance, completion, ordering and data-validity rules.

## Buffer ownership and reset

There is one 512-byte filesystem sector buffer. The block-device implementation
may have its own transport buffer; this package does not control that storage.
Small filename, geometry, cursor and result fields remain registers. GPT entries
are scanned through the sector buffer instead of retaining the whole table.

Accepted sector reads invalidate old contents. Bytes become visible only after
all 512 bytes have been copied successfully. Cancellation, failed reads and
observed media loss invalidate the buffer. Reset clears metadata and ownership;
it does not require payload memory to be reset. The borrowed block device and
filesystem reader must reset coherently.

A larger file limit does not allocate destination memory. Applications retain or
process bytes after `TryReadByte` succeeds and decide whether partial output can
be used after failure. Stop consuming to apply backpressure; continue `Service`
to progress transport work or drain cancellation. Only `FileResult.Success`
marks a complete file.

## Compile-time policy bounds

| Generic | Default | Supported values and effect |
|---|---:|---|
| `MAX_FILE_BYTES` | 16,777,216 | Positive `uint`; larger files return `LimitExceeded` before streaming |
| `MAX_DIRECTORY_CLUSTERS` | 256 | Positive `uint`; bounds root traversal, also limited by medium geometry |
| `MAX_GPT_ENTRIES` | 128 | 1–4096; larger declared tables return `Unsupported` |

These bounds limit accepted work and traversal time. They do not reserve an
array of that size. A cyclic directory eventually reaches the traversal limit.
A file chain is checked against the exact cluster count implied by its file size
before payload delivery. All addresses remain bounded by the selected partition
and block-device capacity.

See [FAT32 contracts](fat32.md) for lifecycle, integrity checks and terminal results.

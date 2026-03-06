# MEP: Local Format — Pluggable On-Node Data Formats for Sealed Segments

- **Created:** 2026-03-05
- **Author(s):** @zhicheng
- **Status:** Draft
- **Component:** QueryNode | DataNode | Storage
- **Related Issues:** TBD
- **Released:** TBD

## Summary

Introduce a **local_format** field-level property that controls how sealed segment data is stored and accessed on query nodes. The default format is the existing Milvus row-oriented layout (RowChunk); the first alternative is **Vortex**, a compressed columnar format that reduces memory and disk footprint while maintaining query performance through on-demand decompression.

## Motivation

Milvus sealed segments currently store field data in an uncompressed, row-oriented binary layout (RowChunk). This works well for random access but has two drawbacks:

1. **Memory pressure**: All field data must be fully decompressed in memory (or mmap'd at full size). For large scalar fields (VARCHAR, JSON), this can consume significant RAM.
2. **No format flexibility**: The on-node data format is hard-coded. There is no way to choose a format that trades CPU (decompression) for memory/disk savings.

The **local_format** feature addresses this by:

- Allowing users to specify `local_format=vortex` per field at collection creation time.
- Keeping compressed data on disk (via mmap) and decompressing only the accessed portion on demand.
- Providing a unified data access interface (`ChunkDataView`) so upper layers are format-agnostic.

## Public Interfaces

### Collection Creation

A new type parameter `local_format` is added to field schemas:

```python
schema.add_field(
    field_name="description",
    datatype=DataType.VARCHAR,
    max_length=65535,
    local_format="vortex"    # new parameter
)
```

Valid values: `"row"` (default), `"vortex"`.

Applies to **non-primary-key scalar fields only**. This includes all non-vector, non-PK types: INT8/16/32/64, FLOAT, DOUBLE, BOOL, VARCHAR, JSON, ARRAY, BSON, etc. Vector fields and primary key fields ignore this parameter.

### Storage Configuration

A global storage format config controls the file format written to object storage:

```yaml
common:
  storage:
    format: vortex  # options: parquet, vortex
```

When `format=vortex`, the write path (DataNode flush / compaction) produces Vortex-encoded files for column groups that contain `local_format=vortex` fields. Other column groups continue to use Parquet.

### Relationship between `storage.format` and `local_format`

These two configs are independent and serve different purposes:

- **`storage.format`** controls the **write path** — what file format is used when flushing/compacting new segments to object storage (S3). Changing this config only affects newly written files; existing files on S3 are not affected.
- **`local_format`** controls the **read path** — how query nodes load and access field data from sealed segments.

`local_format=vortex` requires Storage V3 and the actual file on S3 to be in vortex format. If a segment's files are still in Parquet format (e.g., written before `storage.format` was changed to vortex), the segment **gracefully falls back to Row format** loading — no error is raised. This means `local_format` is a "preferred" format, not a hard requirement. Only when the on-disk file is actually vortex will the vortex loading path be used.

## Design Details

### 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Query / Expression                    │
│                 chunk_view<T>() unified API              │
├─────────────────────────────────────────────────────────┤
│               AnyDataView (type erasure)                │
├──────────────────────┬──────────────────────────────────┤
│   ContiguousDataView │       VortexDataView             │
│   (RowChunk path)    │       (VortexChunk path)         │
│   data in memory or  │       lazy decompression         │
│   mmap'd flat buffer │       via Reader/ChunkReader     │
├──────────────────────┴──────────────────────────────────┤
│                    Chunk Layer                           │
│   RowChunk (ScalarChunk, StringChunk, ...)              │
│   VortexChunk (lightweight, no decompressed data)       │
├─────────────────────────────────────────────────────────┤
│               GroupChunkTranslator (Parquet)             │
│               VortexGroupChunkTranslator (Vortex)       │
├─────────────────────────────────────────────────────────┤
│           Object Storage (S3 / MinIO / Local)           │
│           Parquet files  |  Vortex files                │
└─────────────────────────────────────────────────────────┘
```

### 2. Column Group Splitting

Fields with `local_format=vortex` are split into a separate column group from default (row) format fields. This is handled by `LocalFormatPolicy` in the column group splitting pipeline:

```
DefaultPolicies execution order:
1. SystemColumnPolicy  — split system fields (RowID, Timestamp) + PK
2. AvgSizePolicy       — split large fields by avg size
3. SelectedDataTypePolicy — split vector / text fields (each gets own group)
4. LocalFormatPolicy   — split vortex fields into separate group
5. RemanentShortPolicy — merge remaining short fields
```

This ensures vortex fields and non-vortex fields are never mixed in the same column group file.

### 3. Write Path

When `storage.format=vortex`, the packed writer (via Rust FFI to milvus-storage) encodes column groups in Vortex format. The Vortex file is a single file per column group, containing all fields in that group.

The binlog path structure is unchanged:
```
insert_log/{collID}/{partID}/{segID}/{groupID}/{logID}
```

Where `groupID` is the column group ID (assigned by split policies), and `logID` is the allocated log ID.

### 4. Read Path — VortexGroupChunkTranslator

On query node load, the segment loading path checks per-field `local_format`:

- If all fields in a column group have `local_format=vortex` AND the file format is vortex → use `VortexGroupChunkTranslator`
- Otherwise → use standard `GroupChunkTranslator` (Parquet path)

**Key design: BufferFileSystem as data bridge**

The milvus-storage `Reader` API reads data through an `arrow::fs::FileSystem` abstraction. It does not accept raw memory buffers directly. To enable decompression from in-memory (or mmap'd) data, we implement a custom `BufferFileSystem` (a subclass of `arrow::fs::FileSystem`) that serves compressed vortex data from memory buffers as if they were files.

The flow is:

```
S3 download → Arrow Buffer (in-memory)
    → BufferFileSystem::Register("mem://path", buffer)
    → register BufferFileSystem in milvus-storage FilesystemCache (key="mem")
    → Reader resolves "mem://path" via FilesystemCache → BufferFileSystem
    → BufferFileSystem::OpenInputFile() → BufferReader (zero-copy)
    → Reader/ChunkReader decompress from buffer on demand
```

This same pattern extends to mmap: instead of downloading into an Arrow Buffer, the vortex file is written to local disk and mmap'd. The mmap pointer is wrapped as `arrow::Buffer::Wrap(mmap_ptr, size)` and registered in BufferFileSystem identically. The Reader operates on the mmap'd data transparently — the only difference is where the underlying bytes reside (heap vs mmap'd pages).

`BufferFileSystem` is a read-only singleton that implements:
- `Register(path, buffer)` / `Unregister(path)` — manage path-to-buffer mappings
- `OpenInputFile(path)` → `arrow::io::BufferReader` (zero-copy read from buffer)
- `GetFileInfo(path)` → file size from buffer
- `type_name()` → `"mem"` (used as FilesystemCache key)

**VortexGroupChunkTranslator construction:**

1. Download vortex files from object storage into Arrow Buffers (in-memory)
2. Register buffers in `BufferFileSystem` singleton with `mem://` paths
3. Register `BufferFileSystem` in milvus-storage `FilesystemCache` with key `"mem"` (idempotent)
4. Rewrite column group file paths from `s3://...` to `mem://...`
5. Create milvus-storage `Reader` from rewritten column groups — Reader resolves `mem://` paths through `FilesystemCache` → `BufferFileSystem`
6. Extract row group metadata (sizes, row counts) via `ChunkReader`
7. Merge row groups into cache cells (4 row groups per cell, same as Parquet)

**VortexGroupChunkTranslator::get_cells():**

For each requested cell:
1. Create a shared `ChunkReader` for the cell
2. For each field in the column group, create a `VortexChunk` holding:
   - Shared `Reader` (for point queries via `take()`)
   - Shared `ChunkReader` (for bulk decompression via `get_chunks()`)
   - Row group indices for this cell
   - Column index in batch, row start offset

### 5. VortexChunk — Lightweight Chunk

`VortexChunk` extends `Chunk` but holds **no decompressed data**. It is a lightweight handle:

```cpp
class VortexChunk : public Chunk {
    // These throw NotImplemented — callers must use GetAnyDataView()
    const char* ValueAt(int64_t idx) const override;
    const char* Data() const override;

    // Returns lazy VortexDataView — decompression on demand
    AnyDataView GetAnyDataView() const override;
    AnyDataView GetAnyDataView(int64_t offset, int64_t length) const override;

private:
    std::shared_ptr<Reader> reader_;           // for point queries
    std::shared_ptr<ChunkReader> chunk_reader_; // for bulk decompression
    std::vector<int64_t> chunk_indices_;        // row group indices
    int column_in_batch_;                       // column position
    int64_t row_start_;                         // global row offset
};
```

### 6. VortexDataView — Lazy Decompression

Each `GetAnyDataView()` call returns a type-specific `VortexDataView` that decompresses on demand:

| DataView Class | Data Types | operator[](idx) | Data() (bulk) |
|---|---|---|---|
| `VortexNumericDataView<T>` | int8–64, float, double | `Reader::take({row_start+idx})` | `ChunkReader::get_chunks()` → memcpy |
| `VortexBoolDataView` | bool | `take()` → BooleanArray | `get_chunks()` → bit unpack |
| `VortexStringDataView` | string_view (VARCHAR, BSON) | `take()` → BinaryArray | `get_chunks()` → string_view array |
| `VortexJsonDataView` | Json | `take()` → BinaryArray → Json | `get_chunks()` → Json array |
| `VortexArrayDataView` | ArrayView (ARRAY) | `take()` → ListArray → ArrayView | `get_chunks()` → ArrayView array |

All types share `VortexDataViewCore` which encapsulates:
- Point query: `Reader::take()` for single-row access
- Bulk query: `ChunkReader::get_chunks()` for full decompression
- `data_offset` field for range sub-views (`GetAnyDataView(offset, length)`)

**Decompressed data lifetime**: Data is held in the `VortexDataView` instance (mutable lazy fields). When the `AnyDataView` is destroyed, decompressed data is freed. This keeps memory usage bounded to the currently active query's working set.

**Thread safety**:
- `Reader` and `ChunkReader` (from milvus-storage) are **thread-safe** (Rust `Send + Sync`). Multiple VortexChunks safely share the same instances.
- `VortexDataView` instances are **not thread-safe**. Internal mutable lazy caches have no locking. This matches the usage pattern: each query thread holds its own `PinWrapper<DataView>` instance, so concurrent access to a single DataView does not occur.

### 7. Unified Data Access — ChunkDataView

The `ChunkDataView<T>` interface provides format-agnostic data access:

```cpp
template <typename T>
class ChunkDataView : public BaseDataView {
    virtual const T& operator[](int64_t idx) const = 0;  // point access
    virtual const T* Data() const = 0;                     // bulk access
    int64_t RowCount() const;
    const bool* ValidData() const;
};
```

Two concrete implementations:
- `ContiguousDataView<T>`: For RowChunk — wraps raw pointer to in-memory/mmap'd data
- `VortexDataView` variants: For VortexChunk — lazy decompression

Upper layers (expression evaluation, search, retrieve) call `chunk_view<T>()` which returns a `PinWrapper<shared_ptr<ChunkDataView<T>>>`. They never know which format underlies the data.

### 8. Mmap Support (WIP)

> This section is work-in-progress. Details will be finalized in a follow-up iteration.

**Current state**: Vortex files are downloaded into Arrow Buffers in RAM.

**Planned design**: Mmap the compressed vortex file to local disk.

```
S3 download → write to local file → mmap(PROT_READ, MAP_SHARED)
    → arrow::Buffer::Wrap(mmap_ptr, size)  // zero-copy
    → register in BufferFileSystem
    → Reader works normally over mmap'd data
```

Key differences from Parquet mmap:

| | Parquet mmap | Vortex mmap |
|---|---|---|
| What's on disk | Decompressed flat data | Compressed vortex file |
| Disk usage | = raw data size | << raw data size |
| Access pattern | Direct pointer read | Decompress on access |
| CPU overhead | None | Decompression per access |
| Page-in granularity | OS 4KB pages | OS 4KB pages |

The OS transparently manages page-in/page-out at 4KB granularity. Compressed data that hasn't been accessed recently is evicted from physical RAM by the OS without application involvement. Decompressed data is temporary (lives only in VortexDataView lifetime) and freed after use.

Memory footprint ≈ size of one decompressed row group batch, regardless of total segment size.

## Compatibility, Deprecation, and Migration Plan

- **Backward compatible**: Fields without `local_format` default to `"row"` (existing behavior).
- **Mixed format**: A collection can have some fields with `local_format=vortex` and others with default. They are stored in separate column groups.
- **Upgrade path**: Existing segments remain in Parquet/row format. New segments written with `storage.format=vortex` use the vortex format. Compaction can convert old segments to new format.
- **Downgrade**: If `local_format=vortex` fields are loaded by a Milvus version that doesn't support vortex, loading will fail with a clear error message.

## Test Plan

- **Unit tests**: ChunkDataView, VortexDataView, VortexChunk GetAnyDataView for all scalar types
- **Integration tests**: Insert → flush → load → query/search/retrieve with vortex VARCHAR, INT, JSON fields
- **Mmap tests**: Verify vortex mmap path produces correct results with memory-constrained scenarios
- **Compatibility tests**: Mixed format collections (some fields vortex, some row)
- **Performance benchmarks**: Compare query latency and memory usage between row and vortex formats

## References

- [Apache Vortex](https://github.com/spiraldb/vortex) — Compressed columnar format
- [milvus-storage](https://github.com/milvus-io/milvus-storage) — Milvus storage layer with Vortex support
- Milvus ChunkDataView design: `internal/core/src/common/ChunkDataView.h`
- Milvus CacheSlot architecture: `cachinglayer/CacheSlot.h`

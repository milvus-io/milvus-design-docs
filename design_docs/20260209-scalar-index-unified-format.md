# Scalar Index V3 Format

## 1. Goals

| Goal | Description |
|------|-------------|
| Single file | All entries packed into one remote object |
| Concurrent upload | Multipart Upload in parallel |
| Concurrent encrypt/decrypt | Slice-level parallelism, same granularity as V2 |
| Concurrent download | `ReadAt` multi-threaded parallel |
| Low memory peak | `O(W × slice_size)`, does not grow with entry size |
| Encryption transparent | Index classes unaware of encryption |
| Reduce small IO | O(1) meta packed into a single entry |
| Compatible with V2 | Go control plane routes by `SCALAR_INDEX_ENGINE_VERSION`; data lake by filename format |

---

## 2. File Format

### 2.1 Layout

```
┌─────────────────────────────────────────┐
│ Magic Number (8 bytes): "MVSIDXV3"       │
├──────────── Data Region ────────────────┤
│ Entry 0 data                             │
│ Entry 1 data                             │
│ ...                                      │
│ Entry N data                             │
├──────────── Directory Table (JSON) ─────┤
│ { "version": 3, "entries": [...] }       │
├──────────── Footer (8 bytes) ───────────┤
│ directory_table_size (uint64)            │
└─────────────────────────────────────────┘
```

Directory Table is at file tail. Data is written first, offsets recorded, then Directory Table and Footer appended.

### 2.2 Slice — Encryption Only

Slice is the **encryption boundary**, not a general format concept:

- **Encrypted**: Each entry split into fixed-size slices (default 16MB). Each slice independently encrypted via `IEncryptor::Encrypt()`. Slice boundaries recorded in Directory Table.
- **Unencrypted**: Entry is contiguous plaintext. Directory Table records `offset` + `size`. EntryReader self-splits by range for concurrent `ReadAt`.

### 2.3 Directory Table

**Unencrypted**:

```json
{
  "version": 3,
  "entries": [
    {"name": "SORT_INDEX_META", "offset": 0, "size": 48},
    {"name": "index_data", "offset": 48, "size": 100000000}
  ]
}
```

**Encrypted**:

```json
{
  "version": 3,
  "slice_size": 16777216,
  "entries": [
    {
      "name": "SORT_INDEX_META",
      "original_size": 48,
      "slices": [
        {"offset": 0, "size": 76}
      ]
    },
    {
      "name": "index_data",
      "original_size": 100000000,
      "slices": [
        {"offset": 76, "size": 16777244},
        {"offset": 16777320, "size": 16777244}
      ]
    }
  ],
  "__edek__": "base64_encoded_encrypted_dek",
  "__ez_id__": "12345"
}
```

Field semantics:
- Unencrypted: `offset` + `size` are position within Data Region (relative to Magic end).
- Encrypted: all entries use `original_size` + `slices[]` format uniformly.
  - `original_size`: plaintext size, for pre-allocating buffer.
  - `slices[]`: each slice's position and ciphertext size (including IV/Tag overhead) in Data Region.
  - Even single-slice entries use `slices[]`.
- `__edek__` / `__ez_id__`: encryption metadata, present only when encrypted. `__ez_id__` stored as string.
- `slice_size`: plaintext slice size, present only when encrypted.

**Detection**: `__edek__` present → encrypted; absent → unencrypted.

---

## 3. Upload Path

### 3.1 IndexEntryWriter Interface

Index classes only see this interface, unaware of encryption and transport:

```cpp
// storage/IndexEntryWriter.h
namespace milvus::storage {

constexpr char MILVUS_V3_MAGIC[] = "MVSIDXV3";
constexpr size_t MILVUS_V3_MAGIC_SIZE = 8;

class IndexEntryWriter {
public:
    virtual ~IndexEntryWriter() = default;
    virtual void WriteEntry(const std::string& name, const void* data, size_t size) = 0;
    virtual void WriteEntry(const std::string& name, int fd, size_t size) = 0;
    virtual void Finish() = 0;
    virtual size_t GetTotalBytesWritten() const = 0;
};
}
```

**Entry name uniqueness**: `CheckDuplicateName` is in the base class, inherited by both subclasses. Duplicate names throw immediately.

### 3.2 Two Implementations

```
CreateIndexEntryWriterV3(filename, is_index_file)
    ├─ No encryption → IndexEntryDirectStreamWriter → direct write RemoteOutputStream
    └─ Encryption    → IndexEntryEncryptedLocalWriter → local temp file → upload
```

**Why two paths**:
- Unencrypted: data size predictable, write directly to remote stream. `RemoteOutputStream` internal `background_writes` handles parallel upload.
- Encrypted: ciphertext size unpredictable. Write to local file (>1GB/s, no network backpressure blocking encryption pipeline). Local file also provides task-level retry capability.

### 3.3 IndexEntryDirectStreamWriter (Unencrypted)

```
Index → WriteEntry → RemoteOutputStream → background_writes → S3 Multipart
```

```cpp
// storage/IndexEntryDirectStreamWriter.h
class IndexEntryDirectStreamWriter : public IndexEntryWriter {

    explicit IndexEntryDirectStreamWriter(
        std::shared_ptr<milvus::OutputStream> output,
        size_t read_buf_size = 16 * 1024 * 1024);

    void WriteEntry(const std::string& name, const void* data, size_t size) override;
    void WriteEntry(const std::string& name, int fd, size_t size) override;
    void Finish() override;
    size_t GetTotalBytesWritten() const override;
};
```

Constructor writes Magic. `Finish()` writes Directory Table JSON + Footer (uint64 dir_size), computes `total_bytes_written_`, closes stream.

### 3.4 IndexEntryEncryptedLocalWriter (Encrypted)

```
Index → WriteEntry → encrypt thread pool (parallel) → ordered queue → sequential write local file
                                                            ↓ Finish()
                                                 upload local file → S3
```

Encryption pipeline uses sliding window: W slices encrypt in parallel, dequeued in submission order, written sequentially to local file. Offset from actual `encrypted.size()` increment, always correct.

**Thread pool**: Reuses V2 global priority thread pool `ThreadPools::GetThreadPool(MIDDLE)`.

**Thread safety**: Each slice encryption task creates its own `IEncryptor` inside the lambda, destroyed after use. No sharing, no caching — global thread pool threads outlive Writer, caching would leave stale edek.

```cpp
// storage/IndexEntryEncryptedLocalWriter.h
class IndexEntryEncryptedLocalWriter : public IndexEntryWriter {
public:
    IndexEntryEncryptedLocalWriter(const std::string& remote_path,
                         milvus_storage::ArrowFileSystemPtr fs,
                         std::shared_ptr<plugin::ICipherPlugin> cipher_plugin,
                         int64_t ez_id, int64_t collection_id,
                         size_t slice_size = 16 * 1024 * 1024);
    ~IndexEntryEncryptedLocalWriter();

    void WriteEntry(const std::string& name, const void* data, size_t size) override;
    void WriteEntry(const std::string& name, int fd, size_t size) override;
    void Finish() override;
    size_t GetTotalBytesWritten() const override;

private:
    void EncryptAndWriteSlices(const std::string& name, uint64_t original_size,
                               const uint8_t* data, size_t size);
    void UploadLocalFile();
};
```

Constructor obtains edek, creates temp file at `/tmp/milvus_enc_<uuid>`, writes Magic. Destructor cleans up fd and temp file. `Finish()` writes Directory Table (with `__edek__`, `__ez_id__`, `slice_size`) + Footer, closes fd, uploads local file via `UploadLocalFile()`, unlinks temp file.

### 3.5 Factory Method

```cpp
// In FileManagerImpl (storage/FileManager.h)
std::unique_ptr<IndexEntryWriter>
FileManagerImpl::CreateIndexEntryWriterV3(const std::string& filename,
                                          bool is_index_file = true) {
    if (plugin_context_) {
        auto cipher_plugin = PluginLoader::GetInstance().getCipherPlugin();
        if (cipher_plugin) {
            auto remote_path = is_index_file ? GetRemoteIndexObjectPrefixV2()
                                             : GetRemoteTextLogPrefixV2();
            remote_path += "/" + GetFileName(filename);
            return std::make_unique<IndexEntryEncryptedLocalWriter>(
                remote_path, fs_, cipher_plugin,
                plugin_context_->ez_id, plugin_context_->collection_id);
        }
    }
    return std::make_unique<IndexEntryDirectStreamWriter>(
        OpenOutputStream(filename, is_index_file));
}
```

---

## 4. Load Path

### 4.1 IndexEntryReader Interface

```cpp
// storage/IndexEntryReader.h
struct Entry {
    std::vector<uint8_t> data;
};

class IndexEntryReader {
public:
    static std::unique_ptr<IndexEntryReader> Open(
        std::shared_ptr<milvus::InputStream> input,
        int64_t file_size,
        int64_t collection_id = 0,
        ThreadPoolPriority priority = ThreadPoolPriority::HIGH);

    std::vector<std::string> GetEntryNames() const;
    Entry ReadEntry(const std::string& name);
    void ReadEntryToFile(const std::string& name, const std::string& local_path);
    void ReadEntriesToFiles(
        const std::vector<std::pair<std::string, std::string>>& name_path_pairs);

private:
    // Small entries (≤ 1MB) are cached after first read
    static constexpr size_t kSmallEntryCacheThreshold = 1 * 1024 * 1024;
    std::unordered_map<std::string, Entry> small_entry_cache_;
};
```

**Encryption plugin**: `Open()` does not take `ICipherPlugin` explicitly. If Directory Table contains `__edek__`, `IndexEntryReader` obtains the global cipher plugin via `PluginLoader::GetInstance().getCipherPlugin()`. Decryption uses `collection_id`, `ez_id` (from Directory Table), and `edek` to create `IDecryptor`.

**Thread pool**: Reuses V2 global priority pool `ThreadPools::GetThreadPool(priority)`. Default `HIGH`.

### 4.2 Load Flow

1. **Open()**: Validates Magic (1 ReadAt). Reads file tail (up to 64KB in one ReadAt) to get Footer + Directory Table. If Directory Table exceeds 64KB, does one additional ReadAt. Parses JSON, builds `name → EntryMeta` index. If `__edek__` present, obtains cipher plugin.
2. **ReadEntry(name)** (small entry): Checks `small_entry_cache_` first. One `ReadAt`; if encrypted, creates `IDecryptor` to decrypt. Result cached if ≤ 1MB.
3. **ReadEntry(name)** (large entry):
   - Encrypted: Thread pool concurrent `ReadAt` for each slice (boundaries from Directory Table), each task creates own `IDecryptor`, decrypts, copies to target buffer.
   - Unencrypted: Self-splits by 16MB ranges, concurrent `ReadAt`, copies to target buffer.
4. **ReadEntryToFile(name, path)**: Same as above but output via `pwrite` to local file. `pwrite` is lock-free (each range writes different offset). Encrypted path does `ftruncate` first.
5. **ReadEntriesToFiles(pairs)**: Sequential iteration, calls `ReadEntryToFile` for each pair.

**Thread safety**: Each decryption task creates own `IDecryptor` in lambda, destroyed after use.

### 4.3 IO Counts

| Step | IO Count |
|------|----------|
| Read Directory Table | 1–2 (tail read) |
| Read meta entry | 1 (all O(1) meta packed) |
| Read large data entry | Concurrent, by slice/range count |

---

## 5. Index Class Interface

### 5.1 Base Class

```cpp
// index/ScalarIndex.h
template <typename T>
class ScalarIndex : public IndexBase {
public:
    // V3 streaming upload — calls WriteEntries(), returns IndexStats
    IndexStatsPtr UploadV3(const Config& config) override;

    // V3 streaming load — opens packed file, calls LoadEntries()
    void LoadV3(const Config& config) override;

    virtual void WriteEntries(storage::IndexEntryWriter* writer) {
        ThrowInfo(Unsupported, "WriteEntries is not implemented");
    }
    virtual void LoadEntries(storage::IndexEntryReader& reader, const Config& config) {
        ThrowInfo(Unsupported, "LoadEntries is not implemented");
    }

protected:
    storage::MemFileManagerImplPtr file_manager_;

    // Controls remote path prefix for V3 upload/load:
    //   true  → index_files/...  (default, normal scalar indexes)
    //   false → text_log/...     (TextMatchIndex)
    bool is_index_file_ = true;
};
```

**UploadV3 flow**:
1. Build filename: `milvus_packed_<type>_index.v3` (type from `GetIndexType()`, lowercased).
2. Create writer via `file_manager_->CreateIndexEntryWriterV3(filename, is_index_file_)`.
3. Call `WriteEntries(writer.get())`.
4. Call `writer->Finish()`.
5. Return `IndexStats` with the single packed file path and `writer->GetTotalBytesWritten()`.

**LoadV3 flow**:
1. Read `INDEX_FILES` from config — must contain exactly 1 packed file path.
2. Open `InputStream` via `file_manager_->OpenInputStream(packed_file, is_index_file_)`.
3. Create `IndexEntryReader::Open(input, file_size, collection_id)`.
4. Call `LoadEntries(*reader, config)`.

### 5.2 Meta Packing

Each index packs all O(1) metadata into a **single meta entry**, eliminating multiple small IOs by design.

| Index | Meta Entry | Data Entries |
|-------|-----------|-------------|
| ScalarIndexSort | `SORT_INDEX_META` | `index_data` |
| BitmapIndex | `BITMAP_INDEX_META` | `BITMAP_INDEX_DATA` |
| StringIndexMarisa | (none) | `MARISA_TRIE_INDEX`, `MARISA_STR_IDS` |
| StringIndexSort | `STRING_SORT_META` | `index_data`, `valid_bitset` |
| InvertedIndexTantivy | `TANTIVY_META` | N tantivy files, `index_null_offset` |
| JsonInvertedIndex | `TANTIVY_META` (with `has_non_exist`) | N tantivy files, `index_null_offset`, `json_index_non_exist_offsets` |
| NgramInvertedIndex | `TANTIVY_META` | N tantivy files, `index_null_offset`, `ngram_avg_row_size` |
| RTreeIndex | `RTREE_META` | N rtree files, `index_null_offset` |
| HybridScalarIndex | `HYBRID_INDEX_META` | (delegated to internal index) |

Meta structure definitions:

```cpp
// ScalarIndexSort:  {"index_length": N, "num_rows": N, "is_nested": bool}
// BitmapIndex:      YAML format: {bitmap_index_length: N, bitmap_index_num_rows: N}
// StringIndexSort:  {"version": V, "num_rows": N, "is_nested": bool}
// Tantivy:          {"file_names": [...], "has_null": bool}
// Tantivy (Json):   {"file_names": [...], "has_null": bool, "has_non_exist": bool}
// RTree:            {"file_names": [...], "has_null": bool}
// Hybrid:           {"index_type": uint8}
```

### 5.3 Inheritance Patterns

**InvertedIndexTantivy** provides `BuildTantivyMeta()` as a virtual hook. Subclasses extend by:

| Subclass | Pattern |
|----------|---------|
| JsonInvertedIndex | Overrides `BuildTantivyMeta` to add `has_non_exist`. Overrides `WriteEntries`/`LoadEntries`, calling parent first then adding extra entries. |
| NgramInvertedIndex | Overrides `WriteEntries`/`LoadEntries`, calling parent first then adding `ngram_avg_row_size`. |

This keeps parent meta fields (`file_names`, `has_null`) in the single `TANTIVY_META` entry, one IO. Subclass-specific entries are separate.

---

## 6. Concurrency Model

All thread pools reuse V2 `ThreadPools::GetThreadPool(priority)`, no custom pools.

| Path | Thread Pool | V2 Equivalent |
|------|------------|---------------|
| Encrypted upload | `MIDDLE` (`MIDD_SEGC_POOL`) | V2 `PutIndexData` |
| Download/decrypt | Caller-specified, default `HIGH` (`HIGH_SEGC_POOL`) | V2 `GetObjectData` |
| Unencrypted upload | None (RemoteOutputStream internal `background_writes`) | — |

### Upload — Unencrypted

```
Main thread: [Write to RemoteOutputStream] ──→ sequential fill
RemoteOutputStream:  background_writes parallel upload Part 0, 1, 2...  ──→ S3
```

### Upload — Encrypted

```
MIDD_SEGC_POOL (W threads):  parallel Encrypt slice 0, 1, 2...
Main thread:  in-order .get() → sequential write to local file (>1GB/s)
    ↓ Finish()
Upload local file via RemoteOutputStream → S3
```

### Download/Load (Unified)

```
HIGH_SEGC_POOL (N threads, or caller-specified priority):
  Thread i: ReadAt(range_i) → [Decrypt*] → output to memory or pwrite to file
  * Decrypt only when encrypted
  "range" = encrypted: slice boundaries; unencrypted: 16MB self-split
```

---

## 7. Memory Peak

| Scenario | Peak |
|----------|------|
| Upload, unencrypted, memory entry | entry itself |
| Upload, unencrypted, disk entry | 1 × `read_buf_size` (16MB) |
| Upload, encrypted, memory entry | entry + W × `slice_size` |
| Upload, encrypted, disk entry | W × `slice_size` × 2 |
| Download, to memory | N × `range_size` + `original_size` |
| Download, to file | N × `range_size` (reusable) |

Consistent with V2: peak determined by concurrency × slice size, does not grow with entry size.

---

## 8. Compatibility

### Version Routing

V3 does **not** detect version by reading file Magic Number at load time. Version routing is entirely caller-side:

1. **Build side**: `ScalarIndexCreator::Upload()` checks `SCALAR_INDEX_ENGINE_VERSION` from config. If `>= 3`, calls `index->UploadV3(config)`; otherwise calls `index->Upload(config)`.
2. **Load side**: `SealedIndexTranslator` checks `scalar_index_engine_version` from config. If `>= 3` and not vector type, calls `index->LoadV3(config)`; otherwise calls `index->Load(ctx, config)`.
3. **Go control plane**: `CurrentScalarIndexEngineVersion` in `pkg/common/common.go` (currently `2`). Bump to `3` to enable V3 for new index builds.
4. **UT routing**: `kScalarIndexUseV3` global bool (default `false`). When set to `true`, individual index `Upload()` and `Load()` methods internally redirect to `UploadV3()`/`LoadV3()`. This flag is for unit testing only; production routing uses the caller-side config.

### File Naming

V3 packed file name format: `milvus_packed_<type>_index.v3`, where `<type>` is the lowercased `ScalarIndexType` string (e.g., `stlsort`, `bitmap`, `marisa`, `inverted`, `hybrid`, `rtree`, `ngram`).

### Remote Paths

V3 keeps V2 remote root directory classification unchanged:

| Root | Index Type | V3 Path |
|------|-----------|---------|
| `index_files/` | Normal scalar indexes, InvertedIndexTantivy, etc. | `.../milvus_packed_<type>_index.v3` |
| `text_log/` | TextMatchIndex | `.../milvus_packed_<type>_index.v3` |

`FileManagerImpl` has `OpenOutputStream(filename, bool is_index_file)` and `CreateIndexEntryWriterV3(filename, bool is_index_file)`. Default `is_index_file=true`. TextMatchIndex sets `is_index_file_ = false` on its `ScalarIndex` base.

### Backward Compatibility

- V2 `Upload()` and `Load()` methods retained, not deleted.
- Rollback: set `CurrentScalarIndexEngineVersion` back to `2`. No code change needed.

---

## 9. Open Questions

### 9.1 Data Integrity Checksums

V3 currently relies only on Magic Number + JSON Directory Table structural validation. No entry/slice-level integrity checks (CRC32/SHA256/HMAC).

Directory Table JSON is extensible — checksum fields can be added later without format changes. Decision: deferred until driven by actual need.

### 9.2 Why not encode the index into existing file format, like Lance?

Lance is a columnar storage format that stores data in a columnar format. Any single Lance file has its own schema, and multiple Lance files form a table format.

In the end of the day, Lance still follows a tabular format design, while using a columnar storage.

#### How does Lance store an index?

In most cases, indexes would have their own specialized data structure, algorithm, etc. Those are highly optimized for the index type, and may not fit easily into a tabular format.

Lance addresses this by storing indexes in **separate Lance files** (typically `index.idx` + `auxiliary.idx`) that are still columnar under the hood. These files use Lance's own Arrow-based schema tailored to the index type:

- For vector indexes (e.g. IVF_PQ), the index file stores graph/structure (e.g. IVF partitions, HNSW neighbors), while the auxiliary file stores quantized vector codes, both organized as columns with `_rowid`, codes, distances, etc.
- Even scalar indexes (bitmap, BTree-like) are encoded into columnar pages with metadata in Protobuf.

This approach works well for Lance's ecosystem — it reuses the same file format, zero-copy reads, and manifest versioning. However, it comes with trade-offs:

- **Forced columnar mapping** — arbitrary graph structures, tree nodes, hash tables, or custom layouts must be “flattened” into columns (e.g. lists/arrays), which can introduce overhead in encoding, padding, or indirection.
- **Schema rigidity** — every index type needs a custom Arrow schema + Protobuf metadata, limiting flexibility if you want to support diverse or evolving index algorithms.
- **Not truly generic** — third-party indexes (e.g. modified Faiss, custom implementations) are hard to import directly without re-implementation in Lance's layout.

For a **generic scalar index format** that aims to support many different scalar index types (bitmap, trie, inverted list, RTree, etc.) with minimal adaptation cost, a **pure KV style** (key → serialized index blob + metadata) avoids these constraints entirely.

It trades some of Lance's columnar optimizations for maximum flexibility and easier integration with existing index libraries.

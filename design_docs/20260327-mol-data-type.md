# MOL Data Type

## Summary

Introduce a new `MOL` (Molecule) data type in Milvus to support cheminformatics workloads, enabling native molecular storage, substructure/superstructure search, and molecular fingerprint-based similarity retrieval — all within the vector database.

## Background

Drug discovery, materials science, and chemical informatics pipelines increasingly need to combine **molecular substructure queries** with **vector similarity search** (e.g., find compounds that contain a benzene ring AND are most similar to a target in embedding space). Today this requires stitching together separate tools — RDKit for chemistry, a relational DB for substructure filters, and a vector DB for similarity — resulting in fragmented workflows and duplicated data.

Milvus already serves as the vector search backbone in many life-science applications. By adding a native MOL type with built-in RDKit integration, Milvus can offer an end-to-end solution: **store molecules, generate fingerprints, filter by substructure, and search by similarity — all in one system**.

## Requirement

Typical use cases:

- **Virtual screening**: Given a scaffold (e.g., benzene ring), find all molecules containing that scaffold, then rank by embedding similarity to a lead compound.
- **Chemical library management**: Store millions of SMILES, insert via CSV/JSON/Parquet bulk import, and query with substructure filters.
- **SAR (Structure-Activity Relationship) analysis**: Combine substructure filters with metadata filtering and vector search to explore structure-activity landscapes.

Key requirements:

1. A new scalar data type `MOL` that accepts SMILES strings as input and stores molecules in a canonical binary format.
2. Built-in `MOL_CONTAINS` expression for substructure and superstructure queries.
3. A `MOL_PATTERN` index for accelerating substructure queries via fingerprint pre-filtering.
4. Built-in `MolFingerprint` function type to generate BinaryVector fingerprints (Morgan, MACCS, RDKit) for vector similarity search.
5. Bulk import support for all formats (CSV, JSON, Parquet, NumPy).

## Design

### Data Model — Dual Representation

The MOL type uses a dual-representation model:

| Aspect | Format | Description |
|--------|--------|-------------|
| User-facing | SMILES string | Simplified Molecular-Input Line-Entry System, e.g., `CCO` (ethanol), `c1ccccc1` (benzene) |
| Internal storage | RDKit binary pickle | Canonical binary serialization via RDKit `MolToMolBlock` / `MolToBinary` |

This design keeps the user interface simple (text strings) while storing a canonical binary form that is efficient for serialization and enables direct RDKit operations without re-parsing.

### Schema

**Proto changes:**

```protobuf
enum DataType {
    // ... existing types ...
    Mol = 27;
}

// Internal binary storage
message MolArray {
    repeated bytes data = 1;
}

// User-facing SMILES strings
message MolSmilesArray {
    repeated string data = 1;
}

message ScalarField {
    oneof data {
        // ... existing fields ...
        MolArray mol_data = 13;
        MolSmilesArray mol_smiles_data = 14;
    }
}
```

**pymilvus usage:**

```python
from pymilvus import MilvusClient, DataType

client = MilvusClient("http://localhost:19530")

schema = MilvusClient.create_schema(auto_id=False, enable_dynamic_field=False)
schema.add_field(name="id", datatype=DataType.INT64, is_primary=True)
schema.add_field(name="smiles", datatype=DataType.VARCHAR, max_length=512)
schema.add_field(name="mol", datatype=DataType.MOL)
schema.add_field(name="fp", datatype=DataType.BINARY_VECTOR, dim=2048)

# Define a MolFingerprint function: mol -> fingerprint
schema.add_function(
    name="mol_to_fp",
    function_type=FunctionType.MolFingerprint,
    input_field_names=["mol"],
    output_field_names=["fp"],
    params={"fingerprint_type": "morgan", "fingerprint_size": "2048", "radius": "2"},
)

client.create_collection("molecules", schema=schema)

# Insert — user provides SMILES strings
data = [
    {"id": 1, "smiles": "CCO",         "mol": "CCO"},
    {"id": 2, "smiles": "c1ccccc1",    "mol": "c1ccccc1"},
    {"id": 3, "smiles": "CC(=O)Oc1ccccc1C(=O)O", "mol": "CC(=O)Oc1ccccc1C(=O)O"},
]
client.insert("molecules", data)

# Substructure query — find molecules containing benzene ring
results = client.query(
    collection_name="molecules",
    filter="MOL_CONTAINS(mol, 'c1ccccc1')",
    output_fields=["smiles", "mol"],
)

# Superstructure query — find molecules that are substructures of aspirin
results = client.query(
    collection_name="molecules",
    filter="MOL_CONTAINS('CC(=O)Oc1ccccc1C(=O)O', mol)",
    output_fields=["smiles", "mol"],
)

# Similarity search — find top-10 most similar molecules to query
results = client.search(
    collection_name="molecules",
    data=["c1ccc(O)cc1"],  # SMILES as search input, auto-converted to fingerprint
    anns_field="fp",
    limit=10,
    output_fields=["smiles", "mol"],
)

# Hybrid: substructure filter + similarity search
results = client.search(
    collection_name="molecules",
    data=["c1ccc(O)cc1"],
    anns_field="fp",
    limit=10,
    filter="MOL_CONTAINS(mol, 'c1ccccc1')",
    output_fields=["smiles", "mol"],
)
```

### RDKit Integration via CGO

Milvus integrates [RDKit](https://www.rdkit.org/) (the de-facto open-source cheminformatics toolkit) through a C-linkage bridge layer:

```
Go code (internal/util/function/mol/*.go)
    |
    v  CGO bridge
C API (internal/core/src/common/mol_c.h)  -- extern "C"
    |
    v
C++ RDKit library (Conan package rdkit/2025.09.1@milvus/dev)
```

The C API (`mol_c.h`) exposes the following categories of functions:

| Category | Functions | Description |
|----------|-----------|-------------|
| Conversion | `ConvertSMILESToPickle`, `ConvertPickleToSMILES` | Bidirectional SMILES ↔ pickle conversion |
| Fingerprint | `GenerateMorganFingerprint`, `GenerateMACCSFingerprint`, `GenerateRDKitFingerprint`, `GeneratePatternFingerprint` | Generate various molecular fingerprints (from SMILES or pickle) |
| Substructure | `HasSubstructMatch`, `HasSubstructMatchWithQuery`, `HasSubstructMatchWithMol` | Exact substructure/superstructure matching |
| Handle mgmt | `ParseSMILESToMol`, `FreeMolHandle` | Parse SMILES into opaque handle for reuse across rows |

**Build considerations:**

- `mol_c.cpp` is compiled with GCC 11 (required for RDKit's C++20 constexpr destructors) while the rest of Milvus uses GCC 9.
- RDKit is integrated as a Conan package (`rdkit/2025.09.1@milvus/dev`).
- SMILES parsing uses relaxed sanitization (`SmilesToMolRelaxed`) that skips valence checking to accept hypervalent molecules common in drug discovery datasets.

### Insert Flow

```
Client sends SMILES strings
    │
    ▼
Proxy: checkMOLFieldData()
    - Validates each SMILES via ConvertSMILESToPickle()
    - Replaces MolSmilesData with MolData (pickle bytes) in-place
    │
    ▼  [if MolFingerprint function defined]
Proxy: MolFingerprintEmbeddingFunction.ProcessInsert()
    - Generates fingerprint BinaryVector from SMILES
    - Appends fingerprint output field data
    │
    ▼
StreamingNode: writes to segment
    - MOL data stored as binary bytes in binlog (arrow::binary)
    - Growing segment: MolPatternIndex.AppendMolRow() incrementally
    │
    ▼
IndexNode: builds sealed segment indexes
    - MolPatternIndex.Build(): PatternFP for all rows
    - Standard BinaryVector index on fingerprint output field
```

On disk, MOL data maps to `arrow::binary()`, following the same storage path as the Geometry type.

### Query / Search Flow

```
Client sends query with MOL_CONTAINS filter and/or fingerprint search
    │
    ├── Search path (fingerprint similarity):
    │   Proxy: MolFingerprintEmbeddingFunction.ProcessSearch()
    │       SMILES → pickle → fingerprint
    │       (ensures canonical consistency with insert path)
    │   QueryNode: standard ANN search on BinaryVector index
    │
    └── Filter path (MOL_CONTAINS):
        Proxy: parser creates MolFunctionFilterExpr
        QueryNode/Segment:
            1. Parse query SMILES into MolHandle (once, reused for all rows)
            2. If MOL_PATTERN index exists:
               - Generate PatternFP for query molecule
               - BruteForce RangeSearch on PatternFP index → candidate bitmap
               (conservative: false positives OK, false negatives not)
            3. For each candidate row:
               - HasSubstructMatchWithQuery() (substructure)
               - or HasSubstructMatchWithMol() (superstructure)
            4. Result: boolean bitmap

Result return:
    QueryNode returns MolData (pickle bytes)
    Proxy: validateMOLFieldSearchResult()
        pickle → SMILES via ConvertPickleToSMILES()
        Returns MolSmilesArray to client
```

### MOL_CONTAINS Expression

**Syntax:**

```sql
-- Substructure: find molecules in 'mol_field' that contain query as substructure
MOL_CONTAINS(mol_field, 'query_smiles')

-- Superstructure: find molecules in 'mol_field' that are substructures of query
MOL_CONTAINS('query_smiles', mol_field)
```

The argument order determines the operation:

| Syntax | Semantics | RDKit Operation |
|--------|-----------|-----------------|
| `MOL_CONTAINS(field, 'smiles')` | field molecule contains query substructure | `HasSubstructMatch(field_mol, query_mol)` |
| `MOL_CONTAINS('smiles', field)` | query molecule contains field as substructure | `HasSubstructMatch(query_mol, field_mol)` |

**Proto definition:**

```protobuf
message MolFunctionFilterExpr {
    ColumnInfo column_info = 1;
    string smiles_string = 2;
    MolOp op = 3;
}

enum MolOp {
    Invalid = 0;
    Substructure = 1;
    Superstructure = 2;
}
```

**Grammar (Plan.g4):**

```antlr
MOLContains: 'mol_contains' | 'MOL_CONTAINS';
```

### MOL_PATTERN Index

A specialized scalar index that pre-computes **Pattern Fingerprints** for every molecule in a segment, enabling conservative pre-filtering of substructure queries.

| Property | Value |
|----------|-------|
| Index type | `PATTERN` |
| Default dimension | 2048 bits |
| False positives | Possible (conservative) |
| False negatives | Not possible |
| Standard scalar ops | Not supported (In, NotIn, Range all throw) |

**Growing segment support (lock-free):**

- Pre-allocates `max_row_count * bytes_per_row` buffer at construction (zero-filled).
- `AppendMolRow(row_offset, pickle_data, is_valid)`: computes PatternFP at exact offset.
- `published_row_count_` is an `atomic<int64_t>` high-water mark, advanced via CAS by concurrent writers.
- Zero-FP gaps are safe because query visibility is bounded by `AckResponder::GetAck()` (continuous prefix guarantee).

**Sealed segment support:**

- `Build()`: reads raw pickle data, generates PatternFP for each row.
- `Serialize()`: meta (dim + row_count) + FP data binary blob.
- `Load()`: supports both memory and mmap paths.
- `Upload()`: serializes to object storage via `MemFileManager`.

### MolFingerprint Function

A new `FunctionType.MolFingerprint` (value 5) that automatically generates BinaryVector fingerprints from MOL fields, enabling vector similarity search on molecular structures.

**Supported fingerprint types:**

| Type | Function | Default Params | Output Dim |
|------|----------|---------------|------------|
| `morgan` | Morgan/ECFP circular fingerprints | radius=2, size=2048 | Configurable |
| `maccs` | MACCS structural keys | (none) | 168 (167 meaningful + 1 padding) |
| `rdkit` | RDKit path-based fingerprints | minPath=1, maxPath=7, size=2048 | Configurable |

**Parameters** (set via `FunctionSchema.params`):

| Parameter | Default | Description |
|-----------|---------|-------------|
| `fingerprint_type` | `"morgan"` | One of: morgan, maccs, rdkit |
| `fingerprint_size` | `2048` | Bit length of fingerprint (ignored for MACCS) |
| `radius` | `2` | Morgan only: circular radius |
| `min_path` | `1` | RDKit only: minimum path length |
| `max_path` | `7` | RDKit only: maximum path length |

**Insert path**: Takes SMILES from `MolSmilesData`, generates fingerprints, returns `BinaryVector` FieldData.

**Search path**: Takes SMILES from `PlaceholderGroup`, converts SMILES→pickle→fingerprint to ensure canonical consistency with insert path, returns `BinaryVector` PlaceholderGroup.

### Bulk Import

All four import formats support MOL fields. In each case, molecules are provided as SMILES strings and converted to pickle during import.

| Format | Source Type | Conversion |
|--------|------------|------------|
| CSV | String column | `ConvertSMILESToPickle(string_value)` |
| JSON | String value | `ConvertSMILESToPickle(string_value)` |
| Parquet | Arrow StringType | `ReadMolData()` → string → pickle |
| NumPy | VarChar array | String → `ConvertSMILESToPickle()` |

All formats support nullable MOL fields with default values.

### Storage

MOL data is stored as binary bytes (`arrow::binary()`) in the columnar storage layer, sharing the same serialization path as the Geometry type.

| Component | Handling |
|-----------|----------|
| `MolFieldData` | `Data [][]byte`, `ValidData []bool`, `Nullable bool` |
| `data_codec.go` | Serializes via `AddOneMolToPayload()`, deserializes as `[][]byte` |
| `payload_writer/reader` | Binary bytes path (same as Geometry) |
| `serde.go` | Mapped to `byteEntry` serialization |
| `data_sorter.go` | Sorted by pickle byte representation |

## Implementation Plan

### Development Sequence (by PR)

| PR | Scope | Description |
|----|-------|-------------|
| PR1 | Go layer basics | MOL type registration, insert/query/test, `GenEmptyFieldData` fix |
| PR2 | C++ layer | MOL storage and query support in segcore, expression evaluation |
| PR3 | RDKit CGO | RDKit Conan package + CGO bridge + SMILES↔Pickle conversion |
| PR4 | Fingerprint functions | Morgan/MACCS/RDKit fingerprint functions + embedding pipeline |
| PR5 | Bulk import | CSV/JSON/Parquet/NumPy import support |
| PR6 | MOL_CONTAINS + index | `MOL_CONTAINS` expression + `MOL_PATTERN` index + `FromPickle` optimization |

Dependency chain: PR1 → PR2 → PR3 → PR4 → PR5 → PR6

### Modification Points

#### Proto Layer

| File | Change |
|------|--------|
| `schema.proto` | Add `DataType.Mol = 27`, `MolArray`, `MolSmilesArray`, `FunctionType.MolFingerprint = 5` |
| `plan.proto` | Add `MolFunctionFilterExpr`, `MolOp` enum |

#### Go Layer

| File | Change |
|------|--------|
| `internal/proxy/validate_util.go` | `checkMOLFieldData()` (insert), `validateMOLFieldSearchResult()` (query/search) |
| `internal/parser/planparserv2/Plan.g4` | `MOLContains` grammar rule |
| `internal/parser/planparserv2/parser_visitor.go` | `VisitMolSubstructure()`, `VisitMolSuperstructure()` |
| `internal/util/function/mol_fingerprint_function.go` | `MolFingerprintFunctionRunner` |
| `internal/util/function/embedding/mol_fingerprint_embedding_function.go` | Embedding pipeline integration |
| `internal/util/function/mol/*.go` | CGO bridge + Go wrappers for all fingerprint types |
| `internal/util/indexparamcheck/mol_pattern_checker.go` | `MOLPatternChecker` for index validation |
| `internal/storage/insert_data.go` | `MolFieldData` struct |
| `internal/storage/data_codec.go`, `serde.go`, `data_sorter.go` | Serialization support |
| `internal/util/importutilv2/csv/`, `json/`, `parquet/`, `numpy/` | Bulk import parsers |
| `pkg/util/typeutil/schema.go` | `IsMolType()` utility |

#### C++ Core Layer

| File | Change |
|------|--------|
| `internal/core/src/common/Types.h` | `DataType::MOL = 27`, `IsMolDataType()` |
| `internal/core/src/common/mol_c.h/.cpp` | C-linkage RDKit bridge (conversion, fingerprint, substructure) |
| `internal/core/src/exec/expression/MolFunctionFilterExpr.h/.cpp` | `PhyMolFunctionFilterExpr` with FP pre-filter + exact match |
| `internal/core/src/index/MolPatternIndex.h/.cpp` | Pattern fingerprint index (growing + sealed) |
| `internal/core/src/segcore/FieldIndexing.cpp` | Growing segment MOL_PATTERN index integration |
| `internal/core/src/query/PlanProto.cpp` | Plan proto → expr conversion |
| `internal/core/src/exec/expression/Expr.cpp` | Expression compiler registration |
| `internal/core/thirdparty/rdkit/CMakeLists.txt` | RDKit Conan integration |
| `internal/core/src/common/CMakeLists.txt` | Separate GCC 11 compilation for mol_c.cpp |

## Limitations

- MOL fields only accept SMILES strings as input; other molecular formats (SDF, InChI) are not supported in the initial version.
- `MOL_CONTAINS` performs exact substructure matching, which can be computationally expensive for large molecules. The `MOL_PATTERN` index mitigates this via fingerprint pre-filtering.
- SMILES parsing uses relaxed sanitization (skips valence checking) to support hypervalent molecules — this means some chemically invalid SMILES may be accepted.
- MACCS fingerprint dimension is fixed at 168 bits (167 meaningful + 1 padding bit); it cannot be configured.
- The `MOL_PATTERN` index only supports substructure/superstructure screening; standard scalar index operations (In, NotIn, Range) are not supported on MOL fields.

## References

- [RDKit: Open-Source Cheminformatics](https://www.rdkit.org/)
- [SMILES specification](https://www.daylight.com/dayhtml/doc/theory/theory.smiles.html)
- [Morgan fingerprints (ECFP)](https://pubs.acs.org/doi/10.1021/ci100050t)
- [MACCS keys](https://docs.chemaxon.com/display/docs/maccs-fingerprints.md)

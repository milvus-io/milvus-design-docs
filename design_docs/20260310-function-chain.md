# Function Chain

## Summary

The function chain system is a composable, operator-based data processing pipeline built on Apache Arrow for reranking and transforming search results in Milvus. It models multi-stage result processing as a sequence of typed operators (Merge, Map, Filter, Sort, Limit, GroupBy, Select) applied to columnar DataFrames, enabling flexible construction of reranking strategies such as RRF, Weighted, Decay, and Model-based reranking.

## Motivation

Milvus hybrid search produces multiple result sets from different vector indexes and scoring metrics. These results must be merged, rescored, filtered, and truncated before returning to the user. Prior to the function chain system, this logic was implemented as monolithic, hard-to-extend code paths.

Key requirements driving this design:

- **Multi-strategy reranking** — users need different fusion strategies (RRF, weighted scoring, decay functions, model-based reranking) depending on their use case.
- **Composable processing** — operations like score normalization, filtering, grouping, and limiting should be independently composable rather than hard-coded into a single pipeline.
- **Mixed metric handling** — hybrid search may combine results scored with different metrics (IP, L2, COSINE, BM25), requiring metric-aware normalization before fusion.
- **Columnar efficiency** — search results carry IDs, scores, and arbitrary field data. A columnar representation based on Apache Arrow avoids per-row allocation overhead and integrates well with vectorized operations.
- **Serialization** — chains must be representable as JSON for storage, debugging, and potential cross-component transfer.

## Design

### Architecture Overview

The system is organized into four layers:

```
┌─────────────────────────────────────────────────────────┐
│                   Integration Layer                      │
│  search_pipeline.go (rerankOperator)                     │
│  - SearchResultData → DataFrame (converter)              │
│  - BuildRerankChain (rerank_builder)                     │
│  - ExecuteWithContext                                    │
│  - DataFrame → SearchResultData (converter)              │
├─────────────────────────────────────────────────────────┤
│                   Orchestration Layer                     │
│  FuncChain                                               │
│  - Fluent builder API (Merge/Map/Filter/Sort/Limit/...)  │
│  - Sequential operator execution                         │
│  - Stage validation, error deferral                      │
│  - Intermediate result lifecycle management              │
├─────────────────────────────────────────────────────────┤
│                    Operator Layer                         │
│  MergeOp │ MapOp │ FilterOp │ SortOp │ LimitOp │ ...    │
│  - Each operator: DataFrame in → DataFrame out           │
│  - MapOp/FilterOp delegate to FunctionExpr               │
│  - Per-chunk independent processing                      │
├─────────────────────────────────────────────────────────┤
│                   Expression Layer                        │
│  DecayExpr │ ScoreCombineExpr │ ModelExpr │ ...          │
│  - Stateless column-level computation                    │
│  - []*arrow.Chunked in → []*arrow.Chunked out            │
│  - Stage-aware (IsRunnable)                              │
├─────────────────────────────────────────────────────────┤
│                     Data Layer                            │
│  DataFrame (immutable, Arrow-backed)                     │
│  DataFrameBuilder / ChunkCollector                       │
│  Converter (Milvus protobuf ↔ Arrow)                     │
└─────────────────────────────────────────────────────────┘
```

- **Integration Layer** — connects the chain to Milvus's proxy search pipeline. Handles data format conversion and chain construction from user-facing schema definitions.
- **Orchestration Layer** — `FuncChain` manages operator sequencing, validation, context propagation, and intermediate result cleanup.
- **Operator Layer** — each operator transforms a DataFrame independently. `MapOp` and `FilterOp` delegate computation to `FunctionExpr`, keeping column mapping separate from computation logic.
- **Expression Layer** — stateless functions that operate on raw Arrow columns. Decoupled from column naming, making them reusable across different operator contexts.
- **Data Layer** — `DataFrame` as the universal data currency between operators, with builder utilities for safe construction and a converter for bridging Milvus protobuf types.

#### Operator Pipeline

`FuncChain` executes operators sequentially. Each operator produces a new DataFrame consumed by the next:

```
Input DataFrames
      |
      v
 +---------+     +-------+     +--------+     +-------+     +--------+
 | MergeOp | --> | MapOp | --> | SortOp | --> | Limit | --> | Select |
 +---------+     +-------+     +--------+     +-------+     +--------+
                                                                 |
                                                                 v
                                                         Output DataFrame
```

The chain supports a fluent builder API:

```go
chain := NewFuncChainWithAllocator(alloc).
    SetStage(types.StageL2Rerank).
    Merge(MergeStrategyRRF, WithRRFK(60)).
    Sort(types.ScoreFieldName, true).
    Limit(10).
    Select(types.IDFieldName, types.ScoreFieldName)
```

Errors in fluent calls are deferred — accumulated internally and surfaced at `Execute()` or `Validate()` time. This keeps the builder API clean without requiring error checks at every step.

### Data Model

The core data structure is `DataFrame`, an immutable columnar container backed by Apache Arrow chunked arrays.

**Key properties:**

- **Columns** are named `*arrow.Chunked` arrays, accessed by name via `Column(name)`.
- **Chunks** represent per-query partitions. For search results with NQ=3, each column has 3 chunks, one per query. Operators like Sort and Limit process chunks independently, preserving per-query semantics.
- **Schema** tracks Arrow types, while additional metadata maps (`fieldTypes`, `fieldIDs`, `fieldNullables`) preserve Milvus-specific type information through the pipeline.
- **Reserved column names**: `$id` for result IDs (Int64 or String), `$score` for result scores (Float32).

Construction uses `DataFrameBuilder` for safe resource management — columns are added incrementally, and `Build()` atomically transfers ownership. If the builder is released before building, all accumulated arrays are freed.

`ChunkCollector` assists multi-column operators that build output chunk-by-chunk, tracking which chunks have been consumed to prevent resource leaks on error.

### Core Interfaces

**Operator** — the pipeline unit:

```go
type Operator interface {
    Name() string
    Execute(ctx *types.FuncContext, input *DataFrame) (*DataFrame, error)
    Inputs() []string
    Outputs() []string
    String() string
}
```

**FunctionExpr** — a stateless computation applied within Map/Filter operators:

```go
type FunctionExpr interface {
    Name() string
    OutputDataTypes() []arrow.DataType
    Execute(ctx *FuncContext, inputs []*arrow.Chunked) ([]*arrow.Chunked, error)
    IsRunnable(stage string) bool
}
```

The separation between Operator and FunctionExpr keeps column mapping (which columns to read/write) in the Operator layer, while FunctionExpr focuses purely on computation. This allows the same function to be reused in different column contexts.

**FuncContext** carries execution state:

- `context.Context` for cancellation and timeouts
- `memory.Allocator` for Arrow memory management
- `stage` string for stage-based function filtering

### Execution Stages

Functions and chains are tagged with execution stages to control where they run:

| Stage | Value | Description |
|-------|-------|-------------|
| Ingestion | `ingestion` | Insert/upsert/import |
| L2 Rerank | `L2_rerank` | Proxy-level reranking |
| L1 Rerank | `L1_rerank` | QueryNode/StreamingNode reranking |
| L0 Rerank | `L0_rerank` | Segment-level reranking |
| Pre-Process | `pre_process` | Pre-processing |
| Post-Process | `post_process` | Post-processing |

A `FunctionExpr` declares which stages it supports via `IsRunnable(stage)`. The chain validates stage compatibility at execution time.

### Built-in Operators

#### MergeOp

Merges multiple DataFrames into one, combining scores from rows sharing the same ID. Must be the first operator in a chain. Supports five strategies:

| Strategy | Behavior |
|----------|----------|
| `rrf` | Reciprocal Rank Fusion: `score += 1 / (k + rank)`, default k=60 |
| `weighted` | Weighted sum of normalized scores |
| `max` | Maximum score across inputs |
| `sum` | Sum of scores across inputs |
| `avg` | Average of scores across inputs |

**Metric-aware normalization** handles mixed metric types. Each metric has a normalization function that maps scores to [0, 1]:

- COSINE: `(1 + score) * 0.5`
- IP: `0.5 + atan(score) / pi`
- BM25: `2 * atan(score) / pi`
- L2: `1.0 - 2 * atan(distance) / pi`

For non-normalized mode with mixed metrics, a direction conversion function flips L2-style scores (smaller-is-better) to larger-is-better without full normalization.

The output sort direction is determined by the metric types — if all metrics are L2, output is ascending; otherwise descending.

#### MapOp

Applies a `FunctionExpr` to specified input columns and writes results to output columns. The function's output column count must match the declared `OutputDataTypes()` length (skipped for dynamic-type functions). Non-output columns pass through unchanged.

#### FilterOp

Applies a boolean-returning `FunctionExpr` to input columns. Rows where the function returns `true` are kept; others are removed. Chunk sizes are updated to reflect filtered row counts.

#### SortOp

Sorts rows within each chunk by a specified column in ascending or descending order. Supports an optional tie-break column (the chain's fluent `Sort()` defaults tie-breaking on `$id`). Sorting is stable (`sort.SliceStable`). Supports all comparable types: integers, floats, and strings.

#### LimitOp

Truncates each chunk independently to `limit` rows after skipping `offset` rows. Applied per-chunk, so for NQ=3 with limit=10, each of the 3 result sets is independently limited to 10 rows.

#### GroupByOp

Groups rows within each chunk by a field value, keeps the top `groupSize` rows per group (sorted by `$score` descending), scores each group, then returns the top groups after applying offset/limit.

Group scoring strategies:

| Scorer | Behavior |
|--------|----------|
| `max` | Highest score in the group (default) |
| `sum` | Sum of scores in the group |
| `avg` | Average score in the group |

Adds a `$group_score` column to the output DataFrame.

#### SelectOp

Projects the DataFrame to a subset of columns, dropping all others. Used at the end of chains to select only the columns needed in the final result.

### Built-in Function Expressions

#### DecayExpr

Applies a distance-based decay to scores. Takes two inputs: a numeric distance column and the current score column. The output score is `original_score * decay_factor`.

Three decay functions are supported:

- **Gaussian**: `exp(-(d^2) / (scale^2 / ln(decay)))`
- **Exponential**: `exp((ln(decay) / scale) * d)`
- **Linear**: `max(decay, 1 - ((1-decay) / scale) * d)`

Where `d = max(0, |value - origin| - offset)`.

Parameters: `function` (gauss/exp/linear), `origin`, `scale` (>0), `offset` (>=0, default 0), `decay` (0 < decay < 1, default 0.5). Validated for numerical stability (0.001 < decay < 0.999).

#### ScoreCombineExpr

Combines multiple score columns into a single score. Modes: `multiply` (default), `sum`, `max`, `min`, `avg`, `weighted`.

#### RoundDecimalExpr

Rounds float32 scores to a specified number of decimal places (0-6). Used internally by the rerank builder, not registered in the public function registry.

#### ModelExpr

Performs model-based reranking by calling an external ML rerank service. Takes a text/VarChar column as input, sends documents to the model provider in batches, and returns model-assigned scores. One query string per chunk (NQ).

### Rerank Chain Builder

`BuildRerankChain` constructs a complete reranking `FuncChain` from a `FunctionSchema` definition. This is the primary entry point used by the proxy search pipeline.

Each reranker name maps to a specific chain structure:

**RRF** (`rrf`):
```
MergeOp(rrf, k) -> SortOp($score DESC) -> LimitOp(limit, offset) -> SelectOp(...)
```

**Weighted** (`weighted`):
```
MergeOp(weighted, weights, normalize) -> SortOp($score DESC) -> LimitOp(limit, offset) -> SelectOp(...)
```

**Decay** (`decay`):
```
MergeOp(max) -> MapOp(DecayExpr, [field, $score] -> [$score]) -> SortOp($score DESC) -> LimitOp(limit, offset) -> SelectOp(...)
```

**Model** (`model`):
```
MergeOp(max) -> MapOp(ModelExpr, [text_field] -> [$score]) -> SortOp($score DESC) -> LimitOp(limit, offset) -> SelectOp(...)
```

When grouping is enabled, `SortOp + LimitOp` is replaced by `GroupByOp`:
```
MergeOp(...) -> [MapOp(...)] -> GroupByOp(field, groupSize, limit, offset) -> SelectOp(...)
```

`BuildRerankChainWithLegacy` provides backward compatibility by converting legacy rank parameters (used by older SDK versions) into the `FunctionSchema` format before delegating to the same builder.

### Serialization

Chains are serializable to/from JSON via a representation layer. The JSON format:

```json
{
  "name": "rerank_chain",
  "stage": "L2_rerank",
  "operators": [
    {
      "type": "sort",
      "params": {"column": "$score", "desc": true, "tie_break_col": "$id"},
      "inputs": ["$score"],
      "outputs": ["$score"]
    },
    {
      "type": "limit",
      "params": {"limit": 10, "offset": 0}
    },
    {
      "type": "select",
      "inputs": ["$id", "$score"]
    }
  ]
}
```

For Map/Filter operators, a nested `function` field describes the FunctionExpr:

```json
{
  "type": "map",
  "function": {"name": "decay", "params": {"function": "gauss", "origin": 0, "scale": 10}},
  "inputs": ["distance", "$score"],
  "outputs": ["$score"]
}
```

Deserialization uses factory registries — `MustRegisterOperator` for operators and `MustRegisterFunction` for function expressions. Each operator and function type registers its factory in an `init()` function, enabling extensibility without modifying the core parsing code.

> **TODO:** MergeOp is currently not registered in the operator registry and is special-cased in `FuncChain`. It should be registered like other operators to support full JSON round-trip serialization.

### Data Conversion

The converter bridges Milvus protobuf types and Arrow-based DataFrames:

**Import** (`FromSearchResultData`): Converts `schemapb.SearchResultData` into a DataFrame. Each query's results become one chunk. Handles Int64 and String ID types, scores, field data with null bitmaps, and all scalar Milvus types.

**Export** (`ToSearchResultData`): Converts a DataFrame back to `schemapb.SearchResultData`. Supports an `ExportOptions` struct for specifying the group-by field (which becomes `GroupByFieldValue` in the output).

**Type mapping** between Milvus and Arrow:

| Milvus Type | Arrow Type |
|-------------|------------|
| Bool | Boolean |
| Int8 | Int8 |
| Int16 | Int16 |
| Int32 | Int32 |
| Int64 | Int64 |
| Float | Float32 |
| Double | Float64 |
| VarChar/String | String (utf8) |

### Integration with Proxy Search Pipeline

#### Pipeline Structure

The proxy search pipeline (`search_pipeline.go`) is a node-based DAG. Each node wraps an operator, with named inputs/outputs wired between nodes. The pipeline variant is selected at construction time based on search type, rerank presence, and requery needs:

```
┌─────────────────────────────────────────────────────────────────┐
│                     newBuiltInPipeline(searchTask)               │
│                                                                  │
│  Conditions:                     Pipeline Selected:              │
│  ─────────                       ─────────────────               │
│  simple search, no rerank     →  reduce → pick                   │
│  common search + rerank       →  reduce → rerank → pick → ...    │
│  hybrid search                →  reduce(hybrid) → rerank → ...   │
│  hybrid + requery + rerank    →  reduce(hybrid) → requery        │
│                                    → organize → rerank → ...     │
└─────────────────────────────────────────────────────────────────┘
```

The `rerankOperator` always sits **immediately after the reduce phase** (or after requery when field data is needed for reranking). It consumes reduced `SearchResults` from all shards and produces a single reranked result.

A typical hybrid search with rerank pipeline:

```
reduce(hybrid) → rerank → pick_ids → requery → organize → result
       │              │
       │              └─ rerankOperator: builds and executes FuncChain
       └─ hybridSearchReduceOperator: merges results from all shards
```

#### Rerank Metadata

The `rerankOperator` receives its configuration through a `rerankMeta` interface, which has two implementations:

- **`funcScoreRerankMeta`** — constructed from `FunctionScore` in the search request. Carries the `FunctionScore` proto, input field names, and field IDs.
- **`legacyRerankMeta`** — constructed from legacy `SearchParams` key-value pairs for backward compatibility with older SDKs.

The metadata is populated during search task preparation:

```go
// task_search.go
if t.request.FunctionScore != nil {
    t.rerankMeta = newRerankMeta(t.schema.CollectionSchema, t.request.FunctionScore)
} else {
    t.rerankMeta = newRerankMetaFromLegacy(t.request.GetSearchParams())
}
```

#### Execution Flow

Within `rerankOperator.run()`:

1. **Input preparation** — each sub-search's `SearchResultData` is converted to a DataFrame via `FromSearchResultData`, importing only fields needed by the chain (whitelist-based filtering via `neededFields`).
2. **Chain construction** — `buildChainFromMeta` dispatches by metadata type:
   - `funcScoreRerankMeta` → `chain.BuildRerankChain(collSchema, funcScore, metrics, searchParams, alloc)`
   - `legacyRerankMeta` → `chain.BuildRerankChainWithLegacy(collSchema, legacyParams, metrics, searchParams, alloc)`
3. **Execution** — `chain.ExecuteWithContext(ctx, dataframes...)` runs the full operator pipeline. Context propagation enables cancellation.
4. **Output conversion** — the result DataFrame is converted back to `SearchResultData` via `ToSearchResultDataWithOptions`, preserving group-by field information if applicable.

### Memory Management

The system uses Apache Arrow's reference-counted memory model with explicit lifecycle management:

- **Allocator injection** — `NewFuncChainWithAllocator` accepts a `memory.Allocator` (defaults to `memory.DefaultAllocator`). The allocator is propagated to all operators via `FuncContext`.
- **Intermediate cleanup** — the chain automatically releases intermediate DataFrames between operators. Only the original inputs and final output survive execution.
- **Builder safety** — `DataFrameBuilder.Release()` frees all accumulated arrays if `Build()` was never called, preventing leaks on error paths.
- **ChunkCollector** — tracks consumed vs. unconsumed chunks. On `Release()`, only unconsumed chunks are freed, avoiding double-free.
- **Ownership transfer** — `DataFrameBuilder.Build()` transfers ownership atomically; subsequent `Release()` on the builder is a no-op.

## Limitations

- MergeOp must be the first operator in a chain; it cannot appear at other positions.
- Only one rerank function per search is currently supported at the proxy level (boost functions are handled at QueryNode).
- `FunctionExpr` instances are stateless — any state (e.g., model connections) must be managed externally and injected at construction time.
- GroupByOp and SortOp+LimitOp are mutually exclusive in rerank chains — grouping replaces sorting and limiting.
- The JSON serialization format does not include MergeOp (merge configuration is reconstructed from the rerank function schema at build time).
- Decay function validation enforces numerical stability bounds (0.001 < decay < 0.999), which may reject extreme decay values.

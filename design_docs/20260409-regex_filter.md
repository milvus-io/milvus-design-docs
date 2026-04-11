# Regex Filter Expression Support

## Background

Milvus currently supports the `LIKE` operator for pattern matching on VARCHAR fields, which covers simple wildcard patterns (`%`, `_`). However, users frequently need more expressive pattern matching capabilities — for example, matching email addresses (`[a-zA-Z]+@gmail\.com`), extracting log entries with structured patterns (`ERROR.*timeout`), or filtering URLs by path segments (`/api/v[0-9]+/users`).

While `LIKE` can handle some of these cases, it requires awkward workarounds or is simply incapable of expressing complex patterns. Regular expressions are the industry standard for this class of problems, supported natively by PostgreSQL (`~`), MySQL (`REGEXP`), ClickHouse (`match()`), and Elasticsearch (`regexp` query).

---

## Design Principles

- Provide a first-class regex operator in the Milvus expression language, with syntax and semantics familiar to users of SQL databases.
- Use **substring matching** semantics (match if any substring matches), which is the natural expectation for most users and consistent with `grep`, PostgreSQL `~`, and RE2 `PartialMatch`.
- Leverage existing infrastructure (RE2, ngram index, expression execution framework) to minimize implementation surface area.
- Support the same field types as `LIKE`: VARCHAR, JSON string values, and Array\<VARCHAR\>.

---

## Regex Flavor and Feature Scope

The regex engine is [RE2](https://github.com/google/re2) (already a Milvus dependency), which provides a safe, linear-time regex implementation. RE2 is the same engine used by Google BigQuery, ClickHouse, and CockroachDB.

### Supported Syntax

| Category | Examples | Supported |
|----------|---------|-----------|
| Literals | `abc`, `hello world` | Yes |
| Character classes | `[a-z]`, `[^0-9]`, `\d`, `\w`, `\s` | Yes |
| Quantifiers | `*`, `+`, `?`, `{n}`, `{n,m}` | Yes |
| Alternation | `cat\|dog` | Yes |
| Grouping | `(abc)`, `(?:abc)` | Yes |
| Anchors | `^` (start), `$` (end) | Yes |
| Dot (any character) | `.` (matches any char including `\n`) | Yes |
| Escape sequences | `\.`, `\*`, `\\` | Yes |
| Unicode | `\p{Han}`, `\p{Latin}` | Yes |
| Named groups | `(?P<name>...)` | Yes |
| Flags | `(?i)` case-insensitive, `(?s)` dot-all | Yes |

### NOT Supported (RE2 limitations)

| Feature | Notes |
|---------|-------|
| Backreferences | `\1`, `\2` — not supported by RE2 (requires backtracking) |
| Lookahead/Lookbehind | `(?=...)`, `(?<=...)` — not supported by RE2 |
| Possessive quantifiers | `*+`, `++` — not supported by RE2 |
| Atomic groups | `(?>...)` — not supported by RE2 |

These features require backtracking and could lead to catastrophic exponential runtime. RE2's linear-time guarantee makes it safe for use in a database context where untrusted patterns may be submitted.

### Matching Semantics

**Substring matching**: The expression `field =~ "pattern"` returns `true` if any substring of the field value matches the given regex pattern. This is equivalent to `RE2::PartialMatch` and to `grep` behavior.

Examples:
- `field =~ "error"` matches `"connection error occurred"` (substring "error" matches)
- `field =~ "^hello"` matches `"hello world"` but not `"say hello"` (anchored to start)
- `field =~ "world$"` matches `"hello world"` but not `"world cup"` (anchored to end)
- `field =~ "^hello world$"` matches only the exact string `"hello world"` (full-string match via anchors)
- `field =~ "[0-9]{3}-[0-9]{4}"` matches any string containing a phone-number-like pattern

Users can achieve full-string matching by using `^` and `$` anchors explicitly: `field =~ "^pattern$"`.

---

## Public Interfaces

### Expression Syntax

```
field =~ "regex_pattern"       -- regex match (substring semantics)
field !~ "regex_pattern"       -- regex not match (negation)
```

- `=~` is a new binary operator for regex matching. `!~` is its negation.
- `!~` is syntactic sugar: `field !~ "pattern"` is equivalent to `NOT (field =~ "pattern")`.
- The right operand must be a string literal containing a valid RE2 regex pattern.
- The left operand must be a VARCHAR field, a JSON field path that resolves to a string, or an Array\<VARCHAR\> field (element-level matching).

### SDK Examples

```python
# Python SDK
collection.query(
    expr='email =~ "[a-zA-Z0-9.]+@gmail\\.com"',
    output_fields=["email"]
)

# Negation: exclude entries matching a pattern
collection.query(
    expr='message !~ "^DEBUG"',
    output_fields=["message"]
)

# Filter log messages containing error codes
collection.query(
    expr='message =~ "E[0-9]{4}:"',
    output_fields=["message", "timestamp"]
)

# JSON field regex — must use path specifier, not root node
collection.query(
    expr='metadata["tag"] =~ "v[0-9]+\\.[0-9]+"',
    output_fields=["metadata"]
)

# Array<VARCHAR> field — must specify element index
collection.query(
    expr='tags[0] =~ "release-v[0-9]+"',
    output_fields=["tags"]
)

# Combined with vector search
collection.search(
    data=[query_vector],
    anns_field="embedding",
    param={"metric_type": "L2", "params": {"nprobe": 10}},
    filter='url =~ "/api/v[0-9]+/users"',
    output_fields=["url"]
)
```

### NULL Handling

Following SQL standard three-valued logic:
- `NULL =~ "pattern"` evaluates to `NULL` (unknown), not `false`.
- `NULL !~ "pattern"` evaluates to `NULL`.
- In filtering context, `NULL` results are treated as non-matching (rows are excluded).

### Error Handling

- Invalid regex patterns are rejected at parse time with a descriptive error message:
  ```
  Error: invalid regex pattern in =~ operator: missing closing ): `(unclosed`
  ```
- Applying `=~` to non-string fields is rejected at parse time:
  ```
  Error: regex match on non-string field is unsupported
  ```

---

## Design Details

### 1. Expression Parsing

The `=~` and `!~` operators are added to the ANTLR grammar (`Plan.g4`) as new tokens and rules:

```
REGEXMATCH: '=~';
REGEXNOTMATCH: '!~';

expr: ...
    | expr REGEXMATCH StringLiteral       # RegexMatch
    | expr REGEXNOTMATCH StringLiteral    # RegexNotMatch
    ...
```

The Go visitor (`VisitRegexMatch`) validates the field type (VARCHAR / JSON / Array\<VARCHAR\>), validates the regex syntax using Go's `regexp.Compile`, and produces a `UnaryRangeExpr` with `OpType = RegexMatch`:

```protobuf
// plan.proto
enum OpType {
  ...
  InnerMatch = 15;
  RegexMatch = 16;  // new
}
```

`VisitRegexNotMatch` reuses `VisitRegexMatch` and wraps the result with a `NOT` expression. No separate `RegexNotMatch` OpType is needed.

Before emitting `RegexMatch`, the parser attempts to optimize simple patterns to faster LIKE operations (see Section 5 below). Only patterns that cannot be optimized remain as `RegexMatch`.

### 2. Execution Engine

The C++ execution layer adds a `PartialRegexMatcher` class that wraps `RE2::PartialMatch`:

```cpp
struct PartialRegexMatcher {
    explicit PartialRegexMatcher(const std::string& pattern);
    bool operator()(const std::string& value);      // RE2::PartialMatch
    bool operator()(const std::string_view& value);  // RE2::PartialMatch
};
```

This contrasts with the existing `RegexMatcher` which uses `RE2::FullMatch` (used internally by `LIKE`'s `Match` op when the LIKE pattern has complex wildcards).

The `RegexMatch` op is integrated into the `UnaryExpr` execution pipeline:

- **Raw data path**: For each row, apply `PartialRegexMatcher` to produce a boolean result. The matcher is constructed once per batch (RE2 compilation), then reused across all rows. For JSON shared-data paths, a pre-built `PartialRegexMatcher` is stored in `UnaryCompareContext` to avoid per-row RE2 compilation.
- **Scalar index path**: Indexes that support `PatternMatch()` (Sort, Marisa, Bitmap, Inverted) iterate unique values and apply matching, which is O(unique\_values) instead of O(total\_rows). For inverted index (tantivy), the regex pattern is converted to tantivy-compatible syntax (`.` → `[\s\S]` for dot\_nl alignment, wrapped with `[\s\S]*(?:...)[\s\S]*` for substring semantics) and executed via `regex_query` on the term dictionary.
- **Execution ordering**: `RegexMatch` is classified as a "heavy" operation (same as `LIKE`), so it is reordered after cheaper indexed expressions in conjunctive filters.

### 3. Index Optimization: Ngram Index (Primary Path)

The ngram index is the primary optimization path for regex queries. It uses a two-phase approach:

**Phase 1 — Coarse Filter (index-only)**:
1. Extract fixed literal substrings from the regex pattern. For example:
   - `"error.*timeout"` → `["error", "timeout"]`
   - `"user_[0-9]+@gmail\.com"` → `["user_", "@gmail.com"]`
   - `"[a-z]+"` → `[]` (no literals extractable)
2. Break each literal into ngrams and query the ngram index.
3. Intersect posting lists to produce a candidate bitmap.
4. If no literals can be extracted, skip ngram optimization entirely (fall through to brute force).

**Phase 2 — Fine Filter (raw data verification)**:
1. For each candidate from Phase 1, load the raw field value.
2. Apply `RE2::PartialMatch` to verify the match.
3. Eliminate false positives from the coarse filter.

This is the same two-phase architecture already used by `LIKE` on ngram indexes.

**Literal extraction strategy**: A conservative parser walks the regex pattern character-by-character, collecting runs of non-metacharacter bytes. Escaped metacharacters (`\.`, `\*`) are treated as literal. Shorthand character classes (`\d`, `\w`) and other metacharacters are treated as split points. This approach is simple, correct, and does not depend on RE2 internals.

Example extractions:

| Regex Pattern | Extracted Literals |
|--------------|-------------------|
| `"hello"` | `["hello"]` |
| `"abc.*def"` | `["abc", "def"]` |
| `"user_[0-9]+@gmail\.com"` | `["user_", "@gmail", "com"]` |
| `"(?i)error"` | ngram skipped (case-insensitive, brute force) |
| `"[a-z]+"` | `[]` (no optimization, brute force) |
| `"\d{3}-\d{4}"` | `["-"]` |

### 4. Case-Insensitive Matching and Ngram

RE2 supports inline flags such as `(?i)` for case-insensitive matching. The expression `field =~ "(?i)error"` matches `"Error"`, `"ERROR"`, `"error"`, etc.

However, ngram indexes store tokens from the original (case-preserved) text. When a case-insensitive pattern is used, extracted literals like `"error"` may not match ngram terms like `"Err"` or `"ERR"` in the index. This causes the ngram coarse filter to produce false negatives — missing rows that should match.

**V1 approach**: When the regex pattern contains a case-insensitive flag (`(?i)`), ngram index optimization is skipped entirely. The query falls back to brute-force `RE2::PartialMatch` on raw data. This is functionally correct but slower.

**Future optimization (V2)**: Introduce a case-folding option for ngram indexes. When enabled, both the indexed text and query literals are lowercased before ngram tokenization, enabling ngram acceleration for case-insensitive patterns.

### 5. Regex to LIKE Optimization

Many real-world regex patterns are simple enough to be expressed as equivalent `LIKE` operations, which have dedicated fast paths (e.g., `memcmp` for prefix matching, `string::find` for substring matching) that are significantly faster than invoking the RE2 engine.

The parser detects such patterns and transparently downgrades them to the corresponding `LIKE` OpType:

| Regex Pattern | Equivalent | OpType |
|--------------|------------|--------|
| `"abc"` (pure literal, no metacharacters) | `LIKE "%abc%"` | `InnerMatch("abc")` |
| `"^abc"` (anchored start + literal) | `LIKE "abc%"` | `PrefixMatch("abc")` |
| `"abc$"` (literal + anchored end) | `LIKE "%abc"` | `PostfixMatch("abc")` |
| `"^abc$"` (both anchors + literal) | `== "abc"` | `Equal("abc")` |

The conversion is conservative: only patterns composed entirely of literal characters and `^`/`$` anchors are converted. Any regex metacharacter (`.`, `*`, `+`, `?`, `[`, `(`, `{`, `\d`, etc.) causes the pattern to remain as `RegexMatch`.

This optimization is applied in the Go parser layer (`VisitRegexMatch`), making it transparent to the execution engine and index layer.

### 6. Index Support

Regex filtering should work with all index types that support LIKE, using the same or equivalent mechanisms. The goal is: wherever LIKE works, `=~` should also work.

| Index Type | LIKE Support | Regex (`=~`) Strategy |
|------------|-------------|----------------------|
| **No index (brute force)** | RE2 FullMatch on raw data | RE2 PartialMatch on raw data |
| **Ngram index** | Two-phase: ngram filter + LIKE verify | Two-phase: ngram filter (literal extraction) + RE2 PartialMatch verify |
| **Inverted index (tantivy)** | PatternQuery (LIKE→regex→FST) | Convert `.` → `[\s\S]` for dot\_nl alignment, wrap with `[\s\S]*(?:...)[\s\S]*` for substring semantics, call `regex_query` on term dictionary. Iterates unique values via FST intersection. |
| **Sort index (StringIndexSort)** | PrefixMatch via range query | Iterate unique values with RE2 PartialMatch, union posting lists of matches. O(unique\_values). |
| **Marisa index (StringIndexMarisa)** | PatternMatch dispatch | Iterate unique trie keys with RE2 PartialMatch, union row offsets. O(unique\_values). |
| **Bitmap index** | PatternMatch/PatternQuery | Iterate unique keys with RE2 PartialMatch, union bitmaps. O(unique\_values). |

All index types that support `PatternMatch()` iterate unique values rather than per-row `Reverse_Lookup`, making regex on indexed fields O(unique\_values) instead of O(total\_rows).

For non-ngram indexes, the regex-to-LIKE optimization (Section 5) is important: simple patterns like `"^prefix"` are converted to `PrefixMatch` which can leverage sort index range queries and inverted index prefix queries natively.

### 7. Supported Field Types

Regex supports the same field types and access patterns as LIKE:

| Field Type | Syntax | Notes |
|------------|--------|-------|
| VARCHAR | `field =~ "pattern"` | Primary use case |
| JSON (string path) | `metadata["key"] =~ "pattern"` | Requires JSON path specifier. `JSONField =~ "..."` on the root node is accepted (consistent with LIKE) but only matches when the JSON value itself is a bare string — object/array roots return false. |
| Array\<VARCHAR\> (indexed) | `arr[0] =~ "pattern"` | Matches the element at the specified index, same as LIKE. Does NOT match "any element" — an explicit index is required. |
| JSON (array of strings) | N/A | Not supported, same as LIKE |
| INT/FLOAT/BOOL/other | N/A | Rejected at parse time |

**Note on Array\<VARCHAR\>**: The `=~` operator on arrays follows the same semantics as LIKE — the user must specify an element index (e.g., `arr[0]`). "Match if any element matches" is not supported in V1. This is consistent with the existing LIKE behavior.

---

## Test Plan

### Unit Tests (Go)
- Parser correctly produces `UnaryRangeExpr` with `OpType_RegexMatch` for `=~` expressions.
- Parser correctly produces `NOT(UnaryRangeExpr{RegexMatch})` for `!~` expressions.
- Parser rejects invalid regex syntax with descriptive error.
- Parser rejects `=~` on non-string fields.
- JSON and Array\<VARCHAR\> fields are handled correctly.
- Regex-to-LIKE optimization: `"abc"` → InnerMatch, `"^abc"` → PrefixMatch, `"abc$"` → PostfixMatch, `"^abc$"` → Equal.
- Patterns with metacharacters remain as RegexMatch (not converted to LIKE).

### Unit Tests (C++)
- `PartialRegexMatcher` correctly implements substring matching.
- `UnaryExpr` with `RegexMatch` produces correct bitmaps on raw data.
- Ngram index `extract_literals_from_regex` correctly extracts literals from various patterns.
- Ngram two-phase execution (Phase1 + Phase2) produces correct results for `RegexMatch`.
- Case-insensitive `(?i)` patterns bypass ngram optimization and still produce correct results via brute force.
- NULL field values produce NULL results (excluded from filter).
- Edge cases: empty pattern, empty string values, pattern with no extractable literals, Unicode patterns.

### Integration Tests
- End-to-end: create collection with VARCHAR field, insert data, query with `=~` and `!~`, verify results.
- With ngram index: create ngram index on field, verify regex queries use ngram acceleration.
- Case-insensitive: verify `(?i)` patterns return correct results with and without ngram index.
- Combined with vector search: hybrid search with regex filter.

---

## Relationship with LIKE

`LIKE` and `=~` are independent operators that coexist. `LIKE` is not deprecated.

| Aspect | `LIKE` | `=~` |
|--------|--------|------|
| Pattern language | SQL wildcards (`%`, `_`) | RE2 regex |
| Matching semantics | Full-string | Substring |
| Pattern decomposition | Yes (PrefixMatch, InnerMatch, etc.) | No (except regex-to-LIKE optimization) |
| Execution engine | Specialized matchers (memcmp, string::find) | RE2 |
| Index optimization | Ngram, inverted (PatternQuery), sort (prefix) | Ngram (primary), brute force (fallback) |

For simple patterns, `LIKE` is faster because it avoids the regex engine entirely. Users should prefer `LIKE` when the pattern can be expressed with `%` and `_` wildcards. `=~` is for patterns that require regex expressiveness.

---

## Rejected Alternatives

### 1. Full-string matching semantics (`^pattern$`)

Full-string matching would allow tantivy's FST+Automaton to be used for index acceleration. However, most real-world regex use cases are substring matching (finding patterns within larger strings). Requiring users to write `.*pattern.*` for every substring match would be unintuitive and error-prone. PostgreSQL, MySQL, and grep all default to substring matching.

### 2. Function syntax `REGEXP(field, pattern)`

A function-based syntax like `REGEXP(field, "pattern")` was considered. While it's more extensible (easy to add flags parameter), it doesn't integrate as naturally into the expression language. The `=~` operator syntax is more concise and consistent with PostgreSQL and Ruby conventions. The operator approach also fits better into the existing `UnaryRangeExpr` infrastructure used by `LIKE`.

### 3. Using tantivy's built-in regex for index acceleration

Tantivy's `RegexQuery` uses FST+Automaton intersection for regex matching on the term dictionary. This is efficient for full-string matching with selective patterns. However, for substring matching (our chosen semantics), every pattern becomes `.*pattern.*`, which means the automaton must traverse nearly all FST branches — no pruning benefit. The ngram index approach is more effective for substring patterns because it directly identifies documents containing the literal substrings present in the regex.

---

## TODO — Remaining Work for Feature Release

### P0 — Must Have

- [x] **Brute-force regex (no index)**: Raw data scan path verified via C++ UT on growing segments. RE2 PartialMatch on every row works correctly.
- [x] **Sealed segment verification**: Analysis confirmed sealed and growing segments share the same UnaryExpr execution path. Growing segment UT provides coverage.
- [x] **All index types support regex**: All index types that support LIKE now support `=~`:
  - Inverted index (tantivy): PatternMatch converts `.` → `[\s\S]`, wraps for substring semantics, calls `regex_query` on term dictionary
  - Sort index (StringIndexSort): unique-value iteration with PartialRegexMatcher (both Memory and Mmap impl)
  - Marisa index (StringIndexMarisa): unique-value iteration with PartialRegexMatcher
  - Bitmap index: unique-value iteration with PartialRegexMatcher (all three build modes)
  - Ngram index: two-phase literal extraction (with group penetration, quantifier expansion) + RE2 PartialMatch verification
- [ ] **E2E integration tests**: Python pymilvus end-to-end tests:
  - Create collection → insert → query with `=~` and `!~` → verify results
  - Test with no index, inverted index, ngram index configurations
  - Hybrid search: vector search combined with regex filter
  - JSON field: `metadata["key"] =~ "pattern"`
  - Array\<VARCHAR\> field: element-level regex matching
- [x] **Retrieve path**: Analysis confirmed query and search use the same expression executor. No separate handling needed.

### P1 — Important

- [x] **Proxy layer validation**: All OpType enumeration points in proxy/delegator code reviewed. Default/fallback behaviors are correct for RegexMatch (scalar pruning returns allTrue, PK filter returns unsupported, reverseOrder not applicable). No changes needed.
- [ ] **Template expression support**: Verify `field =~ {pattern}` works with parameterized queries
- [x] **SDK check**: Confirmed Python (pymilvus) and Go client pass expression strings through without client-side parsing or validation. No SDK changes needed.

### P2 — Before GA

- [ ] **User documentation**: Expression syntax reference — document `=~` and `!~` operators, supported regex syntax (RE2), matching semantics (substring), and known limitations
- [ ] **Known limitations documentation**:
  - `(?i)` case-insensitive skips ngram optimization (brute force fallback)
  - RE2 does not support backreferences, lookahead/lookbehind
  - Complex patterns on large datasets without ngram index will be slow

### P3 — Future Optimizations

- [ ] **Alternation OR splitting for ngram**: Currently `abc|xyz` bails out of ngram entirely. Optimization: split on top-level `|`, extract literals from each branch independently, query ngrams per branch, OR the candidate bitmaps. Requires changing `ExecutePhase1` from a single `vector<string>` (AND) to `vector<vector<string>>` (OR of ANDs). Ref: ClickHouse `analyzeImpl()` extracts `alternatives` and tests them individually.
- [ ] **Brute-force pre-filter (Volnitsky-style)**: When no ngram index exists, extract a required literal from the regex and use `string::find` to pre-filter rows before running RE2. ClickHouse uses the Volnitsky algorithm (bigram hash table, 64KB L2-cache-friendly, 10-100x faster than `strstr`) for this. Milvus can start with `string::find` and optionally adopt Volnitsky later for further speedup. This avoids running the RE2 engine on rows that obviously don't contain the required literal.
- [ ] **Hyperscan multi-pattern optimization**: When multiple regex filters target the same field (e.g., `field =~ "pattern_a" || field =~ "pattern_b"` or `field =~ "pattern_a" && field =~ "pattern_b"`), compile all patterns into a single Hyperscan DFA and scan each row once. O(n) regardless of pattern count, vs O(n×k) for k independent RE2 evaluations. Requires an expression optimization pass to detect multi-regex conjunctions/disjunctions on the same field and batch them. Ref: ClickHouse uses Hyperscan for `multiMatchAny()`.
- [x] ~~**Group penetration for literal extraction**~~: Implemented. `abc(de)fg` now extracts `"abcdefg"`. Non-optional groups are penetrated; optional groups (`?`, `*`, `{0,...}`) are skipped.

---

## Comparison with Other Systems

| Feature | **Milvus** | **PostgreSQL** | **ClickHouse** | **Elasticsearch** |
|---------|-----------|----------------|----------------|-------------------|
| **Regex Engine** | RE2 | Spencer/Tcl ARE | RE2 (+Hyperscan) | Lucene automaton |
| **Matching Semantics** | Substring | Substring | Substring | Full-string (anchored) |
| **Operators** | `=~`, `!~` | `~`, `~*`, `!~`, `!~*` | `match()`, `REGEXP` | `regexp` JSON query |
| **Negation** | `!~` | `!~`, `!~*` | `NOT match()` | `bool.must_not` |
| **Case-Insensitive** | `(?i)` flag | `~*` operator or `(?i)` | `(?i)` flag | `case_insensitive` param |
| **Backreferences** | No (RE2) | Yes | No (RE2) | No |
| **Lookahead/Lookbehind** | No (RE2) | Yes | No (RE2) | No |
| **Named Groups** | Yes | Yes | Yes | No |
| **Unicode `\p{}`** | Yes | Yes | Yes | No |
| **`\d` `\w` `\s`** | Yes | Yes | Yes | No |
| **Index Acceleration** | Ngram (two-phase) | pg_trgm GIN/GiST | ngrambf bloom skip index | Automaton × term dictionary |
| **Performance Guardrail** | RE2 linear time | None (use `statement_timeout`) | RE2 linear time | `max_determinized_states` |

**Design rationale:** Milvus's regex support is most similar to ClickHouse — both use RE2 with substring matching semantics. RE2 is the preferred choice for database systems because it guarantees linear-time execution (no catastrophic backtracking / ReDoS), at the cost of not supporting backreferences and lookahead/lookbehind. PostgreSQL's Spencer engine supports these features but has no built-in protection against pathological patterns. Elasticsearch's Lucene regex is the most limited (no `\d`/`\w`/`\s`, no named groups) and uses full-string matching semantics.

The ngram index acceleration approach is analogous to PostgreSQL's `pg_trgm` extension: extract literal substrings from the regex, look them up in the ngram/trigram index to narrow candidates, then verify with the full regex engine.

### Detailed Alignment with ClickHouse

Milvus's regex implementation is designed to align with ClickHouse's `match()` function semantics, since both use RE2 as the underlying engine. The following details are verified against ClickHouse source code (`src/Common/OptimizedRegularExpression.cpp`, `src/Functions/MatchImpl.h`).

#### RE2 Options

| Option | ClickHouse | Milvus | Aligned? |
|--------|-----------|--------|----------|
| `dot_nl` | ON by default (`.` matches `\n`; use `(?-s)` to disable) | ON | Yes |
| `log_errors` | OFF | OFF | Yes |
| `encoding` | UTF-8, with Latin-1 fallback on invalid UTF-8 | UTF-8 only | Acceptable difference |
| `case_sensitive` | ON by default; OFF via flag or `(?i)` | ON by default; OFF via `(?i)` | Yes |
| `max_mem` | Not explicitly set (RE2 default) | Not explicitly set | Yes |

#### Matching Semantics

| Behavior | ClickHouse | Milvus | Aligned? |
|----------|-----------|--------|----------|
| Anchoring | UNANCHORED (`RE2::Match` with `UNANCHORED`) | `RE2::PartialMatch` | Yes (equivalent) |
| `match('Hello', '')` | Returns 1 (empty pattern matches everything) | `PartialMatch` returns true | Yes |
| `match('x', '.*')` / `match('x', '.*?')` | Fast path: returns 1 without running RE2 | Runs RE2 (correct but no fast path) | Functionally equivalent |
| `(?-s)` flag | Disables dot_nl (`.` stops matching `\n`) | Disables dot_nl | Yes |

#### Regex-to-Literal Optimization

Both ClickHouse and Milvus optimize simple regex patterns to avoid the RE2 engine:

| Aspect | ClickHouse (`OptimizedRegularExpression::analyze`) | Milvus |
|--------|---------------------------------------------------|--------|
| **Trivial literal detection** | If pattern has no metacharacters → `is_trivial=true`, uses `strstr()` | `tryOptimizeRegexToLike`: pure literal → `InnerMatch` (uses `string::find`) |
| **Prefix detection** | Detects if required substring is at the start of the regex | `^literal` → `PrefixMatch` |
| **Alternation handling** | Extracts list of alternative substrings for multi-pattern matching | Returns empty (bails out on `\|`) — more conservative |
| **Required substring extraction** | Extracts longest required literal for pre-filtering with Volnitsky search | Extracts all required literals for ngram AND-filtering |

ClickHouse's `OptimizedRegularExpression` is more sophisticated in handling alternation — it can extract alternative substrings and test them individually. Milvus's `extract_literals_from_regex` takes a more conservative approach: any pattern containing alternation (`|`) at any nesting level skips ngram optimization entirely, falling back to brute-force RE2. This is safe (no false negatives) but may be less efficient for alternation-heavy patterns.

#### Differences and Rationale

| Difference | ClickHouse | Milvus | Rationale |
|-----------|-----------|--------|-----------|
| **UTF-8 fallback** | Falls back to Latin-1 if UTF-8 parsing fails | UTF-8 only, fails on invalid UTF-8 | Milvus VARCHAR fields are expected to be valid UTF-8. Binary data is not a target use case. |
| **Fast path for `.*`** | Special-cased to return all-true without running RE2 | Runs RE2 (returns true for all inputs) | Correctness equivalent. Could be added as a performance optimization later. |
| **Alternation optimization** | Extracts alternative substrings, tests each with Volnitsky | Bails out, falls back to brute-force RE2 | Conservative approach avoids false negatives. ClickHouse's approach is more complex but more efficient for `abc\|xyz`-style patterns. |
| **String searcher** | Volnitsky algorithm (SIMD-optimized) for substring pre-filter | `string::find` (via InnerMatch) or ngram index | Different optimization strategies suited to different architectures. |
| **Capturing groups** | Supports `extract()`, `extractAll()`, `extractGroups()` with up to 1024 groups | Not supported (filter-only, no extraction) | Milvus is a vector database; regex is used for filtering, not data extraction. |

#### Verified Edge Case Alignment

The following edge cases were tested against ClickHouse's documented behavior and confirmed to produce identical results in Milvus:

| Pattern | Input | ClickHouse `match()` | Milvus `=~` | Aligned? |
|---------|-------|---------------------|-------------|----------|
| `""` (empty) | `"Hello"` | 1 | true | Yes |
| `".*"` | `""` | 1 | true | Yes |
| `"c.d"` | `"abc\ndef"` | 1 (dot_nl=true) | true | Yes |
| `"(?-s)c.d"` | `"abc\ndef"` | 0 (dot_nl disabled) | false | Yes |
| `"abc\|xyz"` | `"only_abc"` | 1 | true | Yes |
| `"abc\|xyz"` | `"only_xyz"` | 1 | true | Yes |
| `"abc(de)?fg"` | `"abcfg"` | 1 | true | Yes |
| `"abc(de)?fg"` | `"abcdefg"` | 1 | true | Yes |
| `"(?i)hello"` | `"HELLO"` | 1 | true | Yes |
| `"\\bhello\\b"` | `"say hello world"` | 1 | true | Yes |
| `"\\p{Han}+"` | `"中文"` | 1 | true | Yes |
| `"\\d{3}-\\d{4}"` | `"555-1234"` | 1 | true | Yes |
| `"(a)\\1"` | any | Error (RE2 rejects) | Error (RE2 rejects) | Yes |

---

## References

- [RE2 Syntax Reference](https://github.com/google/re2/wiki/Syntax)
- [RE2 Safety Guarantees](https://swtch.com/~rsc/regexp/regexp1.html) — linear-time matching, no catastrophic backtracking
- [Google Code Search](https://swtch.com/~rsc/regexp/regexp4.html) — trigram index for regex acceleration (same approach as our ngram optimization)
- [PostgreSQL Pattern Matching](https://www.postgresql.org/docs/current/functions-matching.html) — `~` operator with substring semantics

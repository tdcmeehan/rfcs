# Deep Dive Review: RFC-0022 JVector Integration

## Summary

This review analyzes the SPI design and abstraction boundaries in the proposed JVector integration RFC. The RFC proposes adding Approximate Nearest Neighbor (ANN) search capabilities to Presto via JVector, but the current design has significant concerns around abstraction layers and the SPI/engine boundary.

---

## Executive Summary of Concerns

| Area | Concern | Severity |
|------|---------|----------|
| **Existing SPI** | Does not reference existing index join SPI pattern | High |
| SPI Boundary | Tight coupling to Iceberg connector | High |
| Abstraction | `VectorIndexManager` conflates concerns | Medium |
| API Design | Stored procedure on `system` catalog operates on external tables | High |
| Extensibility | No clear connector SPI for vector capabilities | High |
| TVF Design | Column reference includes full table path | Medium |

---

## 1. Overlap with Existing Index Join SPI

Presto already has an index join SPI that the RFC does not reference. This is a significant oversight—the RFC should either extend this existing pattern or explicitly justify why a different approach is needed.

### Existing Index Join SPI Components

```java
// Marker interface for index handles (opaque to engine)
public interface ConnectorIndexHandle { }

// Result of index resolution during planning
public class ConnectorResolvedIndex {
    ConnectorIndexHandle indexHandle;
    TupleDomain<ColumnHandle> unresolvedTupleDomain;
}

// Connector advertises index support via metadata
public interface ConnectorMetadata {
    // Called during planning to find applicable indexes
    default Optional<ConnectorResolvedIndex> resolveIndex(
        ConnectorSession session,
        ConnectorTableHandle tableHandle,
        Set<ColumnHandle> indexableColumns,    // Columns to lookup by
        Set<ColumnHandle> outputColumns,       // Columns to return
        TupleDomain<ColumnHandle> tupleDomain  // Filter predicates
    ) {
        return Optional.empty();
    }
}

// Connector provides index implementation
public interface Connector {
    default ConnectorIndexProvider getIndexProvider() {
        throw new UnsupportedOperationException();
    }
}

// Factory for obtaining index instances
public interface ConnectorIndexProvider {
    ConnectorIndex getIndex(
        ConnectorTransactionHandle transactionHandle,
        ConnectorSession session,
        ConnectorIndexHandle indexHandle,
        List<ColumnHandle> lookupSchema,   // Keys to lookup
        List<ColumnHandle> outputSchema    // Columns to return
    );
}

// The actual index implementation
public interface ConnectorIndex {
    // Given a RecordSet of lookup keys, return matching rows
    ConnectorPageSource lookup(RecordSet recordSet);
}
```

### How Existing Index Joins Work

1. **Planning Phase**: Engine calls `resolveIndex()` to ask if connector has an index for the join columns
2. **Resolution**: Connector returns `ConnectorResolvedIndex` with opaque handle + any predicates it couldn't push
3. **Execution Phase**: Engine calls `getIndex()` to obtain `ConnectorIndex` instance
4. **Lookup**: For each batch of probe-side keys, engine calls `index.lookup(recordSet)` to get matching rows

### Comparison: Existing Index Join vs. Proposed Vector Search

| Aspect | Existing Index Join | Proposed Vector Search |
|--------|--------------------|-----------------------|
| **Lookup semantic** | Exact match on key columns | Approximate nearest neighbors |
| **Result cardinality** | Variable (0 to many per key) | Fixed K per query vector |
| **Resolution** | `resolveIndex()` in planning | Not specified (VectorIndexManager?) |
| **Handle pattern** | `ConnectorIndexHandle` (opaque) | Not specified |
| **Execution** | `ConnectorIndex.lookup(RecordSet)` | TVF with unclear execution model |
| **Integration** | Transparent to SQL (join optimization) | Explicit TVF call required |

### Key Observation: Different Semantics, Similar Pattern

Vector search has fundamentally different semantics than exact-match index lookups:
- **Input**: Single query vector (not a batch of keys)
- **Output**: Top-K nearest neighbors with scores
- **Algorithm**: ANN search (not B-tree/hash lookup)

However, the **SPI pattern** could be similar:

```java
// Extend the existing pattern for vector indexes
public interface ConnectorMetadata {
    // Existing method
    Optional<ConnectorResolvedIndex> resolveIndex(...);

    // NEW: Resolve vector index for ANN search
    default Optional<ConnectorResolvedVectorIndex> resolveVectorIndex(
        ConnectorSession session,
        ConnectorTableHandle tableHandle,
        ColumnHandle vectorColumn,
        SimilarityFunction similarityFunction
    ) {
        return Optional.empty();
    }
}

// NEW: Vector index handle (parallel to ConnectorIndexHandle)
public interface ConnectorVectorIndexHandle { }

// NEW: Resolution result (parallel to ConnectorResolvedIndex)
public class ConnectorResolvedVectorIndex {
    ConnectorVectorIndexHandle indexHandle;
    VectorIndexMetadata metadata;  // dimension, similarity function, etc.
}

// NEW: Vector index provider (parallel to ConnectorIndexProvider)
public interface ConnectorVectorIndexProvider {
    ConnectorVectorIndex getVectorIndex(
        ConnectorTransactionHandle transactionHandle,
        ConnectorSession session,
        ConnectorVectorIndexHandle indexHandle
    );
}

// NEW: Vector index execution (parallel to ConnectorIndex)
public interface ConnectorVectorIndex {
    // Perform ANN search, return top-K with scores
    VectorSearchResult search(
        float[] queryVector,
        int k,
        SearchParameters params  // ef_search, etc.
    );
}
```

### Recommendation: Align with Existing Pattern

The RFC should:

1. **Reference the existing index join SPI** and explain why it doesn't fit vector search (different semantics)

2. **Follow the same structural pattern**:
   - Resolution in `ConnectorMetadata` (planning phase)
   - Opaque handles (`ConnectorVectorIndexHandle`)
   - Provider interface (`ConnectorVectorIndexProvider`)
   - Execution interface (`ConnectorVectorIndex`)

3. **Consider whether vector search could use index join infrastructure** for the final join-back to base table data (after getting row IDs from ANN search)

4. **Address lifecycle differences**: Existing index join assumes indexes are pre-existing; vector indexes need explicit creation DDL

### Why This Matters

If the RFC introduces a completely parallel set of abstractions without acknowledging the existing pattern, it:
- Creates inconsistency in Presto's SPI design
- Misses opportunity to reuse infrastructure (split handling, caching, etc.)
- Confuses connector developers who must learn two different patterns

---

## 2. SPI vs Engine Boundary Analysis

### What Should Be in the SPI (Connector Layer)

The SPI should define **contracts** that allow connectors to advertise and provide vector index capabilities. The RFC currently lacks this abstraction layer.

#### Recommended SPI Interfaces

```java
/**
 * SPI interface for connectors that support vector indexes.
 * This should live in presto-spi.
 */
public interface ConnectorVectorIndexProvider {

    /**
     * Returns capabilities of this connector's vector index support.
     */
    VectorIndexCapabilities getCapabilities();

    /**
     * Creates a vector index on the specified column.
     * The connector owns index storage and lifecycle.
     */
    void createVectorIndex(
        ConnectorSession session,
        ConnectorTableHandle table,
        String columnName,
        VectorIndexProperties properties);

    /**
     * Drops a vector index.
     */
    void dropVectorIndex(
        ConnectorSession session,
        String indexName);

    /**
     * Returns metadata for available vector indexes on a table.
     */
    List<VectorIndexMetadata> getVectorIndexes(
        ConnectorSession session,
        ConnectorTableHandle table);

    /**
     * Creates splits for vector search execution.
     * Connector knows how to partition the index.
     */
    List<ConnectorSplit> getVectorSearchSplits(
        ConnectorSession session,
        VectorIndexHandle indexHandle,
        VectorSearchConstraint constraint);
}
```

```java
/**
 * Capabilities advertised by a connector for vector operations.
 */
public interface VectorIndexCapabilities {

    Set<SimilarityFunction> getSupportedSimilarityFunctions();

    Set<VectorIndexType> getSupportedIndexTypes();

    boolean supportsPartitionedIndexes();

    boolean supportsIncrementalIndexUpdates();

    int getMaxVectorDimension();

    Optional<Long> getMaxIndexSize();
}
```

```java
/**
 * Handle to a vector index, opaque to the engine.
 */
public interface VectorIndexHandle {
    // Marker interface - connector-specific implementation
}
```

#### Current RFC Gap

The RFC describes `VectorIndexManager` as handling lifecycle operations, but it's unclear whether this lives in the engine or connector. The mention of "Iceberg snapshot summaries" for metadata storage suggests tight coupling to a specific connector.

**Recommendation:** Index metadata storage should be a connector responsibility, not hardcoded to Iceberg.

### What Should Be in the Engine

The engine should handle:

1. **Query Planning & Optimization**
   - Recognizing vector search patterns
   - Partition pruning based on WHERE clauses
   - Cost estimation for vector search vs. brute force
   - Plan optimization (when to use index vs. scan)

2. **Distributed Execution Coordination**
   - Split distribution to workers
   - Result aggregation (K-way merge)
   - Resource management (memory, concurrent searches)

3. **Table-Valued Function Framework**
   - Registration and invocation of `approx_nearest_neighbors`
   - Parameter validation
   - Result schema definition

4. **Caching Infrastructure**
   - Worker-level index caching (L1/L2)
   - Cache eviction policies
   - Memory management

---

## 3. Abstraction Layer Problems

### Problem 1: Monolithic VectorIndexManager

The RFC describes `VectorIndexManager` with these responsibilities:
- Index creation
- Metadata retrieval
- Index listing
- Deletion
- Refresh/rebuild

This conflates **connector concerns** (storage, metadata) with **engine concerns** (caching, coordination).

**Recommendation:** Split into:

```
┌─────────────────────────────────────────────────────────┐
│                    Engine Layer                          │
├─────────────────────────────────────────────────────────┤
│  VectorSearchCoordinator                                 │
│  - Query planning                                        │
│  - Split generation (delegates to connector)             │
│  - Result aggregation                                    │
│  - Cache coordination                                    │
└─────────────────────────────────────────────────────────┘
                           │
                           │ SPI Boundary
                           ▼
┌─────────────────────────────────────────────────────────┐
│                    Connector Layer                       │
├─────────────────────────────────────────────────────────┤
│  ConnectorVectorIndexProvider (per connector)            │
│  - Index creation/deletion                               │
│  - Metadata storage (connector-specific)                 │
│  - Split generation for vector search                    │
│  - Index file management                                 │
└─────────────────────────────────────────────────────────┘
```

### Problem 2: VectorIndexMetadata Abstraction

The RFC's `VectorIndexMetadata` contains:
- Index name, table reference, column name
- Similarity function, vector dimension
- HNSW parameters (M, ef_construction)
- **Partition-to-path mappings**
- Creation timestamps, status

The "partition-to-path mappings" is problematic—this is storage-layer detail leaking into what should be a clean metadata interface.

**Recommendation:** Separate concerns:

```java
/**
 * SPI-level metadata (connector-agnostic)
 */
public interface VectorIndexMetadata {
    String getIndexName();
    SchemaTableName getTableName();
    String getColumnName();
    SimilarityFunction getSimilarityFunction();
    int getVectorDimension();
    VectorIndexType getIndexType();
    Map<String, String> getIndexParameters(); // e.g., M, ef_construction
    Instant getCreatedAt();
    VectorIndexStatus getStatus();
}

/**
 * Connector-internal: Iceberg-specific implementation
 * This does NOT go in SPI
 */
class IcebergVectorIndexMetadata implements VectorIndexMetadata {
    // ... base fields ...

    // Iceberg-specific
    private final Map<PartitionSpec, String> partitionToIndexPath;
    private final long snapshotId;
}
```

### Problem 3: Stored Procedure Design

The RFC proposes:
```sql
CALL system.create_vector_index(
    table_name => 'catalog.schema.table',
    ...
);
```

**Issues:**
1. The `system` catalog shouldn't manage indexes for other catalogs
2. This bypasses the connector abstraction entirely
3. Cross-catalog operations have authorization implications

**Recommendation:** Use connector-scoped procedures:
```sql
CALL iceberg.system.create_vector_index(
    table_name => 'schema.table',
    ...
);
```

Or better, use DDL syntax:
```sql
CREATE VECTOR INDEX idx_name
ON catalog.schema.table(embedding)
USING HNSW
WITH (similarity_function = 'COSINE', m = 16);
```

This integrates with Presto's existing DDL handling and authorization framework.

---

## 4. TVF Design Analysis

### Current Proposal
```sql
SELECT * FROM TABLE(approx_nearest_neighbors(
    query_vector => ARRAY[0.1, 0.2, ...],
    column_name => 'catalog.schema.documents.embedding',
    limit => 10
)) ann
```

### Concerns

1. **Column reference as string**: The `column_name` parameter includes the full catalog.schema.table.column path as a string. This:
   - Bypasses SQL's normal name resolution
   - Doesn't integrate with Presto's authorization
   - Makes it unclear how JOINs work

2. **No explicit index reference**: The TVF doesn't specify which index to use if multiple exist on the same column.

3. **Result schema unclear**: What columns does the TVF return? Just `row_id` and `score`?

### Recommended TVF Design

```sql
-- Option A: Implicit index selection (engine picks best index)
SELECT d.*, ann.distance
FROM documents d,
     TABLE(approx_nearest_neighbors(
         d.embedding,                    -- Column reference, not string
         ARRAY[0.1, 0.2, ...],          -- Query vector
         10                              -- K
     )) ann;

-- Option B: Explicit index reference
SELECT d.*, ann.distance
FROM documents d,
     TABLE(approx_nearest_neighbors(
         INDEX idx_embedding,            -- Explicit index
         ARRAY[0.1, 0.2, ...],
         10
     )) ann;
```

The lateral join pattern (Option A) allows proper column resolution and authorization.

---

## 5. Split Design

### Current Design (Inferred)

The RFC mentions `BuildIndexSplit` and `VectorSearchSplit` but doesn't detail their structure.

### Recommended Split Abstraction

```java
/**
 * SPI interface for vector search splits.
 * Connectors create these; engine distributes them.
 */
public interface VectorSearchSplit extends ConnectorSplit {

    /**
     * Opaque handle to the index partition to search.
     */
    VectorIndexPartitionHandle getPartitionHandle();

    /**
     * Estimated number of vectors in this split.
     */
    long getEstimatedVectorCount();
}
```

The engine should NOT know about:
- File paths
- Partition key values
- Storage format details

These belong in connector-specific split implementations.

---

## 6. Caching Architecture

### RFC Proposal
- L1: Memory cache on workers
- L2: Local disk cache on workers

### Concern

The caching strategy is engine-level, but what gets cached is connector-specific (index file format, memory mapping strategy).

### Recommendation

```java
/**
 * SPI interface for index loading.
 * Connector provides; engine manages cache.
 */
public interface VectorIndexLoader {

    /**
     * Load an index into memory for searching.
     * @return Handle that can be used for searches
     */
    LoadedVectorIndex loadIndex(
        VectorIndexPartitionHandle partition,
        IndexLoadContext context);

    /**
     * Estimate memory footprint of loaded index.
     */
    DataSize estimateMemoryUsage(VectorIndexPartitionHandle partition);
}

/**
 * Returned by connector, used by engine for search execution.
 */
public interface LoadedVectorIndex extends Closeable {

    /**
     * Execute ANN search on this index.
     */
    VectorSearchResults search(
        float[] queryVector,
        int k,
        SearchParameters params);
}
```

This keeps the engine in control of caching policy while connectors control loading mechanics.

---

## 7. Specific Recommendations

### High Priority

1. **Define a proper SPI interface** (`ConnectorVectorIndexProvider`) that connectors implement. This is the most critical gap.

2. **Move index metadata storage to connectors**. The engine should only see the `VectorIndexMetadata` interface, not Iceberg snapshots.

3. **Redesign the stored procedure** to be connector-scoped, or use DDL syntax.

4. **Fix the TVF column reference** to use proper SQL column resolution, not string paths.

### Medium Priority

5. **Separate `VectorIndexManager`** into engine coordination and connector implementation pieces.

6. **Define split interfaces** that hide storage details from the engine.

7. **Abstract the caching layer** so connectors provide loaders and the engine manages cache.

### Lower Priority

8. **Consider supporting multiple index types** (not just HNSW) in the SPI.

9. **Add cost estimation hooks** so the optimizer can choose between index search and brute force.

10. **Define monitoring/metrics interfaces** in the SPI for observability.

---

## 8. Architectural Diagram (Recommended)

```
┌────────────────────────────────────────────────────────────────────┐
│                         Presto Engine                               │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │  SQL Parser     │  │  Query Planner   │  │  TVF Framework   │  │
│  │  (DDL support)  │  │  (cost-based)    │  │  (ANN function)  │  │
│  └────────┬────────┘  └────────┬─────────┘  └────────┬─────────┘  │
│           │                    │                     │             │
│           ▼                    ▼                     ▼             │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │              VectorSearchCoordinator                         │  │
│  │  - Metadata resolution (via SPI)                             │  │
│  │  - Split generation (delegates to connector)                 │  │
│  │  - Result aggregation                                        │  │
│  └──────────────────────────┬──────────────────────────────────┘  │
│                              │                                     │
│  ┌───────────────────────────┼───────────────────────────────────┐│
│  │         Worker Execution  │                                   ││
│  │  ┌────────────────────────▼─────────────────────────────────┐ ││
│  │  │              IndexCacheManager                            │ ││
│  │  │  - L1 (memory) / L2 (disk) caching                       │ ││
│  │  │  - Eviction policy                                        │ ││
│  │  └──────────────────────────────────────────────────────────┘ ││
│  └───────────────────────────────────────────────────────────────┘│
│                              │                                     │
└──────────────────────────────┼─────────────────────────────────────┘
                               │ SPI Boundary
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Connector SPI                                  │
├──────────────────────────────────────────────────────────────────┤
│  ConnectorVectorIndexProvider                                     │
│  ├── getCapabilities()                                            │
│  ├── createVectorIndex(...)                                       │
│  ├── dropVectorIndex(...)                                         │
│  ├── getVectorIndexes(...)                                        │
│  └── getVectorSearchSplits(...)                                   │
│                                                                    │
│  VectorIndexLoader                                                 │
│  ├── loadIndex(...)                                               │
│  └── estimateMemoryUsage(...)                                     │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│              Connector Implementations                            │
├─────────────────────┬────────────────────┬───────────────────────┤
│  Iceberg Connector  │  Hive Connector    │  Delta Connector      │
│  ┌───────────────┐  │  ┌──────────────┐  │  ┌─────────────────┐  │
│  │ Iceberg       │  │  │ Hive         │  │  │ Delta           │  │
│  │ VectorIndex   │  │  │ VectorIndex  │  │  │ VectorIndex     │  │
│  │ Provider      │  │  │ Provider     │  │  │ Provider        │  │
│  └───────────────┘  │  └──────────────┘  │  └─────────────────┘  │
│  - Snapshot meta    │  - Metastore       │  - Delta log          │
│  - Partition-aware  │  - Partition-aware │  - Partition-aware    │
│  - JVector impl     │  - JVector impl    │  - JVector impl       │
└─────────────────────┴────────────────────┴───────────────────────┘
```

---

## 9. Open Questions for RFC Authors

1. **Why not extend the existing index join SPI?** Presto already has `ConnectorIndex`, `ConnectorIndexHandle`, `ConnectorIndexProvider`, and `resolveIndex()`. The RFC should explain why this pattern doesn't work for vector search, or propose extensions to it.

2. **Why is index metadata stored in Iceberg snapshots specifically?** This seems to preclude other connectors from supporting vector indexes.

3. **How does authorization work?** If `system.create_vector_index` operates on arbitrary tables, what are the permission requirements?

4. **What happens when the underlying table is modified?** Is index invalidation automatic? How does this interact with Iceberg's snapshot model?

5. **How are concurrent index builds handled?** Multiple users building indexes on the same table?

6. **What is the plan for supporting connectors beyond Iceberg?** The RFC should clarify if this is Iceberg-specific or a general framework.

7. **How does the TVF interact with predicate pushdown?** Can WHERE clauses on the outer query be pushed through the ANN search?

---

## 10. Conclusion

The RFC proposes valuable functionality, but the current design lacks proper abstraction layers. The tight coupling to Iceberg and the absence of a clear SPI will:

1. **Limit adoption** to only Iceberg tables
2. **Create maintenance burden** when supporting other connectors
3. **Risk architectural debt** that's expensive to fix later

**Primary recommendation:** Before implementation, define a proper `ConnectorVectorIndexProvider` SPI that allows any connector to support vector indexes. This is foundational and should not be deferred.

The JVector integration itself (the actual ANN algorithm) can remain an implementation detail that connectors use internally, but the interface between engine and connectors must be clean and connector-agnostic.

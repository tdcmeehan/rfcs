# Vector Similarity Search in Presto: Design Analysis and Alternative Approach

## Executive Summary

This document summarizes an analysis of RFC-0022 (JVector Integration) for vector similarity search in Presto, identifies key concerns, and proposes an alternative approach using Lance format with IVF indexing that could provide architectural advantages for distributed query execution.

---

## Part 1: RFC-0022 Analysis (JVector/HNSW Approach)

### Overview

RFC-0022 proposes integrating JVector, a pure-Java HNSW (Hierarchical Navigable Small World) library, into Presto's Iceberg connector for Approximate Nearest Neighbor (ANN) search.

**Key Components:**
- `VectorIndexCache`: Worker-side caching of loaded indexes
- `PartitionAwareIndexManager`: Manages partition-level indexes
- `TableFunctionSplitProcessor`: Distributes search work to workers
- `TopKAggregationOperator`: Coordinator-side result merging
- `mapping.bin`: Node-to-row-ID mapping files

### Architecture

```
Index Building (Distributed Procedure):
  Coordinator → generates splits per partition
  Workers → build HNSW indexes in parallel
  Indexes stored in S3 → metadata in Iceberg snapshot summary

Query Execution:
  Coordinator → resolves index metadata, generates splits
  Workers → load indexes, execute ANN search, return top-K per partition
  Coordinator → merges results via TopKAggregationOperator
```

### SQL Interface

```sql
-- Index creation
CALL system.create_vector_index(
    table_name => 'catalog.schema.table',
    column_name => 'embedding',
    index_name => 'idx_name',
    similarity_function => 'COSINE'
);

-- Vector search (returns row_id + score only)
SELECT d.*, ann.score
FROM documents d
JOIN TABLE(approx_nearest_neighbors(
    query_vector => ARRAY[...],
    column_name => 'documents.embedding',
    limit => 10
)) ann ON d.row_id = ann.row_id;
```

**Note:** User must write explicit JOIN to get full row data.

---

## Part 2: Identified Concerns

### 2.1 Row ID Stability (Critical)

The RFC assumes "row IDs" are stable, but this varies by Iceberg table format:

| Table Format | Row ID Status |
|--------------|---------------|
| V1 | No row ID concept |
| V2 | Row IDs exist but are NOT stable (positional, break on compaction) |
| V3 | Stable row IDs via row lineage |

**Impact:** On V1/V2 tables, any compaction or Copy-on-Write operation invalidates the index's row-ID mappings, causing stale or incorrect results.

**Recommendation:** RFC should explicitly require V3 table format, or document this as a significant limitation.

### 2.2 TopKAggregationOperator (Core Engine Change)

The RFC claims changes are "connector-only," but `TopKAggregationOperator` is a new Presto operator requiring:
- New `PlanNode` subclass
- Planner rules to insert it
- `LocalExecutionPlanner` changes
- Optimizer integration

**Questions not addressed:**
- How does the planner know to insert this operator?
- What does the corresponding PlanNode look like?
- How does the optimizer reason about it?

### 2.3 User-Facing JOIN Requirement

The `approx_nearest_neighbors` TVF returns only `(row_id, score)`, requiring users to:
1. Understand the row_id mechanism
2. Write explicit JOINs to get full row data

This leaks implementation details to users.

### 2.4 Metadata Storage

The RFC uses Iceberg snapshot summaries for index metadata. Consider:
- Snapshot summaries have size constraints
- Puffin files are Iceberg's standard format for auxiliary metadata/indexes
- Multiple Puffin files (one per partition) would integrate better with Iceberg's architecture

### 2.5 No Coordinator-Level Pruning

With HNSW:
- Workers must load full partition indexes
- No ability to prune before distributing work
- All partitions searched, results merged afterward

---

## Part 3: Alternative Approach (Lance + IVF)

### Core Idea

Use Lance format with IVF (Inverted File Index) instead of JVector/HNSW to enable:
1. Coordinator-level centroid pruning
2. Disk-based indexing (memory-mapped)
3. Encapsulated TVF (no user-facing JOIN)
4. Integration with Iceberg V3 row lineage

### Why Lance?

| Aspect | JVector (HNSW) | Lance (IVF) |
|--------|----------------|-------------|
| Index location | Workers (must load full index) | Coordinator (centroids) + Workers (clusters) |
| Memory model | In-memory graph | Disk-based, memory-mapped |
| Pruning | None at coordinator | Centroid comparison prunes 97-99% |
| Parallelization | HNSW traversal is sequential | IVF clusters searched independently |
| Java integration | Pure Java | Rust + JNI (bindings exist) |

### IVF vs HNSW Trade-offs

**HNSW Strengths:**
- Higher recall, especially in high dimensions
- Supports dynamic updates
- Better for smaller datasets that fit in memory

**IVF Strengths:**
- Faster index builds
- Lower memory usage (disk-based)
- Enables coordinator-level pruning
- Better for large-scale datasets
- Easier to parallelize (clusters are independent)

**Hybrid Option:** IVF for coarse search + HNSW within clusters (Lance supports IVF_HNSW_PQ)

### Memory/Scale Analysis

For 768-dimensional float32 vectors:

| Vectors | Raw Data | HNSW Index | IVF Centroids (1000 clusters) |
|---------|----------|------------|-------------------------------|
| 1M | 3 GB | 5-6 GB | 3 MB |
| 10M | 30 GB | 50-60 GB | 3 MB |
| 100M | 300 GB | 500 GB | 3 MB |

**After IVF pruning to 1% (10 clusters):**
- 100M vectors → search only 1M vectors
- With Product Quantization: ~24 MB compressed index
- Disk-based access via mmap: minimal memory footprint

**Key Insight:** Lance can potentially handle 100M+ vectors on coordinator alone due to:
- Small centroid metadata in memory
- Disk-based search via memory-mapped files
- SSD latency is the bottleneck, not RAM

### Lance Disk-Based Architecture

Lance uses memory-mapped files (Arrow format):
- Vectors stored on disk, accessed via mmap
- OS pages data on demand
- Query directly from disk at near-memory speeds
- Benchmarks: <20ms query latency on 1M vectors, ~5ms with tuning

For Java/Presto integration:
- JNI call to Lance (Rust)
- Memory mapping happens in Rust, not Java
- Java coordinator receives results, doesn't manage mmap
- JNI overhead negligible relative to search time

---

## Part 4: Proposed Architecture

### Mirrored Partition Structure

```
Iceberg (source of truth):
  s3://bucket/iceberg/documents/
    ├── date=2024-01-01/
    │     └── *.parquet (full row data)
    ├── date=2024-01-02/
    │     └── *.parquet

Lance (vector index, mirrors Iceberg partitions):
  s3://bucket/lance/documents_vectors/
    ├── date=2024-01-01/
    │     └── Lance dataset (vectors + _row_id + IVF index)
    ├── date=2024-01-02/
    │     └── Lance dataset (vectors + _row_id + IVF index)
```

### Data Flow

```
Iceberg Table                    Lance Index
┌─────────────────┐             ┌─────────────────┐
│ _row_id (V3)    │────────────▶│ _row_id         │
│ doc_id          │             │ embedding vector│
│ title           │             │ IVF index       │
│ content         │             └─────────────────┘
│ embedding       │
└─────────────────┘

External embedding pipeline generates vectors,
writes to Lance with _row_id reference back to Iceberg.
```

### Query Execution Flow

```sql
-- User writes (no JOIN required):
SELECT * FROM TABLE(approx_nearest_neighbors(
    table => 'documents',
    query_vector => ARRAY[...],
    limit => 10
)) WHERE date = '2024-01-01'
```

**Execution:**
```
1. Partition Pruning (Iceberg):
   - WHERE date = '2024-01-01' → single partition

2. Centroid Pruning (Coordinator):
   - Load Lance centroids for partition (~MB)
   - Compare query vector to centroids
   - Identify relevant clusters (e.g., 10 of 1000)

3. ANN Search (Coordinator or Workers):
   - If small enough: coordinator searches via JNI to Lance
   - If large: distribute cluster searches to workers

4. Row ID Extraction:
   - Lance returns _row_id + score for top-K

5. Iceberg Fetch (Plan Rewrite):
   - Generate: SELECT * FROM documents WHERE _row_id IN (...)
   - Iceberg resolves _row_id → physical file locations
   - Deletion vectors filter stale entries

6. Return Results:
   - Full rows with scores, no user-facing JOIN
```

### Index Maintenance via Distributed Procedure

```sql
-- Build index (mirrors Iceberg partitions)
CALL system.build_vector_index(
    source_table => 'iceberg.documents',
    vector_column => 'embedding',
    target_location => 's3://bucket/lance/documents_vectors/',
    partition_columns => ['date']
);

-- Rebuild specific partition
CALL system.rebuild_vector_index(
    index => 'documents_vectors',
    partition => 'date=2024-01-01'
);
```

**Distributed Build:**
```
Coordinator:
  - Read Iceberg partition metadata
  - Generate splits (one per partition)

Workers (parallel):
  - Read vectors + _row_id from Iceberg partition
  - Build Lance IVF index
  - Write to mirrored location
```

### Handling Updates with Deletion Vectors

When Iceberg data changes:
1. Row deleted → deletion vector marks _row_id as deleted
2. Lance index still has stale entry
3. Query time: Lance returns candidate, Iceberg filters via deletion vector
4. Correctness maintained without immediate index rebuild

Periodic maintenance rebuilds Lance index to remove stale entries.

---

## Part 5: Integration Points to Research

### 5.1 Plan Rewriting

How to transform TVF into filtered Iceberg scan:

**Options:**
- Connector optimizer rule
- Split generation phase (after planning, before execution)
- TVF internally creates Iceberg scan

**Key question:** Can we inject row_id filter during split generation?

### 5.2 Index Join SPI

Presto has existing index join infrastructure:

```java
interface ConnectorIndex {
    ConnectorPageSource lookup(RecordSet recordSet);
}

interface ConnectorMetadata {
    Optional<ConnectorResolvedIndex> resolveIndex(...);
}
```

**Could this fit?**
- Input: query vector (as RecordSet)
- Output: matching rows (PageSource)
- Index implementation internally does Lance search + Iceberg fetch

### 5.3 TVF Implementation

Research needed:
- Can TVF take table reference as parameter?
- How does TVF interact with partition pruning?
- Can TVF produce dynamic filters for downstream scans?

### 5.4 Lance JNI Integration

Lance provides `lance-jni` crate:
- Rust core with Java bindings
- Native library: `liblancedb_jni.{so|dylib|dll}`
- Loaded via `jar-jni` library

Research needed:
- Performance characteristics of JNI calls
- Thread safety for coordinator-level usage
- Error handling across JNI boundary

### 5.5 Iceberg V3 Row Lineage

Questions:
- Does Presto's Iceberg connector support V3 row lineage?
- Can we filter by `_row_id` efficiently?
- How do deletion vectors integrate with row_id lookups?

---

## Part 6: Comparison Summary

| Aspect | RFC (JVector/HNSW) | Alternative (Lance/IVF) |
|--------|-------------------|------------------------|
| User experience | Requires JOIN | Encapsulated TVF |
| Coordinator pruning | None | Centroid-based |
| Memory model | In-memory index | Disk-based (mmap) |
| Row ID dependency | Unclear, problematic on V1/V2 | V3 row lineage (explicit) |
| Core engine changes | New operator (TopKAggregationOperator) | Plan rewrite (TBD) |
| Index per partition | HNSW graph | IVF (+ optional HNSW sub-index) |
| Java integration | Pure Java | Rust + JNI |
| Recall characteristics | Higher (HNSW) | Good (IVF), tunable |
| Scale limit (coordinator) | ~10M vectors | ~100M+ vectors (disk-based) |

---

## Part 7: Open Questions

1. **Scale threshold:** At what vector count does coordinator-only Lance search become insufficient?

2. **JNI performance:** What's the actual overhead for Lance JNI calls in Presto coordinator?

3. **Plan rewrite mechanism:** What's the best integration point for transforming TVF → filtered scan?

4. **Iceberg V3 adoption:** How widely adopted is V3? Is requiring it acceptable?

5. **Hybrid approach:** Could we support both approaches (HNSW for small, IVF for large)?

6. **Index freshness:** What's the acceptable staleness for vector indexes? How often to rebuild?

7. **Filtered search:** How do pre-filters (WHERE clauses) interact with vector search? Filter before or after ANN?

---

## References

### RFC and Presto
- [RFC-0022: JVector Integration](https://github.com/prestodb/rfcs/pull/55)
- [Presto ConnectorIndex SPI](https://github.com/prestodb/presto/blob/master/presto-spi/src/main/java/com/facebook/presto/spi/ConnectorIndex.java)
- [Presto Index Join Issue #11899](https://github.com/prestodb/presto/issues/11899)

### Lance and LanceDB
- [Lance Format GitHub](https://github.com/lance-format/lance)
- [LanceDB IVF-PQ Concepts](https://lancedb.github.io/lancedb/concepts/index_ivfpq/)
- [Lance JNI Crate](https://crates.io/crates/lance-jni/dependencies)
- [Lance-Trino Connector](https://github.com/lancedb/lance-trino)

### Iceberg
- [Iceberg V3 Row Lineage](https://iceberg.apache.org/spec/)
- [Puffin File Format](https://iceberg.apache.org/puffin-spec/)

### Vector Search Background
- [HNSW vs IVF Comparison (Milvus)](https://milvus.io/blog/understanding-ivf-vector-index-how-It-works-and-when-to-choose-it-over-hnsw.md)
- [DuckDB Vector Similarity Search](https://duckdb.org/2024/05/03/vector-similarity-search-vss)
- [Qdrant Memory Consumption](https://qdrant.tech/articles/memory-consumption/)

---

## Document History

- **Created:** Based on RFC-0022 review conversation
- **Purpose:** Context for prototyping vector search integration in Presto

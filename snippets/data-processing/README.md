# Data Processing

Tabular, streaming, ETL or validation pipeline transformation is the primary
problem.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Aggregate Consecutive Values into Weighted Runs](aggregate-consecutive-values-into-weighted-runs.md) | algorithm | data-transformation | Combine adjacent equal keys in an ordered stream while preserving run order, item count, and a finite non-negative total weight. |
| [Batch Items by Estimated Byte Size](batch-items-by-estimated-byte-size.md) | algorithm | data-transformation, resource-management | Group a stream lazily into tuples whose positive estimated sizes never exceed a strict per-batch byte limit. |
| [Yield Stream Items with Bounded Neighbor Context](yield-stream-items-with-bounded-neighbor-context.md) | recipe | data-transformation, performance-optimization | Yield each stream item with bounded chronological history and one explicit lookahead without materializing the complete source. |
<!-- catalog:category:end -->

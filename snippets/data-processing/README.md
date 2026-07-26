# Data Processing

Tabular, streaming, ETL or validation pipeline transformation is the primary
problem.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Aggregate Consecutive Values into Weighted Runs](aggregate-consecutive-values-into-weighted-runs.md) | algorithm | data-transformation | Combine adjacent equal keys in an ordered stream while preserving run order, item count, and a finite non-negative total weight. |
| [Batch Items by Estimated Byte Size](batch-items-by-estimated-byte-size.md) | algorithm | data-transformation, resource-management | Group a stream lazily into tuples whose positive estimated sizes never exceed a strict per-batch byte limit. |
| [Normalize Optional CSV Columns in a Single Pass](normalize-optional-csv-columns-in-a-single-pass.md) | recipe | data-transformation, parsing, validation | Normalize selected columns in already parsed CSV rows while handling optional columns, short rows, and duplicate normalized records explicitly. |
| [Sample a Stream with a Fixed-Size Reservoir](sample-a-stream-with-a-fixed-size-reservoir.md) | algorithm | data-transformation, resource-management | Choose up to a fixed number of positions uniformly from a single-pass stream without knowing its length in advance. |
| [Sample Stream Items Independently with a Fixed Probability](sample-stream-items-independently-with-a-fixed-probability.md) | algorithm | data-transformation | Select each stream item independently with a known probability without storing the unsampled input. |
| [Split Quoted and Bracketed Log Fields](split-quoted-and-bracketed-log-fields.md) | algorithm | data-transformation, parsing | Split a log line whose fields may be plain tokens, double-quoted text, or bracketed text without losing embedded spaces. |
| [Yield Stream Items with Bounded Neighbor Context](yield-stream-items-with-bounded-neighbor-context.md) | recipe | data-transformation, performance-optimization | Yield each stream item with bounded chronological history and one explicit lookahead without materializing the complete source. |
<!-- catalog:category:end -->

# Data Processing

Tabular, streaming, ETL or validation pipeline transformation is the primary
problem.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Aggregate Consecutive Values into Weighted Runs](aggregate-consecutive-values-into-weighted-runs.md) | algorithm | data-transformation | Combine adjacent equal keys in an ordered stream while preserving run order, item count, and a finite non-negative total weight. |
| [Batch Items by Estimated Byte Size](batch-items-by-estimated-byte-size.md) | algorithm | data-transformation, resource-management | Group a stream lazily into tuples whose positive estimated sizes never exceed a strict per-batch byte limit. |
| [Check a Value Against an Asymmetric Tolerance Band](check-a-value-against-an-asymmetric-tolerance-band.md) | algorithm | validation | Validate one finite observation against independently sized lower and upper margins around a finite reference value. |
| [Collect Expected Parse Failures Without Stopping a Batch](collect-expected-parse-failures-without-stopping-a-batch.md) | pattern | parsing, validation | Represent expected input failures as typed values so one malformed item does not discard successful results from the same batch. |
| [Limit Text Lines Across Arbitrary Chunks](limit-text-lines-across-arbitrary-chunks.md) | recipe | parsing, resource-management | Reassemble separator-delimited text arriving in arbitrary chunks while retaining only a bounded prefix of each logical line. |
| [Measure Time in a State Within a Half-Open Window](measure-time-in-a-state-within-a-half-open-window.md) | algorithm | data-transformation | Measure how long an append-only state history occupies one target state inside an exact half-open reporting window. |
| [Normalize Optional CSV Columns in a Single Pass](normalize-optional-csv-columns-in-a-single-pass.md) | recipe | data-transformation, parsing, validation | Normalize selected columns in already parsed CSV rows while handling optional columns, short rows, and duplicate normalized records explicitly. |
| [Parse Pipe-Delimited Tables with Continuation Rows](parse-pipe-delimited-tables-with-continuation-rows.md) | algorithm | data-transformation, parsing | Parse one deliberately small pipe-delimited table while preserving non-table text and malformed post-header rows for inspection. |
| [Project Nested Records with Explicit Field Paths](project-nested-records-with-explicit-field-paths.md) | recipe | data-transformation, validation | Project heterogeneous nested mappings through one declared field list while auditing required paths that are absent. |
| [Route Items by Ordered Text Prefixes](route-items-by-ordered-text-prefixes.md) | recipe | data-transformation, validation | Partition items by the first matching text prefix while preserving priority, input order, and otherwise-unmatched data. |
| [Sample a Stream with a Fixed-Size Reservoir](sample-a-stream-with-a-fixed-size-reservoir.md) | algorithm | data-transformation, resource-management | Choose up to a fixed number of positions uniformly from a single-pass stream without knowing its length in advance. |
| [Sample a Weighted Stream with a Fixed-Size Reservoir](sample-a-weighted-stream-with-a-fixed-size-reservoir.md) | algorithm | data-transformation, performance-optimization, resource-management | Draw a fixed-size weighted sample without replacement from a finite one-pass stream while retaining only the current reservoir. |
| [Sample Stream Items Independently with a Fixed Probability](sample-stream-items-independently-with-a-fixed-probability.md) | algorithm | data-transformation | Select each stream item independently with a known probability without storing the unsampled input. |
| [Select One Record per Key with an Explicit Ranking Rule](select-one-record-per-key-with-an-explicit-ranking-rule.md) | algorithm | data-transformation, validation | Choose one record per logical key with a stable documented ranking rule and an audit of every group decision. |
| [Split Quoted and Bracketed Log Fields](split-quoted-and-bracketed-log-fields.md) | algorithm | data-transformation, parsing | Split a log line whose fields may be plain tokens, double-quoted text, or bracketed text without losing embedded spaces. |
| [Yield Stream Items with Bounded Neighbor Context](yield-stream-items-with-bounded-neighbor-context.md) | recipe | data-transformation, performance-optimization | Yield each stream item with bounded chronological history and one explicit lookahead without materializing the complete source. |
<!-- catalog:category:end -->

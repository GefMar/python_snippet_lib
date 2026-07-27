# Machine Learning and Statistics

Statistical assumptions, model behavior, features or evaluation semantics
determine correctness.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Assemble Out-of-Fold Scores from Explicit Validation Splits](assemble-out-of-fold-scores-from-explicit-validation-splits.md) | algorithm | data-transformation, validation | Assemble independently computed validation scores into one immutable row-order result while proving that every source row is owned by exactly one fold. |
| [Calculate a Symmetrically Trimmed Mean](calculate-a-symmetrically-trimmed-mean.md) | recipe | data-transformation, performance-optimization | Reduce the influence of extreme observations by removing an explicit count from both sorted ends before taking the mean. |
| [Compute a Row-Wise Maximum of Rolling Minima](compute-a-row-wise-maximum-of-rolling-minima.md) | algorithm | data-transformation, validation | Summarize each bounded time row by taking the largest minimum found across all of its contiguous fixed-width windows. |
| [Compute a Wilson Score Interval for a Binomial Proportion](compute-a-wilson-score-interval-for-a-binomial-proportion.md) | algorithm | data-transformation, validation | Estimate a two-sided confidence interval for a binomial proportion with Wilson score bounds instead of a symmetric Wald interval. |
| [Create Past-Only pandas Lag and Rolling-Mean Columns](create-past-only-pandas-lag-and-rolling-mean-columns.md) | integration | data-transformation, validation | Add deterministic lag and complete rolling-mean features that use only earlier rows from the same contiguous group. |
| [Detect a Recent Drop Against a Disjoint pandas Baseline Window](detect-a-recent-drop-against-a-disjoint-pandas-baseline-window.md) | integration | observability, validation | Compare recent finite measurements with an earlier non-overlapping baseline and return a diagnostic drop decision. |
| [Encode Categories with Out-of-Fold Smoothed Target Means](encode-categories-with-out-of-fold-smoothed-target-means.md) | algorithm | data-transformation, validation | Encode bounded categorical training rows with smoothed target means calculated without any observation from the row's assigned fold. |
| [Encode Cyclic Positions with Sine and Cosine](encode-cyclic-positions-with-sine-and-cosine.md) | algorithm | data-transformation | Map a position on a known cycle to sine and cosine coordinates so the cycle has no artificial numeric discontinuity. |
| [Fit and Apply a Frozen pandas Median Z-Score Profile](fit-and-apply-a-frozen-pandas-median-z-score-profile.md) | integration | data-transformation, validation | Fit median imputation and population z-score statistics once on a bounded pandas training frame, then apply the frozen profile to later frames without changing the inputs or learned state. |
| [Fit and Apply an Exact Categorical Frequency Encoder](fit-and-apply-an-exact-categorical-frequency-encoder.md) | recipe | data-transformation, validation | Fit an immutable exact count-and-total mapping for bounded categories, then encode known frequencies and map unseen categories to zero. |
| [Fit and Apply Fixed Quantile Clipping Bounds](fit-and-apply-fixed-quantile-clipping-bounds.md) | recipe | data-transformation, validation | Fit immutable quantile bounds once from training values, then clip future values to those same bounds without mutating either sequence or refitting. |
| [Fit PCA with NumPy and Report Cumulative Explained Variance](fit-pca-with-numpy-and-report-cumulative-explained-variance.md) | algorithm | data-transformation, validation | Fit principal component analysis to a bounded real matrix and report how much centered variation each singular direction explains. |
| [Flag Groupwise Numeric Outliers with IQR Fences](flag-groupwise-numeric-outliers-with-iqr-fences.md) | algorithm | data-transformation, validation | Flag observations strictly outside per-group interquartile-range fences while preserving enough context to inspect every decision. |
| [Measure Drift Between Two Fixed-Bin Count Distributions with PSI](measure-drift-between-two-fixed-bin-count-distributions-with-psi.md) | algorithm | observability, validation | Calculate Population Stability Index from two bounded count vectors that share one fixed bin definition while making zero-support behavior explicit. |
| [Sample Weighted Negative Items Outside Explicit User Histories](sample-weighted-negative-items-outside-explicit-user-histories.md) | algorithm | data-transformation, validation | Draw reproducible weighted negative items with replacement from a frozen universe while excluding each user's explicit history. |
| [Score Feature Importances Against Bounded Null Runs](score-feature-importances-against-bounded-null-runs.md) | algorithm | data-transformation, validation | Compare each observed feature importance with an aligned matrix of importances from fixed null runs and report its strict empirical rank. |
| [Select a Forecast Vector Only When It Beats a Frozen Baseline](select-a-forecast-vector-only-when-it-beats-a-frozen-baseline.md) | algorithm | data-transformation, validation | Select a precomputed forecast vector only when its finite mean absolute error is strictly lower than that of one frozen baseline. |
<!-- catalog:category:end -->

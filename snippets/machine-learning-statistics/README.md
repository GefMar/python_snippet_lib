# Machine Learning and Statistics

Statistical assumptions, model behavior, features or evaluation semantics
determine correctness.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Calculate a Symmetrically Trimmed Mean](calculate-a-symmetrically-trimmed-mean.md) | recipe | data-transformation, performance-optimization | Reduce the influence of extreme observations by removing an explicit count from both sorted ends before taking the mean. |
| [Compute a Wilson Score Interval for a Binomial Proportion](compute-a-wilson-score-interval-for-a-binomial-proportion.md) | algorithm | data-transformation, validation | Estimate a two-sided confidence interval for a binomial proportion with Wilson score bounds instead of a symmetric Wald interval. |
| [Detect a Recent Drop Against a Disjoint pandas Baseline Window](detect-a-recent-drop-against-a-disjoint-pandas-baseline-window.md) | integration | observability, validation | Compare recent finite measurements with an earlier non-overlapping baseline and return a diagnostic drop decision. |
| [Encode Categories with Out-of-Fold Smoothed Target Means](encode-categories-with-out-of-fold-smoothed-target-means.md) | algorithm | data-transformation, validation | Encode bounded categorical training rows with smoothed target means calculated without any observation from the row's assigned fold. |
| [Encode Cyclic Positions with Sine and Cosine](encode-cyclic-positions-with-sine-and-cosine.md) | algorithm | data-transformation | Map a position on a known cycle to sine and cosine coordinates so the cycle has no artificial numeric discontinuity. |
| [Fit and Apply an Exact Categorical Frequency Encoder](fit-and-apply-an-exact-categorical-frequency-encoder.md) | recipe | data-transformation, validation | Fit an immutable exact count-and-total mapping for bounded categories, then encode known frequencies and map unseen categories to zero. |
| [Fit and Apply Fixed Quantile Clipping Bounds](fit-and-apply-fixed-quantile-clipping-bounds.md) | recipe | data-transformation, validation | Fit immutable quantile bounds once from training values, then clip future values to those same bounds without mutating either sequence or refitting. |
| [Fit PCA with NumPy and Report Cumulative Explained Variance](fit-pca-with-numpy-and-report-cumulative-explained-variance.md) | algorithm | data-transformation, validation | Fit principal component analysis to a bounded real matrix and report how much centered variation each singular direction explains. |
| [Flag Groupwise Numeric Outliers with IQR Fences](flag-groupwise-numeric-outliers-with-iqr-fences.md) | algorithm | data-transformation, validation | Flag observations strictly outside per-group interquartile-range fences while preserving enough context to inspect every decision. |
| [Measure Drift Between Two Fixed-Bin Count Distributions with PSI](measure-drift-between-two-fixed-bin-count-distributions-with-psi.md) | algorithm | observability, validation | Calculate Population Stability Index from two bounded count vectors that share one fixed bin definition while making zero-support behavior explicit. |
| [Score Feature Importances Against Bounded Null Runs](score-feature-importances-against-bounded-null-runs.md) | algorithm | data-transformation, validation | Compare each observed feature importance with an aligned matrix of importances from fixed null runs and report its strict empirical rank. |
<!-- catalog:category:end -->

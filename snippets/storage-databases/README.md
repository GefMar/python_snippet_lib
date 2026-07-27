# Storage and Databases

Durability, transaction, query, migration, filesystem or persistence semantics
determine correctness.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Build and Apply a Deterministic Mapping Delta](build-and-apply-a-deterministic-mapping-delta.md) | algorithm | data-transformation, persistence | Represent the exact differences between two string-key mappings as sorted set and delete operations that can be applied to a shallow copy. |
| [Check Whether a Generated File Is Older Than Its Inputs](check-whether-a-generated-file-is-older-than-its-inputs.md) | recipe | automation, caching, validation | Decide whether a generated file needs rebuilding by comparing its modification time with every declared input without changing any filesystem metadata. |
| [Compare and Swap a Versioned SQLite Setting with One Conditional Update](compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md) | recipe | concurrency-control, persistence | Replace a SQLite setting only when its stored version still matches the version previously observed by the caller. |
| [Compare Bounded Apparent Sizes of Two File-Tree Snapshots](compare-bounded-apparent-sizes-of-two-file-tree-snapshots.md) | recipe | automation, persistence, resource-management | Capture two deterministic file-tree snapshots and report which paths changed after rolling leaf sizes into every parent directory. |
| [Read the Last Bounded Binary Lines with a Read-Only mmap](read-the-last-bounded-binary-lines-with-a-read-only-mmap.md) | recipe | parsing, performance-optimization, resource-management | Read a bounded number of final binary lines newest first without loading or scanning an already size-capped regular file from the beginning. |
| [Replace a File Atomically with a Sibling Temporary File](replace-a-file-atomically-with-a-sibling-temporary-file.md) | recipe | persistence, resource-management | Write and synchronize a sibling temporary file before replacing the target so readers never observe partially written content. |
| [Select Snapshot Representatives by UTC Calendar Buckets](select-snapshot-representatives-by-utc-calendar-buckets.md) | algorithm | automation, persistence | Select the newest snapshot from each recent populated UTC day, ISO week, and month without performing deletion as part of the policy. |
| [Split a Binary Stream into Exclusively Created Numbered Parts](split-a-binary-stream-into-exclusively-created-numbered-parts.md) | recipe | automation, persistence, resource-management | Copy a finite blocking binary stream into bounded numbered files without overwriting any path that already exists. |
| [Split a Half-Open UTC Range Across Ordered Storage Tiers](split-a-half-open-utc-range-across-ordered-storage-tiers.md) | algorithm | data-transformation, persistence, validation | Intersect one half-open time range with ordered storage-tier cutovers so every instant is routed exactly once without building a query string. |
| [Store Bytes by Their Content Digest](store-bytes-by-their-content-digest.md) | recipe | caching, persistence | Store immutable byte payloads under paths derived from their SHA-256 digest so identical content shares one deterministic location. |
<!-- catalog:category:end -->

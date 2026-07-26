# Storage and Databases

Durability, transaction, query, migration, filesystem or persistence semantics
determine correctness.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Build and Apply a Deterministic Mapping Delta](build-and-apply-a-deterministic-mapping-delta.md) | algorithm | data-transformation, persistence | Represent the exact differences between two string-key mappings as sorted set and delete operations that can be applied to a shallow copy. |
| [Store Bytes by Their Content Digest](store-bytes-by-their-content-digest.md) | recipe | caching, persistence | Store immutable byte payloads under paths derived from their SHA-256 digest so identical content shares one deterministic location. |
<!-- catalog:category:end -->

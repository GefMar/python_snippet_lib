# Python Language

Python data-model, typing, decorator, descriptor or metaprogramming semantics
are the learning objective, not merely the implementation language.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Make a Defensive Copy at a Mutable Input Boundary](make-a-defensive-copy-at-a-mutable-input-boundary.md) | idiom | data-transformation, resource-management | Copy a caller-owned mapping before retaining it so later top-level mutations cannot silently change the stored state. |
| [Pass Constructor-Only Context with dataclasses.InitVar](pass-constructor-only-context-with-initvar.md) | idiom | data-transformation, validation | Use an InitVar when dataclass construction needs a temporary input that should not become part of the stored field model. |
| [Read Fixed-Size Blocks with iter() and a Sentinel](read-fixed-size-blocks-with-iter-sentinel.md) | idiom | performance-optimization, resource-management | Use the callable-and-sentinel form of iter to read a binary stream lazily until the end-of-file marker appears. |
| [Walk a Tree Recursively with yield from](walk-a-tree-recursively-with-yield-from.md) | idiom | data-transformation | Delegate each recursive subtree to another generator to produce a lazy preorder traversal with little coordination code. |
<!-- catalog:category:end -->

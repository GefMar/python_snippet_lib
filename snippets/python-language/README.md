# Python Language

Python data-model, typing, decorator, descriptor or metaprogramming semantics
are the learning objective, not merely the implementation language.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Apply Partial Dataclass Updates with an Omitted-Value Sentinel](apply-partial-dataclass-updates-with-an-omitted-value-sentinel.md) | recipe | data-transformation, validation | Use a unique sentinel to distinguish an omitted dataclass patch field from an explicit None, false, or zero value. |
| [Batch Any Iterable Lazily with itertools.batched](batch-any-iterable-lazily-with-itertools-batched.md) | idiom | data-transformation, resource-management | Use the standard library batching iterator to consume any iterable lazily as fixed-size tuples without reimplementing iterator slicing. |
| [Build an Immutable Slice-Aware Sequence](build-an-immutable-slice-aware-sequence.md) | recipe | data-transformation, interoperability | Back a custom sequence with a tuple and return the same documented sequence type for slices instead of exposing the storage tuple. |
| [Dispatch Named Strategies with an Explicit Function Mapping](dispatch-named-strategies-with-an-explicit-function-mapping.md) | pattern | configuration, interoperability | Select stateless behavior by name with an explicit read-only mapping instead of a class hierarchy or hidden global registry. |
| [Keep Exception Handlers Narrow with try/else](keep-exception-handlers-narrow-with-try-else.md) | idiom | parsing, validation | Put success-only work in a try statement's else clause so the preceding handler covers only the operation expected to fail. |
| [Load Text Templates from Package Resources](load-text-templates-from-package-resources.md) | recipe | resource-management, validation | Load a text template from ordered package-resource roots without assuming that packaged data has a persistent filesystem path. |
| [Make a Defensive Copy at a Mutable Input Boundary](make-a-defensive-copy-at-a-mutable-input-boundary.md) | idiom | data-transformation, resource-management | Copy a caller-owned mapping before retaining it so later top-level mutations cannot silently change the stored state. |
| [Model a Quantity with One Canonical Unit](model-a-quantity-with-one-canonical-unit.md) | pattern | data-transformation, validation | Represent a small quantity with one immutable canonical value while exposing named factories and read-only properties for alternate units. |
| [Pass Constructor-Only Context with dataclasses.InitVar](pass-constructor-only-context-with-initvar.md) | idiom | data-transformation, validation | Use an InitVar when dataclass construction needs a temporary input that should not become part of the stored field model. |
| [Read Fixed-Size Blocks with iter() and a Sentinel](read-fixed-size-blocks-with-iter-sentinel.md) | idiom | performance-optimization, resource-management | Use the callable-and-sentinel form of iter to read a binary stream lazily until the end-of-file marker appears. |
| [Validate Reused Fields with a Data Descriptor](validate-reused-fields-with-a-data-descriptor.md) | pattern | validation | Centralize a repeated attribute rule in a data descriptor that validates every assignment and stores each field independently. |
| [Walk a Tree Recursively with yield from](walk-a-tree-recursively-with-yield-from.md) | idiom | data-transformation | Delegate each recursive subtree to another generator to produce a lazy preorder traversal with little coordination code. |
<!-- catalog:category:end -->

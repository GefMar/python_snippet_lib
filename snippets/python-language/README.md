# Python Language

Python data-model, typing, decorator, descriptor or metaprogramming semantics
are the learning objective, not merely the implementation language.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Add Bounded Stage Context Without Replacing an Exception](add-bounded-stage-context-without-replacing-an-exception.md) | idiom | observability, validation | Add one bounded diagnostic note at a synchronous stage boundary, then re-raise the same exception without replacing its type or traceback chain. |
| [Apply Partial Dataclass Updates with an Omitted-Value Sentinel](apply-partial-dataclass-updates-with-an-omitted-value-sentinel.md) | recipe | data-transformation, validation | Use a unique sentinel to distinguish an omitted dataclass patch field from an explicit None, false, or zero value. |
| [Batch Any Iterable Lazily with itertools.batched](batch-any-iterable-lazily-with-itertools-batched.md) | idiom | data-transformation, resource-management | Use the standard library batching iterator to consume any iterable lazily as fixed-size tuples without reimplementing iterator slicing. |
| [Build a Read-Only Mapping with Canonical Text Keys](build-a-read-only-mapping-with-canonical-text-keys.md) | recipe | interoperability, validation | Build a read-only mapping once so construction, lookup, membership, get, and iteration all share one Unicode caseless key invariant. |
| [Build an Immutable Slice-Aware Sequence](build-an-immutable-slice-aware-sequence.md) | recipe | data-transformation, interoperability | Back a custom sequence with a tuple and return the same documented sequence type for slices instead of exposing the storage tuple. |
| [Bypass an LRU Cache with a Per-Call Predicate](bypass-an-lru-cache-with-a-per-call-predicate.md) | pattern | caching, performance-optimization | Wrap a bounded LRU cache with a predicate that can send one call directly to the underlying function without reading or updating cached state. |
| [Cache One Zero-Argument Method Result per Weakly Referenced Instance](cache-one-zero-argument-method-result-per-weakly-referenced-instance.md) | pattern | caching, resource-management | Cache one successful zero-argument method result per object without letting the cache's key keep that object alive. |
| [Collect Decorated Methods in Class Definition Order](collect-decorated-methods-in-class-definition-order.md) | pattern | automation, configuration | Build an immutable per-class registry from explicitly decorated instance methods without inspecting caller frames or maintaining global mutable state. |
| [Dispatch Named Strategies with an Explicit Function Mapping](dispatch-named-strategies-with-an-explicit-function-mapping.md) | pattern | configuration, interoperability | Select stateless behavior by name with an explicit read-only mapping instead of a class hierarchy or hidden global registry. |
| [Dispatch on an Exact Tuple of Argument Types](dispatch-on-an-exact-tuple-of-argument-types.md) | pattern | data-transformation, interoperability, validation | Select one handler from the exact ordered runtime types of several positional arguments, using a bounded registry assembled at an explicit composition point. |
| [Handle Search Exhaustion with for/else](handle-search-exhaustion-with-for-else.md) | idiom | data-transformation, validation | Use a loop else clause to handle search failure only after an iterable ends without a matching break. |
| [Keep Exception Handlers Narrow with try/else](keep-exception-handlers-narrow-with-try-else.md) | idiom | parsing, validation | Put success-only work in a try statement's else clause so the preceding handler covers only the operation expected to fail. |
| [Load Bounded Trusted Extension Factories by One Entry Point](load-bounded-trusted-extension-factories-by-one-entry-point.md) | recipe | interoperability, validation | Load one consistently named factory from each explicitly trusted module after validating the complete bounded request. |
| [Load Text Templates from Package Resources](load-text-templates-from-package-resources.md) | recipe | resource-management, validation | Load a text template from ordered package-resource roots without assuming that packaged data has a persistent filesystem path. |
| [Make a Defensive Copy at a Mutable Input Boundary](make-a-defensive-copy-at-a-mutable-input-boundary.md) | idiom | data-transformation, resource-management | Copy a caller-owned mapping before retaining it so later top-level mutations cannot silently change the stored state. |
| [Model a Quantity with One Canonical Unit](model-a-quantity-with-one-canonical-unit.md) | pattern | data-transformation, validation | Represent a small quantity with one immutable canonical value while exposing named factories and read-only properties for alternate units. |
| [Normalize Bounded Named Options with Explicit Default Semantics](normalize-bounded-named-options-with-explicit-default-semantics.md) | recipe | configuration, data-transformation, validation | Normalize a closed mapping of named scalar options only after validating a complete ordered schema with explicit missing, default, and nullable semantics. |
| [Pass Constructor-Only Context with dataclasses.InitVar](pass-constructor-only-context-with-initvar.md) | idiom | data-transformation, validation | Use an InitVar when dataclass construction needs a temporary input that should not become part of the stored field model. |
| [Read Fixed-Size Blocks with iter() and a Sentinel](read-fixed-size-blocks-with-iter-sentinel.md) | idiom | performance-optimization, resource-management | Use the callable-and-sentinel form of iter to read a binary stream lazily until the end-of-file marker appears. |
| [Type a Narrow Structural Interface with Protocol](type-a-narrow-structural-interface-with-protocol.md) | pattern | interoperability, validation | Describe one consumer-owned capability structurally so compatible adapters do not need to inherit from a shared base class. |
| [Validate Reused Fields with a Data Descriptor](validate-reused-fields-with-a-data-descriptor.md) | pattern | validation | Centralize a repeated attribute rule in a data descriptor that validates every assignment and stores each field independently. |
| [Walk a Tree Recursively with yield from](walk-a-tree-recursively-with-yield-from.md) | idiom | data-transformation | Delegate each recursive subtree to another generator to produce a lazy preorder traversal with little coordination code. |
<!-- catalog:category:end -->

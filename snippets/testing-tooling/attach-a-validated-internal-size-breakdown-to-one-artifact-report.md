---
title: "Attach a Validated Internal Size Breakdown to One Artifact Report"
snippet_type: recipe
use_cases:
  - data-transformation
  - resource-management
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - audit-a-bounded-directive-tree-with-isolated-inherited-context.md
  - ../data-processing/derive-an-other-bucket-from-exact-pandas-totals.md
  - ../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md
---

# Attach a Validated Internal Size Breakdown to One Artifact Report

## Idea and Problem

Attach a validated deep copy of an inclusive size hierarchy without replacing the artifact's authoritative total size.

The hierarchy root is the explicitly attributed byte count. The difference
between the outer total and that root is reported separately as unattributed
bytes. Every internal node may retain its own remainder because only the sum of
its immediate children's inclusive sizes is constrained by its size.

## When to Use

Use this recipe after a trusted parser or measurement stage has already
produced one immutable artifact report and one immutable in-memory hierarchy.
It fits test reports and build summaries that need to add explanatory structure
while retaining a separately measured outer total as the authority.

Names must be non-empty, stripped, printable UTF-8 text. Sibling uniqueness is
defined by `unicodedata.normalize("NFKC", name).casefold()`; the display name is
otherwise preserved. Sizes are exact non-negative integers within the signed
64-bit range. The function accepts no paths, bytes, callbacks, or mutable
containers.

## Implementation

```python
import unicodedata
from dataclasses import dataclass

_MAX_DEPTH = 16
_MAX_NODES = 4_096
_MAX_CHILDREN = 256
_MAX_NAME_BYTES = 256
_MAX_NORMALIZED_NAME_BYTES = 1_024
_MAX_TOTAL_NAME_BYTES = 256 * 1_024
_MAX_WORK_UNITS = 1_000_000
_MAX_SIZE_BYTES = 2**63 - 1


@dataclass(frozen=True, slots=True)
class ArtifactReport:
    total_size_bytes: int


@dataclass(frozen=True, slots=True)
class InternalSizeNode:
    name: str
    size_bytes: int
    children: tuple[InternalSizeNode, ...] = ()


@dataclass(frozen=True, slots=True)
class ArtifactReportWithBreakdown:
    total_size_bytes: int
    hierarchy: InternalSizeNode
    attributed_bytes: int
    unattributed_bytes: int


@dataclass(slots=True)
class _Budget:
    nodes: int = 0
    name_bytes: int = 0
    work_units: int = 0


def _validated_size(value: object, *, field: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact integer")
    if not 0 <= value <= _MAX_SIZE_BYTES:
        raise ValueError(f"{field} is outside the supported size range")
    return value


def _validated_name(value: object) -> tuple[str, str, int, int]:
    if type(value) is not str:
        raise TypeError("node names must be exact strings")
    if not value or len(value) > _MAX_NAME_BYTES:
        raise ValueError("a node name is empty or too long")
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError as error:
        raise ValueError("node names must be valid UTF-8 text") from error
    if len(encoded) > _MAX_NAME_BYTES or value != value.strip() or not value.isprintable():
        raise ValueError("a node name is unstripped, unprintable, or too long")

    key = unicodedata.normalize("NFKC", value).casefold()
    key_size = len(key.encode("utf-8"))
    if key_size > _MAX_NORMALIZED_NAME_BYTES:
        raise ValueError("a normalized node name exceeds the supported byte limit")
    return value, key, len(encoded), key_size


def attach_internal_size_breakdown(
    report: ArtifactReport,
    hierarchy: InternalSizeNode,
) -> ArtifactReportWithBreakdown:
    if type(report) is not ArtifactReport:
        raise TypeError("report must be an exact ArtifactReport")
    outer_total = _validated_size(
        report.total_size_bytes,
        field="report.total_size_bytes",
    )

    budget = _Budget()
    seen_nodes: set[int] = set()
    active_nodes: set[int] = set()

    def charge_work(amount: int) -> None:
        budget.work_units += amount
        if budget.work_units > _MAX_WORK_UNITS:
            raise ValueError("hierarchy exceeds the aggregate work limit")

    def visit(
        raw_node: object,
        *,
        depth: int,
        field: str,
    ) -> tuple[InternalSizeNode, str]:
        if type(raw_node) is not InternalSizeNode:
            raise TypeError(f"{field} must be an exact InternalSizeNode")
        if depth > _MAX_DEPTH:
            raise ValueError("hierarchy exceeds the supported depth")

        identity = id(raw_node)
        if identity in active_nodes:
            raise ValueError("hierarchy contains a cycle")
        if identity in seen_nodes:
            raise ValueError("hierarchy contains a shared-node alias")
        seen_nodes.add(identity)
        active_nodes.add(identity)
        budget.nodes += 1
        try:
            if budget.nodes > _MAX_NODES:
                raise ValueError("hierarchy exceeds the supported node count")

            name, name_key, name_size, key_size = _validated_name(raw_node.name)
            budget.name_bytes += name_size
            if budget.name_bytes > _MAX_TOTAL_NAME_BYTES:
                raise ValueError("hierarchy exceeds the aggregate name-byte limit")
            charge_work(1 + name_size + key_size)
            size = _validated_size(raw_node.size_bytes, field=f"{field}.size_bytes")

            if type(raw_node.children) is not tuple:
                raise TypeError(f"{field}.children must be an exact tuple")
            if len(raw_node.children) > _MAX_CHILDREN:
                raise ValueError("a node exceeds the supported child count")
            charge_work(len(raw_node.children))

            children: list[InternalSizeNode] = []
            sibling_keys: set[str] = set()
            child_total = 0
            for index, raw_child in enumerate(raw_node.children):
                child, child_key = visit(
                    raw_child,
                    depth=depth + 1,
                    field=f"{field}.children[{index}]",
                )
                if child_key in sibling_keys:
                    raise ValueError("sibling names collide after NFKC casefolding")
                sibling_keys.add(child_key)
                child_total += child.size_bytes
                if child_total > size:
                    raise ValueError("the sum of child sizes exceeds the parent size")
                children.append(child)

            return InternalSizeNode(name, size, tuple(children)), name_key
        finally:
            active_nodes.remove(identity)

    frozen_root, _ = visit(hierarchy, depth=1, field="hierarchy")
    if frozen_root.size_bytes > outer_total:
        raise ValueError("the internal root exceeds the authoritative artifact size")

    return ArtifactReportWithBreakdown(
        total_size_bytes=outer_total,
        hierarchy=frozen_root,
        attributed_bytes=frozen_root.size_bytes,
        unattributed_bytes=outer_total - frozen_root.size_bytes,
    )
```

## Example

```python
parsed = InternalSizeNode(
    "content",
    800,
    (
        InternalSizeNode("code", 450),
        InternalSizeNode("data", 300),
    ),
)
result = attach_internal_size_breakdown(
    ArtifactReport(total_size_bytes=1_000),
    parsed,
)

assert result == ArtifactReportWithBreakdown(
    total_size_bytes=1_000,
    hierarchy=InternalSizeNode(
        "content",
        800,
        (
            InternalSizeNode("code", 450),
            InternalSizeNode("data", 300),
        ),
    ),
    attributed_bytes=800,
    unattributed_bytes=200,
)
assert result.hierarchy is not parsed
assert result.hierarchy.children[0] is not parsed.children[0]
```

## Trade-offs and Limitations

Inclusive accounting is an explicit reporting policy, not an inference about
binary layout. A parent can include padding, headers, shared regions, or an
unclassified remainder, and summing every node across levels would double
count bytes. The implementation therefore calls only the root attributed and
uses the authoritative outer total for the outer residual; it never derives
that total from the tree.

The fixed depth, node, fan-out, UTF-8, integer, and aggregate-work limits reject
some legitimate large reports. NFKC casefolding deliberately treats sibling
spellings such as `Core` and `core` as duplicates but leaves equal names in
different parents independent. Recursion is safe only because depth is capped.
The helper reads no filesystem data, parses no binary, runs no analyzer, build
hook, or subprocess, records no provenance, and mutates neither input.

Tests should cover empty child tuples, zero-sized nodes, exact parent and root
fills, local remainders, each boundary and one value past it, boolean and
oversized integers, normalized sibling collisions, root overflow, cycles, and
the same node reused at two paths. Also assert that every returned node is a
distinct object with the same accepted scalar values.

## Related Snippets

<!-- catalog:related:start -->
- [Audit a Bounded Directive Tree with Isolated Inherited Context](audit-a-bounded-directive-tree-with-isolated-inherited-context.md)
- [Derive an Other Bucket from Exact pandas Totals](../data-processing/derive-an-other-bucket-from-exact-pandas-totals.md)
- [Make a Defensive Copy at a Mutable Input Boundary](../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md)
<!-- catalog:related:end -->

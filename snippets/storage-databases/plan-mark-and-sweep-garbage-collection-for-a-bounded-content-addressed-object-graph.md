---
title: "Plan Mark-and-Sweep Garbage Collection for a Bounded Content-Addressed Object Graph"
snippet_type: algorithm
use_cases:
  - persistence
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - store-bytes-by-their-content-digest.md
  - ../algorithms-data-structures/traverse-a-parent-graph-with-breadth-first-search.md
  - ../configuration-serialization/resolve-a-bounded-configuration-fragment-graph.md
---

# Plan Mark-and-Sweep Garbage Collection for a Bounded Content-Addressed Object Graph

## Idea and Problem

Partition one complete content-addressed object inventory into objects reachable from declared roots and objects that a later, separately guarded executor may collect.

Marking follows references through a closed snapshot and handles shared objects
and cycles. Sweeping is deliberately only a calculation: the digest-sorted
result performs no storage reads or deletion.

## When to Use

Use this planner after taking a consistent, bounded inventory of an immutable
object graph. It is useful for offline maintenance previews, deterministic unit
tests, and small stores where every root and reference must resolve before any
collection decision is considered.

Do not apply the collectible list directly to a live or changing store. Real
collectors need snapshot coordination, pins or leases, grace periods, failure
recovery, authorization, and a final reachability check close to deletion.

## Implementation

```python
import re
from dataclasses import dataclass

_MAX_OBJECTS = 4_096
_MAX_REFERENCES_PER_OBJECT = 64
_MAX_REFERENCES = 16_384
_MAX_ROOTS = 4_096
_DIGEST = re.compile(r"[0-9a-f]{64}", re.ASCII).fullmatch


@dataclass(frozen=True, slots=True)
class ObjectRecord:
    digest: str
    references: tuple[str, ...]


@dataclass(frozen=True, slots=True)
class CollectionPlan:
    reachable: tuple[str, ...]
    collectible: tuple[str, ...]


def _validate_digest(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if _DIGEST(value) is None:
        raise ValueError(f"{field} must be 64 lowercase hexadecimal characters")
    return value


def plan_object_collection(
    objects: tuple[ObjectRecord, ...],
    roots: tuple[str, ...],
) -> CollectionPlan:
    if type(objects) is not tuple:
        raise TypeError("objects must be an exact tuple")
    if len(objects) > _MAX_OBJECTS:
        raise ValueError("object count exceeds 4096")

    checked: list[ObjectRecord] = []
    total_references = 0
    for object_index, record in enumerate(objects):
        if type(record) is not ObjectRecord:
            raise TypeError(f"objects[{object_index}] must be an exact ObjectRecord")
        digest = _validate_digest(record.digest, field=f"objects[{object_index}].digest")
        if type(record.references) is not tuple:
            raise TypeError(f"objects[{object_index}].references must be an exact tuple")
        if len(record.references) > _MAX_REFERENCES_PER_OBJECT:
            raise ValueError(f"objects[{object_index}] has more than 64 references")
        references = tuple(
            _validate_digest(
                reference,
                field=f"objects[{object_index}].references[{reference_index}]",
            )
            for reference_index, reference in enumerate(record.references)
        )
        if len(set(references)) != len(references):
            raise ValueError(f"objects[{object_index}] contains duplicate references")
        total_references += len(references)
        if total_references > _MAX_REFERENCES:
            raise ValueError("total reference count exceeds 16384")
        checked.append(ObjectRecord(digest, references))

    ordered_digests = tuple(record.digest for record in checked)
    if len(set(ordered_digests)) != len(ordered_digests):
        raise ValueError("object digests must be unique")
    inventory = {record.digest: record.references for record in checked}

    if type(roots) is not tuple:
        raise TypeError("roots must be an exact tuple")
    if len(roots) > _MAX_ROOTS:
        raise ValueError("root count exceeds 4096")
    checked_roots = tuple(
        _validate_digest(root, field=f"roots[{index}]")
        for index, root in enumerate(roots)
    )
    if len(set(checked_roots)) != len(checked_roots):
        raise ValueError("roots must be unique")

    missing_roots = tuple(root for root in checked_roots if root not in inventory)
    if missing_roots:
        raise ValueError("every root must resolve in the inventory")
    if any(
        reference not in inventory
        for record in checked
        for reference in record.references
    ):
        raise ValueError("every reference must resolve in the inventory")

    marked: set[str] = set()
    pending = list(reversed(checked_roots))
    while pending:
        digest = pending.pop()
        if digest in marked:
            continue
        marked.add(digest)
        pending.extend(reversed(inventory[digest]))

    return CollectionPlan(
        reachable=tuple(sorted(marked)),
        collectible=tuple(sorted(set(ordered_digests) - marked)),
    )
```

## Example

```python
def digest(label: str) -> str:
    from hashlib import sha256

    return sha256(label.encode("ascii")).hexdigest()


root = digest("root")
left = digest("left")
right = digest("right")
shared = digest("shared")
orphan_a = digest("orphan-a")
orphan_b = digest("orphan-b")

inventory = (
    ObjectRecord(orphan_a, (orphan_b,)),
    ObjectRecord(root, (left, right)),
    ObjectRecord(shared, (left,)),
    ObjectRecord(left, (shared,)),
    ObjectRecord(orphan_b, (orphan_a,)),
    ObjectRecord(right, (shared,)),
)
plan = plan_object_collection(inventory, (root,))
empty_plan = plan_object_collection((), ())
collect_everything = plan_object_collection(inventory, ())
self_loop = plan_object_collection((ObjectRecord(root, (root,)),), (root,))


def independent_mark(
    records: tuple[ObjectRecord, ...],
    roots: set[str],
) -> set[str]:
    marked = set(roots)
    changed = True
    while changed:
        changed = False
        for record in records:
            if record.digest in marked:
                before = len(marked)
                marked.update(record.references)
                changed |= len(marked) != before
    return marked


expected_reachable = independent_mark(inventory, {root})
reordered = tuple(reversed(inventory))
reordered_plan = plan_object_collection(reordered, (root,))

try:
    plan_object_collection(
        (*inventory, ObjectRecord(digest("dangling-owner"), (digest("missing"),))),
        (root,),
    )
except ValueError:
    dangling_rejected = True
else:
    dangling_rejected = False

assert (
    set(plan.reachable) == expected_reachable == {root, left, right, shared}
    and plan.reachable
    == tuple(sorted(expected_reachable))
    and plan.collectible == tuple(sorted((orphan_a, orphan_b)))
    and reordered_plan == plan
    and empty_plan == CollectionPlan((), ())
    and collect_everything.reachable == ()
    and collect_everything.collectible
    == tuple(sorted(record.digest for record in inventory))
    and self_loop == CollectionPlan((root,), ())
    and set(plan.reachable).isdisjoint(plan.collectible)
    and set(plan.reachable) | set(plan.collectible)
    == {record.digest for record in inventory}
    and all(
        reference in set(plan.reachable)
        for record in inventory
        if record.digest in set(plan.reachable)
        for reference in record.references
    )
    and dangling_rejected
)
```

## Trade-offs and Limitations

Validation and marking take `O(N + E)` time and memory for `N` objects and `E`
references, followed by `O(N log N)` time for canonical sorting. Reordering the
same complete inventory or its reference tuples cannot change the result.

Digest syntax is validated, but object bytes are not loaded and hashes are not
recomputed. A closed graph is a safety precondition, so one missing root or
reference rejects the whole plan. Empty roots intentionally make every supplied
object collectible, so a caller should treat the root tuple as security-critical
input. The snapshot is assumed immutable for the duration of planning, and the
result contains no deletion order, batch sizing, retry state, or proof that a
later deletion remains safe.

## Related Snippets

<!-- catalog:related:start -->
- [Store Bytes by Their Content Digest](store-bytes-by-their-content-digest.md)
- [Traverse a Parent Graph with Breadth-First Search](../algorithms-data-structures/traverse-a-parent-graph-with-breadth-first-search.md)
- [Resolve a Bounded Configuration Fragment Graph](../configuration-serialization/resolve-a-bounded-configuration-fragment-graph.md)
<!-- catalog:related:end -->

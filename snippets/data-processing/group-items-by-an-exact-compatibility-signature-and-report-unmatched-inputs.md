---
title: "Group Items by an Exact Compatibility Signature and Report Unmatched Inputs"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - route-items-by-ordered-text-prefixes.md
  - select-one-record-per-key-with-an-explicit-ranking-rule.md
  - ../algorithms-data-structures/apportion-a-non-negative-integer-total-without-rounding-drift.md
---

# Group Items by an Exact Compatibility Signature and Report Unmatched Inputs

## Idea and Problem

Assign finite immutable items to uniquely keyed canonical groups while preserving order and returning every unmatched input explicitly.

Compatibility is often an exact set of named properties rather than a shared
label. Canonical sorted tuples make that equality rule visible and hashable.
Rejecting duplicate group signatures prevents silent overwrite, while an
explicit unmatched tuple lets the caller choose fail-fast, review, or fallback
behavior.

## When to Use

Use this algorithm when one item belongs to at most one canonical group and
exact property equality is the complete compatibility rule. Group definitions,
items, IDs, and signature fields must fit the documented bounds. Returned
groups follow canonical input order; members and unmatched items follow item
input order.

Use a constraint solver, predicate matrix, or scoring model when compatibility
is partial, asymmetric, fuzzy, ranked, or many-to-many. Do not broaden the key
silently when an unmatched item appears; update the business contract and its
tests deliberately.

## Implementation

```python
import re
from collections.abc import Iterable
from dataclasses import dataclass
from itertools import islice


Signature = tuple[tuple[str, str], ...]
_MAX_GROUPS = 64
_MAX_ITEMS = 1_024
_MAX_SIGNATURE_FIELDS = 16
_IDENTIFIER = re.compile(r"[A-Za-z0-9][A-Za-z0-9._-]{0,63}", re.ASCII)
_PROPERTY_NAME = re.compile(r"[a-z][a-z0-9._-]{0,31}", re.ASCII)
_MAX_PROPERTY_VALUE_CHARACTERS = 64


@dataclass(frozen=True, slots=True)
class CanonicalGroup:
    name: str
    signature: Signature


@dataclass(frozen=True, slots=True)
class CompatibleItem:
    item_id: str
    signature: Signature


@dataclass(frozen=True, slots=True)
class GroupedItems:
    canonical: CanonicalGroup
    members: tuple[CompatibleItem, ...]


@dataclass(frozen=True, slots=True)
class CompatibilityResult:
    groups: tuple[GroupedItems, ...]
    unmatched: tuple[CompatibleItem, ...]


def _bounded_tuple(values: Iterable[object], *, limit: int, name: str) -> tuple[object, ...]:
    if isinstance(values, (str, bytes)):
        raise TypeError(f"{name} must be a non-text iterable")
    result = tuple(islice(values, limit + 1))
    if len(result) > limit:
        raise ValueError(f"{name} exceeds the supported item limit")
    return result


def _validate_identifier(value: object, *, name: str) -> str:
    if not isinstance(value, str) or _IDENTIFIER.fullmatch(value) is None:
        raise ValueError(f"{name} is outside the canonical format")
    return value


def _validate_signature(value: object) -> Signature:
    if not isinstance(value, tuple):
        raise TypeError("a compatibility signature must be a tuple")
    if not 1 <= len(value) <= _MAX_SIGNATURE_FIELDS:
        raise ValueError("signature field count is outside the supported range")

    previous_name: str | None = None
    for field in value:
        if not isinstance(field, tuple) or len(field) != 2:
            raise TypeError("signature fields must be name-value tuples")
        property_name, property_value = field
        if (
            not isinstance(property_name, str)
            or _PROPERTY_NAME.fullmatch(property_name) is None
        ):
            raise ValueError("a signature property name is not canonical")
        if previous_name is not None and property_name <= previous_name:
            raise ValueError("signature property names must be unique and sorted")
        if (
            not isinstance(property_value, str)
            or len(property_value) > _MAX_PROPERTY_VALUE_CHARACTERS
            or not property_value.isascii()
            or not property_value.isprintable()
        ):
            raise ValueError("a signature property value is outside the format")
        previous_name = property_name
    return value


def group_compatible_items(
    canonical_groups: Iterable[CanonicalGroup],
    items: Iterable[CompatibleItem],
) -> CompatibilityResult:
    raw_groups = _bounded_tuple(
        canonical_groups,
        limit=_MAX_GROUPS,
        name="canonical_groups",
    )
    raw_items = _bounded_tuple(items, limit=_MAX_ITEMS, name="items")

    groups: list[CanonicalGroup] = []
    group_names: set[str] = set()
    group_by_signature: dict[Signature, int] = {}
    for value in raw_groups:
        if not isinstance(value, CanonicalGroup):
            raise TypeError("canonical_groups must contain CanonicalGroup values")
        name = _validate_identifier(value.name, name="group name")
        signature = _validate_signature(value.signature)
        if name in group_names:
            raise ValueError("canonical group names must be unique")
        if signature in group_by_signature:
            raise ValueError("canonical compatibility signatures must be unique")
        group_names.add(name)
        group_by_signature[signature] = len(groups)
        groups.append(value)

    members: list[list[CompatibleItem]] = [[] for _ in groups]
    unmatched: list[CompatibleItem] = []
    item_ids: set[str] = set()
    for value in raw_items:
        if not isinstance(value, CompatibleItem):
            raise TypeError("items must contain CompatibleItem values")
        item_id = _validate_identifier(value.item_id, name="item ID")
        signature = _validate_signature(value.signature)
        if item_id in item_ids:
            raise ValueError("item IDs must be unique")
        item_ids.add(item_id)

        group_index = group_by_signature.get(signature)
        if group_index is None:
            unmatched.append(value)
        else:
            members[group_index].append(value)

    return CompatibilityResult(
        groups=tuple(
            GroupedItems(canonical=group, members=tuple(group_members))
            for group, group_members in zip(groups, members, strict=True)
        ),
        unmatched=tuple(unmatched),
    )
```

## Example

```python
portable = (("format", "portable"), ("revision", "2"))
native = (("format", "native"), ("revision", "1"))
unknown = (("format", "legacy"), ("revision", "1"))

result = group_compatible_items(
    (
        CanonicalGroup("portable-v2", portable),
        CanonicalGroup("native-v1", native),
    ),
    (
        CompatibleItem("item-a", native),
        CompatibleItem("item-b", portable),
        CompatibleItem("item-c", unknown),
        CompatibleItem("item-d", native),
    ),
)

assert (
    tuple(group.canonical.name for group in result.groups),
    tuple(tuple(item.item_id for item in group.members) for group in result.groups),
    tuple(item.item_id for item in result.unmatched),
) == (
    ("portable-v2", "native-v1"),
    (("item-b",), ("item-a", "item-d")),
    ("item-c",),
)
```

## Trade-offs and Limitations

Runtime is linear in groups, items, and their bounded signature sizes, with a
hash lookup per item. The function keeps all inputs and output membership in
memory, so it is not a streaming join. Conservative string-only signatures
make results portable and immutable but exclude richer typed property values.

Exact equality is only as correct as the signature design. Omitting a relevant
property can combine incompatible items, while including an irrelevant one can
create unnecessary unmatched results. The algorithm does not choose a fallback,
merge overlapping groups, copy external payloads, or prove that a group is
safe; it groups the validated immutable records it receives.

## Related Snippets

<!-- catalog:related:start -->
- [Route Items by Ordered Text Prefixes](route-items-by-ordered-text-prefixes.md)
- [Select One Record per Key with an Explicit Ranking Rule](select-one-record-per-key-with-an-explicit-ranking-rule.md)
- [Apportion a Non-Negative Integer Total Without Rounding Drift](../algorithms-data-structures/apportion-a-non-negative-integer-total-without-rounding-drift.md)
<!-- catalog:related:end -->

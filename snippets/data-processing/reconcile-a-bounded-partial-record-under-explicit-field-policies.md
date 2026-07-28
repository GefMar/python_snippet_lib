---
title: "Reconcile a Bounded Partial Record Under Explicit Field Policies"
snippet_type: pattern
use_cases:
  - concurrency-control
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../python-language/apply-partial-dataclass-updates-with-an-omitted-value-sentinel.md
  - ../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md
  - ../reliability-resilience/plan-a-versioned-transition-for-the-current-workflow-attempt.md
---

# Reconcile a Bounded Partial Record Under Explicit Field Policies

## Idea and Problem

Validate an absent or present record and its same-key partial update completely, then return an immutable create, update, unchanged, or revision-conflict plan.

Each patch field carries an explicit `present` bit. An omitted field therefore
cannot be confused with a supplied `None`, `False`, zero, empty string, or empty
collection. Supplied ordinary fields replace their whole top-level value;
`attributes` replaces its whole JSON-compatible tree rather than merging it.
Only `relation_ids` has a different policy: retain current order, then append
each not-yet-seen supplied ID in first-seen order.

## When to Use

Use this pattern at an in-memory boundary that has one bounded snapshot, one
same-key update, and an expected revision. Revision zero means "expect absent";
present records use revisions from one upward. Creation fills omitted fields
with the documented defaults `None`, `False`, `0`, an empty attribute object,
and no relations.

This is suitable when callers need a deterministic plan and field-name audit
before a separate owner applies anything. Use a schema-specific migration or a
configuration merge implementation when nested fields require recursive merge,
deletion markers, or per-path policy.

## Implementation

```python
import json
import math
import re
from collections.abc import Callable
from dataclasses import dataclass, field, replace
from enum import StrEnum
from typing import Literal, cast

_MAX_REVISION = 2**63 - 1
_MAX_NOTE_BYTES = 512
_MAX_RELATIONS = 32
_MAX_DEPTH = 8
_MAX_NODES = 256
_MAX_ATTRIBUTE_BYTES = 24 * 1_024
_MIN_JSON_INTEGER = -(2**53) + 1
_MAX_JSON_INTEGER = 2**53 - 1
_MIN_QUOTA = -1_000_000
_MAX_QUOTA = 1_000_000
_KEY = re.compile(r"[a-z][a-z0-9-]{0,47}", re.ASCII).fullmatch
_RELATION_ID = re.compile(r"[a-z][a-z0-9._-]{0,47}", re.ASCII).fullmatch


@dataclass(frozen=True, slots=True)
class FrozenArray:
    items: tuple[FrozenJSON, ...]


@dataclass(frozen=True, slots=True)
class FrozenObject:
    members: tuple[tuple[str, FrozenJSON], ...]


type FrozenJSON = None | bool | int | float | str | FrozenArray | FrozenObject
type RecordField = Literal["note", "enabled", "quota", "attributes", "relation_ids"]
_FIELDS: tuple[RecordField, ...] = (
    "note",
    "enabled",
    "quota",
    "attributes",
    "relation_ids",
)


@dataclass(frozen=True, slots=True)
class FrozenAttributes:
    root: FrozenJSON
    encoded: bytes = field(repr=False)


@dataclass(frozen=True, slots=True)
class FieldValue[T]:
    present: bool
    value: T | None = None


@dataclass(frozen=True, slots=True)
class AbsentRecord:
    key: str


@dataclass(frozen=True, slots=True)
class RecordSnapshot:
    key: str
    revision: int
    note: str | None
    enabled: bool
    quota: int
    attributes: object
    relation_ids: tuple[str, ...]


type CurrentRecord = AbsentRecord | RecordSnapshot


@dataclass(frozen=True, slots=True)
class PartialRecordUpdate:
    key: str
    note: FieldValue[str | None] = FieldValue(False)
    enabled: FieldValue[bool] = FieldValue(False)
    quota: FieldValue[int] = FieldValue(False)
    attributes: FieldValue[object] = FieldValue(False)
    relation_ids: FieldValue[tuple[str, ...]] = FieldValue(False)


class ReconcileAction(StrEnum):
    CREATE = "create"
    UPDATE = "update"
    UNCHANGED = "unchanged"
    REVISION_CONFLICT = "revision_conflict"


@dataclass(frozen=True, slots=True)
class ReconciliationPlan:
    action: ReconcileAction
    current: CurrentRecord
    planned: CurrentRecord
    changed_fields: tuple[RecordField, ...]


def _key(value: object, *, field: str) -> str:
    if type(value) is not str or _KEY(value) is None:
        raise ValueError(f"{field} must be a conservative ASCII key")
    return value


def _revision(value: object, *, field: str, allow_zero: bool) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact integer")
    minimum = 0 if allow_zero else 1
    if not minimum <= value <= _MAX_REVISION:
        raise ValueError(f"{field} is outside the supported revision range")
    return value


def _text_bytes(value: str, *, field: str, maximum: int) -> str:
    try:
        size = len(value.encode("utf-8"))
    except UnicodeEncodeError as error:
        raise ValueError(f"{field} contains invalid Unicode text") from error
    if size > maximum:
        raise ValueError(f"{field} exceeds its UTF-8 byte limit")
    return value


def _note(value: object) -> str | None:
    if value is None:
        return None
    if type(value) is not str:
        raise TypeError("note must be an exact string or None")
    return _text_bytes(value, field="note", maximum=_MAX_NOTE_BYTES)


def _enabled(value: object) -> bool:
    if type(value) is not bool:
        raise TypeError("enabled must be an exact boolean")
    return value


def _quota(value: object) -> int:
    if type(value) is not int:
        raise TypeError("quota must be an exact integer")
    if not _MIN_QUOTA <= value <= _MAX_QUOTA:
        raise ValueError("quota is outside the supported range")
    return value


def _relations(value: object, *, require_unique: bool) -> tuple[str, ...]:
    if type(value) is not tuple:
        raise TypeError("relation_ids must be an exact tuple")
    if len(value) > _MAX_RELATIONS:
        raise ValueError("relation_ids exceeds the supported item limit")
    result: list[str] = []
    seen: set[str] = set()
    for item in value:
        if type(item) is not str or _RELATION_ID(item) is None:
            raise ValueError("relation_ids contains an invalid identifier")
        if require_unique and item in seen:
            raise ValueError("snapshot relation_ids must be unique")
        seen.add(item)
        result.append(item)
    return tuple(result)


def _encoded(value: object) -> bytes:
    try:
        encoded = json.dumps(
            value,
            allow_nan=False,
            ensure_ascii=False,
            separators=(",", ":"),
        ).encode("utf-8")
    except (UnicodeEncodeError, ValueError) as error:
        raise ValueError("attributes contains invalid JSON text") from error
    if len(encoded) > _MAX_ATTRIBUTE_BYTES:
        raise ValueError("attributes exceeds the encoded byte limit")
    return encoded


def _attributes(value: object) -> FrozenAttributes:
    nodes = 0
    active: set[int] = set()
    seen_containers: set[int] = set()

    def enter(container: object) -> int:
        identity = id(container)
        if identity in active:
            raise ValueError("attributes contains a container cycle")
        if identity in seen_containers:
            raise ValueError("attributes contains an aliased container")
        seen_containers.add(identity)
        active.add(identity)
        return identity

    def visit(node: object, *, depth: int) -> tuple[FrozenJSON, bytes]:
        nonlocal nodes
        nodes += 1
        if nodes > _MAX_NODES:
            raise ValueError("attributes exceeds the node limit")
        if depth > _MAX_DEPTH:
            raise ValueError("attributes exceeds the depth limit")

        if node is None or type(node) is bool:
            return node, _encoded(node)
        if type(node) is int:
            if not _MIN_JSON_INTEGER <= node <= _MAX_JSON_INTEGER:
                raise ValueError("attributes contains an out-of-range integer")
            return node, _encoded(node)
        if type(node) is float:
            if not math.isfinite(node):
                raise ValueError("attributes contains a non-finite number")
            return node, _encoded(node)
        if type(node) is str:
            return node, _encoded(node)

        if type(node) is list or type(node) is FrozenArray:
            identity = enter(node)
            try:
                raw_items = node if type(node) is list else node.items
                if type(raw_items) not in (list, tuple):
                    raise TypeError("frozen array items must be an exact tuple")
                visited = [visit(item, depth=depth + 1) for item in raw_items]
                frozen = FrozenArray(tuple(item for item, _data in visited))
                data = b"[" + b",".join(data for _item, data in visited) + b"]"
                if len(data) > _MAX_ATTRIBUTE_BYTES:
                    raise ValueError("attributes exceeds the encoded byte limit")
                return frozen, data
            finally:
                active.remove(identity)

        if type(node) is dict or type(node) is FrozenObject:
            identity = enter(node)
            try:
                raw_members = node.items() if type(node) is dict else node.members
                if type(node) is FrozenObject and type(raw_members) is not tuple:
                    raise TypeError("frozen object members must be an exact tuple")
                visited: list[tuple[str, FrozenJSON, bytes]] = []
                keys: set[str] = set()
                for member in raw_members:
                    if type(member) is not tuple or len(member) != 2:
                        raise TypeError("attribute object members must be exact pairs")
                    key, item = member
                    if type(key) is not str:
                        raise TypeError("attribute object keys must be exact strings")
                    if key in keys:
                        raise ValueError("attribute object keys must be unique")
                    keys.add(key)
                    frozen, data = visit(item, depth=depth + 1)
                    visited.append((key, frozen, data))
                visited.sort(key=lambda member: member[0])
                data = (
                    b"{"
                    + b",".join(
                        _encoded(key) + b":" + item_data for key, _item, item_data in visited
                    )
                    + b"}"
                )
                if len(data) > _MAX_ATTRIBUTE_BYTES:
                    raise ValueError("attributes exceeds the encoded byte limit")
                return (
                    FrozenObject(tuple((key, item) for key, item, _data in visited)),
                    data,
                )
            finally:
                active.remove(identity)

        raise TypeError("attributes must contain only exact JSON-compatible values")

    if type(value) is FrozenAttributes:
        value = value.root
    frozen, encoded = visit(value, depth=0)
    return FrozenAttributes(frozen, encoded)


def _field[T](
    value: object,
    *,
    name: str,
    validate: Callable[[object], T],
) -> FieldValue[T]:
    if type(value) is not FieldValue:
        raise TypeError(f"update.{name} must be an exact FieldValue")
    if type(value.present) is not bool:
        raise TypeError(f"update.{name}.present must be an exact boolean")
    if not value.present and value.value is not None:
        raise ValueError(f"omitted update.{name} must carry None")
    if not value.present:
        return FieldValue(False)
    return FieldValue(True, validate(value.value))


def _validated_update(value: object) -> PartialRecordUpdate:
    if type(value) is not PartialRecordUpdate:
        raise TypeError("update must be an exact PartialRecordUpdate")
    return PartialRecordUpdate(
        key=_key(value.key, field="update.key"),
        note=_field(value.note, name="note", validate=_note),
        enabled=_field(value.enabled, name="enabled", validate=_enabled),
        quota=_field(value.quota, name="quota", validate=_quota),
        attributes=_field(value.attributes, name="attributes", validate=_attributes),
        relation_ids=_field(
            value.relation_ids,
            name="relation_ids",
            validate=lambda item: _relations(item, require_unique=False),
        ),
    )


def _validated_current(value: object) -> CurrentRecord:
    if type(value) is AbsentRecord:
        return AbsentRecord(_key(value.key, field="current.key"))
    if type(value) is not RecordSnapshot:
        raise TypeError("current must be an exact AbsentRecord or RecordSnapshot")
    return RecordSnapshot(
        key=_key(value.key, field="current.key"),
        revision=_revision(value.revision, field="current.revision", allow_zero=False),
        note=_note(value.note),
        enabled=_enabled(value.enabled),
        quota=_quota(value.quota),
        attributes=_attributes(value.attributes),
        relation_ids=_relations(value.relation_ids, require_unique=True),
    )


def _selected[T](field: FieldValue[T], default: T) -> T:
    return cast("T", field.value) if field.present else default


def _stable_union(
    current: tuple[str, ...],
    supplied: tuple[str, ...],
) -> tuple[str, ...]:
    result = list(current)
    seen = set(current)
    for relation_id in supplied:
        if relation_id not in seen:
            seen.add(relation_id)
            result.append(relation_id)
            if len(result) > _MAX_RELATIONS:
                raise ValueError("relation union exceeds the supported item limit")
    return tuple(result)


def reconcile_partial_record(
    current: CurrentRecord,
    update: PartialRecordUpdate,
    *,
    expected_revision: int,
) -> ReconciliationPlan:
    prior = _validated_current(current)
    patch = _validated_update(update)
    expected = _revision(
        expected_revision,
        field="expected_revision",
        allow_zero=True,
    )
    if prior.key != patch.key:
        raise ValueError("current and update keys must match")

    if type(prior) is AbsentRecord:
        base = RecordSnapshot(prior.key, 0, None, False, 0, _attributes({}), ())
        next_revision = 1
    else:
        base = prior
        next_revision = prior.revision

    supplied_relations = _selected(patch.relation_ids, ())
    candidate = RecordSnapshot(
        key=patch.key,
        revision=next_revision,
        note=_selected(patch.note, base.note),
        enabled=_selected(patch.enabled, base.enabled),
        quota=_selected(patch.quota, base.quota),
        attributes=_selected(patch.attributes, base.attributes),
        relation_ids=(
            _stable_union(base.relation_ids, supplied_relations)
            if patch.relation_ids.present
            else base.relation_ids
        ),
    )

    if type(prior) is AbsentRecord:
        if expected != 0:
            return ReconciliationPlan(ReconcileAction.REVISION_CONFLICT, prior, prior, ())
        return ReconciliationPlan(ReconcileAction.CREATE, prior, candidate, _FIELDS)

    changed = tuple(name for name in _FIELDS if getattr(prior, name) != getattr(candidate, name))
    if prior.revision != expected:
        return ReconciliationPlan(ReconcileAction.REVISION_CONFLICT, prior, prior, ())
    if not changed:
        return ReconciliationPlan(ReconcileAction.UNCHANGED, prior, prior, ())
    if prior.revision == _MAX_REVISION:
        raise OverflowError("a changed record cannot advance beyond the revision limit")
    planned = replace(candidate, revision=prior.revision + 1)
    return ReconciliationPlan(ReconcileAction.UPDATE, prior, planned, changed)
```

## Example

```python
current = RecordSnapshot(
    key="parcel-elm",
    revision=7,
    note="waiting",
    enabled=True,
    quota=12,
    attributes={"lane": "west", "labels": ["fragile"]},
    relation_ids=("link-cedar",),
)
update = PartialRecordUpdate(
    key="parcel-elm",
    note=FieldValue(True, None),
    enabled=FieldValue(True, False),
    quota=FieldValue(True, 0),
    attributes=FieldValue(True, {"caption": "", "labels": []}),
    relation_ids=FieldValue(True, ("link-cobalt", "link-cedar", "link-cobalt")),
)

first = reconcile_partial_record(current, update, expected_revision=7)
second = reconcile_partial_record(first.planned, update, expected_revision=8)
planned = cast(RecordSnapshot, first.planned)
attributes = cast(FrozenAttributes, planned.attributes)

assert (
    first.action,
    first.changed_fields,
    planned.revision,
    planned.note,
    planned.enabled,
    planned.quota,
    attributes.root,
    planned.relation_ids,
    second.action,
    second.changed_fields,
) == (
    ReconcileAction.UPDATE,
    ("note", "enabled", "quota", "attributes", "relation_ids"),
    8,
    None,
    False,
    0,
    FrozenObject((("caption", ""), ("labels", FrozenArray(())))),
    ("link-cedar", "link-cobalt"),
    ReconcileAction.UNCHANGED,
    (),
)
```

## Trade-offs and Limitations

The attribute boundary accepts exact JSON scalars, `dict` objects, and `list`
arrays, plus this snippet's frozen output. It allows at most 256 value nodes,
depth 8 from a depth-zero root, and 24 KiB after compact JSON encoding. Integers
stay within the interoperable 53-bit range; floats must be finite. Container
identity tracking rejects both cycles and repeated aliases, and `FrozenObject`
input makes duplicate keys representable and rejectable. A normal Python
`dict` has already collapsed duplicates, so a text JSON decoder must reject
them before constructing that dictionary.

Notes are capped at 512 UTF-8 bytes, relation inputs and their union at 32 IDs,
quotas at plus or minus one million, and revisions at the nonnegative signed
64-bit maximum. A supplied empty relation tuple adds nothing; this union policy
does not clear existing IDs. Changed fields follow schema order. Conflicts have
an empty audit because the plan changes nothing, even though the fully
validated update may describe different values.

This planner mutates no input and performs no clock read, persistence, ORM
operation, transaction, retry, identity generation, external relation-graph
traversal, or relation-integrity check. Applying a plan still requires an
external atomic revision check. The whole-tree attribute replacement is
intentionally not a generic recursive configuration merge.

## Related Snippets

<!-- catalog:related:start -->
- [Apply Partial Dataclass Updates with an Omitted-Value Sentinel](../python-language/apply-partial-dataclass-updates-with-an-omitted-value-sentinel.md)
- [Make a Defensive Copy at a Mutable Input Boundary](../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md)
- [Plan a Versioned Transition for the Current Workflow Attempt](../reliability-resilience/plan-a-versioned-transition-for-the-current-workflow-attempt.md)
<!-- catalog:related:end -->

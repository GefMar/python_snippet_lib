---
title: "Plan a Bounded JSON Reconciliation Under Explicit Path Policies"
snippet_type: pattern
use_cases:
  - configuration
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - render-a-stable-unified-diff-for-nested-json-values.md
  - fingerprint-a-set-like-json-array-deterministically.md
  - elide-nested-mapping-values-that-match-explicit-defaults.md
---

# Plan a Bounded JSON Reconciliation Under Explicit Path Policies

## Idea and Problem

Compare optional current and required desired JSON-object snapshots under a small, explicit path policy.

An absent current snapshot produces `CREATE`. Present snapshots produce
`NO_OP` or `UPDATE` plus immutable differences containing only an unambiguous
tuple path and a stable kind, never raw values. Exact ignored paths cover
server-managed subtrees, while only explicitly declared arrays compare as
unordered sets of unique scalars. Every other array remains ordered.

## When to Use

Use this planner before a separate owner creates or replaces a bounded JSON
resource. It preserves missing members versus JSON `null`, walks object keys
deterministically, and makes exceptional comparison semantics reviewable as
data rather than hidden predicates.

Ignored paths are a narrow trust boundary: a difference at that exact path,
including its subtree, cannot trigger an update. Keep the list minimal and
version-controlled. Use a text diff for human formatting review, a shallow
mapping delta for flat records, or a formal patch format when edits must be
applied rather than merely planned.

## Implementation

```python
import math
from dataclasses import dataclass
from enum import StrEnum

type PathComponent = str | int
type JSONPath = tuple[PathComponent, ...]

_MIN_JSON_INTEGER = -(1 << 63)
_MAX_JSON_INTEGER = (1 << 63) - 1
_MAX_DEPTH = 12
_MAX_NODES = 1_024
_MAX_CONTAINER_ENTRIES = 128
_MAX_TEXT_BYTES = 4 * 1_024
_MAX_TOTAL_TEXT_BYTES = 64 * 1_024
_MAX_POLICY_PATHS = 64
_MAX_POLICY_TEXT_BYTES = 8 * 1_024
_MAX_UNORDERED_LIST_SIZE = 64
_MAX_DIFFERENCES = 256
_MISSING = object()


class ReconciliationAction(StrEnum):
    CREATE = "create"
    NO_OP = "no_op"
    UPDATE = "update"


class DifferenceKind(StrEnum):
    CURRENT_MISSING = "current_missing"
    DESIRED_MISSING = "desired_missing"
    TYPE_CHANGED = "type_changed"
    SCALAR_CHANGED = "scalar_changed"
    UNORDERED_SET_CHANGED = "unordered_set_changed"


@dataclass(frozen=True, slots=True)
class PathPolicy:
    ignored_paths: tuple[JSONPath, ...] = ()
    unordered_scalar_paths: tuple[JSONPath, ...] = ()


@dataclass(frozen=True, slots=True)
class JSONDifference:
    path: JSONPath
    kind: DifferenceKind


@dataclass(frozen=True, slots=True)
class ReconciliationPlan:
    action: ReconciliationAction
    differences: tuple[JSONDifference, ...]


def _path_sort_key(path: JSONPath) -> tuple[tuple[int, str | int], ...]:
    return tuple(
        (0, component) if type(component) is str else (1, component)
        for component in path
    )


def _validate_documents(current: object | None, desired: object) -> None:
    if current is not None and type(current) is not dict:
        raise TypeError("current must be absent or an exact JSON object")
    if type(desired) is not dict:
        raise TypeError("desired must be an exact JSON object")

    nodes = 0
    text_bytes = 0
    active: set[int] = set()
    seen_containers: set[int] = set()

    def account_text(value: str, *, name: str) -> None:
        nonlocal text_bytes
        try:
            size = len(value.encode("utf-8"))
        except UnicodeEncodeError as error:
            raise ValueError(f"{name} contains text that is not valid UTF-8") from error
        if size > _MAX_TEXT_BYTES:
            raise ValueError(f"{name} contains text above the per-value byte limit")
        text_bytes += size
        if text_bytes > _MAX_TOTAL_TEXT_BYTES:
            raise ValueError("documents exceed the aggregate UTF-8 text limit")

    def visit(node: object, *, depth: int, name: str) -> None:
        nonlocal nodes
        nodes += 1
        if nodes > _MAX_NODES:
            raise ValueError("documents exceed the aggregate node limit")
        if depth > _MAX_DEPTH:
            raise ValueError(f"{name} exceeds the nesting-depth limit")

        if node is None or type(node) is bool:
            return
        if type(node) is int:
            if not _MIN_JSON_INTEGER <= node <= _MAX_JSON_INTEGER:
                raise ValueError(f"{name} contains an integer outside signed 64-bit range")
            return
        if type(node) is float:
            if not math.isfinite(node):
                raise ValueError(f"{name} contains a non-finite float")
            return
        if type(node) is str:
            account_text(node, name=name)
            return
        if type(node) not in (list, dict):
            raise TypeError(f"{name} contains a value that is not an exact JSON type")
        if len(node) > _MAX_CONTAINER_ENTRIES:
            raise ValueError(f"{name} contains an oversized container")

        identity = id(node)
        if identity in active:
            raise ValueError("documents contain a container cycle")
        if identity in seen_containers:
            raise ValueError("documents contain a shared container")
        seen_containers.add(identity)
        active.add(identity)
        try:
            if type(node) is list:
                for item in node:
                    visit(item, depth=depth + 1, name=name)
            else:
                for key in node:
                    if type(key) is not str:
                        raise TypeError(f"{name} object keys must be exact strings")
                    account_text(key, name=name)
                for key in sorted(node):
                    visit(node[key], depth=depth + 1, name=name)
        finally:
            active.remove(identity)

    if current is not None:
        visit(current, depth=0, name="current")
    visit(desired, depth=0, name="desired")


def _validate_policy(policy: object) -> tuple[frozenset[JSONPath], frozenset[JSONPath]]:
    if type(policy) is not PathPolicy:
        raise TypeError("policy must be an exact PathPolicy")
    if type(policy.ignored_paths) is not tuple:
        raise TypeError("ignored_paths must be an exact tuple")
    if type(policy.unordered_scalar_paths) is not tuple:
        raise TypeError("unordered_scalar_paths must be an exact tuple")

    declared = (
        ("ignored", policy.ignored_paths),
        ("unordered", policy.unordered_scalar_paths),
    )
    if sum(len(paths) for _kind, paths in declared) > _MAX_POLICY_PATHS:
        raise ValueError("policy path count exceeds the supported limit")

    policy_text_bytes = 0
    paths_by_kind: dict[str, set[JSONPath]] = {"ignored": set(), "unordered": set()}
    all_paths: list[JSONPath] = []
    for kind, paths in declared:
        for path in paths:
            if type(path) is not tuple or not 1 <= len(path) <= _MAX_DEPTH:
                raise ValueError("policy paths must be non-empty bounded exact tuples")
            validated: list[PathComponent] = []
            for component in path:
                if type(component) is str:
                    try:
                        size = len(component.encode("utf-8"))
                    except UnicodeEncodeError as error:
                        raise ValueError("policy paths contain invalid UTF-8 text") from error
                    if size > _MAX_TEXT_BYTES:
                        raise ValueError("a policy path component exceeds its byte limit")
                    policy_text_bytes += size
                elif type(component) is int:
                    if not 0 <= component < _MAX_CONTAINER_ENTRIES:
                        raise ValueError("a policy array index is outside the supported range")
                else:
                    raise TypeError("policy path components must be exact strings or integers")
                validated.append(component)
            candidate = tuple(validated)
            if candidate in all_paths:
                raise ValueError("policy paths must be unique")
            paths_by_kind[kind].add(candidate)
            all_paths.append(candidate)

    if policy_text_bytes > _MAX_POLICY_TEXT_BYTES:
        raise ValueError("policy paths exceed the aggregate UTF-8 text limit")
    for index, left in enumerate(all_paths):
        for right in all_paths[index + 1 :]:
            shortest = min(len(left), len(right))
            if left[:shortest] == right[:shortest]:
                raise ValueError("policy paths must not overlap")
    return frozenset(paths_by_kind["ignored"]), frozenset(paths_by_kind["unordered"])


def _value_at(document: object, path: JSONPath) -> object:
    current = document
    for component in path:
        if type(current) is dict and type(component) is str:
            if component not in current:
                return _MISSING
            current = current[component]
        elif type(current) is list and type(component) is int and component < len(current):
            current = current[component]
        else:
            return _MISSING
    return current


def _scalar_key(value: object) -> tuple[str, object]:
    if value is None:
        return "null", ""
    if type(value) is bool:
        return "boolean", value
    if type(value) is int:
        return "integer", value
    if type(value) is float:
        return "float", value.hex()
    if type(value) is str:
        return "string", value
    raise TypeError("unordered arrays must contain only JSON scalars")


def _validate_unordered_arrays(
    current: object | None,
    desired: object,
    unordered_paths: frozenset[JSONPath],
) -> None:
    documents = (("desired", desired),) if current is None else (
        ("current", current),
        ("desired", desired),
    )
    for path in sorted(unordered_paths, key=_path_sort_key):
        for name, document in documents:
            value = _value_at(document, path)
            if value is _MISSING:
                continue
            if type(value) is not list:
                raise TypeError(f"{name} unordered path does not select an exact array")
            if len(value) > _MAX_UNORDERED_LIST_SIZE:
                raise ValueError(f"{name} unordered array exceeds the supported size")
            keys: set[tuple[str, object]] = set()
            for item in value:
                key = _scalar_key(item)
                if key in keys:
                    raise ValueError(f"{name} unordered array contains a duplicate scalar")
                keys.add(key)


def plan_json_reconciliation(
    current: object | None,
    desired: object,
    policy: PathPolicy,
    *,
    max_differences: int = _MAX_DIFFERENCES,
) -> ReconciliationPlan:
    if type(max_differences) is not int:
        raise TypeError("max_differences must be an exact integer")
    if not 0 <= max_differences <= _MAX_DIFFERENCES:
        raise ValueError("max_differences is outside the supported range")

    ignored_paths, unordered_paths = _validate_policy(policy)
    _validate_documents(current, desired)
    _validate_unordered_arrays(current, desired, unordered_paths)
    if current is None:
        return ReconciliationPlan(ReconciliationAction.CREATE, ())

    differences: list[JSONDifference] = []

    def add(path: JSONPath, kind: DifferenceKind) -> None:
        if len(differences) >= max_differences:
            raise ValueError("difference count exceeds max_differences")
        differences.append(JSONDifference(path, kind))

    def compare(left: object, right: object, path: JSONPath) -> None:
        if path in ignored_paths:
            return
        if left is _MISSING:
            add(path, DifferenceKind.CURRENT_MISSING)
            return
        if right is _MISSING:
            add(path, DifferenceKind.DESIRED_MISSING)
            return
        if type(left) is not type(right):
            add(path, DifferenceKind.TYPE_CHANGED)
            return
        if path in unordered_paths:
            left_keys = {_scalar_key(item) for item in left}
            right_keys = {_scalar_key(item) for item in right}
            if left_keys != right_keys:
                add(path, DifferenceKind.UNORDERED_SET_CHANGED)
            return
        if type(left) is dict:
            for key in sorted(left.keys() | right.keys()):
                compare(
                    left[key] if key in left else _MISSING,
                    right[key] if key in right else _MISSING,
                    (*path, key),
                )
            return
        if type(left) is list:
            shared_length = min(len(left), len(right))
            for index in range(shared_length):
                compare(left[index], right[index], (*path, index))
            for index in range(shared_length, max(len(left), len(right))):
                compare(
                    left[index] if index < len(left) else _MISSING,
                    right[index] if index < len(right) else _MISSING,
                    (*path, index),
                )
            return
        if _scalar_key(left) != _scalar_key(right):
            add(path, DifferenceKind.SCALAR_CHANGED)

    compare(current, desired, ())
    action = ReconciliationAction.UPDATE if differences else ReconciliationAction.NO_OP
    return ReconciliationPlan(action, tuple(differences))
```

## Example

```python
policy = PathPolicy(
    ignored_paths=(("metadata", "server_revision"),),
    unordered_scalar_paths=(("spec", "tags"),),
)
current = {
    "metadata": {"server_revision": 7},
    "spec": {"replicas": 3, "tags": ["green", "blue"]},
}
equivalent_desired = {
    "metadata": {"server_revision": 8},
    "spec": {"replicas": 3, "tags": ["blue", "green"]},
}
changed_desired = {
    "metadata": {"server_revision": 8},
    "spec": {"replicas": 4, "tags": ["blue", "green"]},
}

no_op = plan_json_reconciliation(current, equivalent_desired, policy)
update = plan_json_reconciliation(current, changed_desired, policy)
create = plan_json_reconciliation(None, changed_desired, policy)

assert (no_op.action, no_op.differences) == (ReconciliationAction.NO_OP, ())
assert update == ReconciliationPlan(
    ReconciliationAction.UPDATE,
    (
        JSONDifference(
            ("spec", "replicas"),
            DifferenceKind.SCALAR_CHANGED,
        ),
    ),
)
assert create == ReconciliationPlan(ReconciliationAction.CREATE, ())
```

## Trade-offs and Limitations

Preflight visits both documents before comparison and rejects shared or cyclic
containers, oversized structures, invalid UTF-8, non-finite floats, and
non-exact JSON types. The node and text budgets are aggregate across both
snapshots. Comparison is linear in bounded structure size apart from sorting
object keys; unordered arrays require additional scalar-key sets. Reaching the
difference cap rejects the plan instead of returning a misleading truncation.

Numeric comparison is deliberately Python-type-sensitive: integers and floats
are different, and hexadecimal float identities preserve signed zero. Strings
are not Unicode-normalized. An ignored node is still fully preflighted, but its
contents cannot cause `UPDATE`; policy review therefore remains essential. The
planner performs no JSON parsing, text diffing, remote I/O, authentication,
retry loop, patch construction, or patch execution.

## Related Snippets

<!-- catalog:related:start -->
- [Render a Stable Unified Diff for Nested JSON Values](render-a-stable-unified-diff-for-nested-json-values.md)
- [Fingerprint a Set-Like JSON Array Deterministically](fingerprint-a-set-like-json-array-deterministically.md)
- [Elide Nested Mapping Values That Match Explicit Defaults](elide-nested-mapping-values-that-match-explicit-defaults.md)
<!-- catalog:related:end -->

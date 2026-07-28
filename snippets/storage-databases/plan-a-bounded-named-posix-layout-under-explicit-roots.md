---
title: "Plan a Bounded Named POSIX Layout Under Explicit Roots"
snippet_type: algorithm
use_cases:
  - data-transformation
  - persistence
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - map-a-namespaced-posix-path-beneath-a-logical-local-root.md
  - build-a-bounded-file-manifest-with-internal-symlink-aliases.md
  - ../security-privacy/validate-a-conservative-unicode-filename-component.md
---

# Plan a Bounded Named POSIX Layout Under Explicit Roots

## Idea and Problem

Turn one immutable profile of logical roots and named roles into a complete, ordered POSIX layout plan.

Validating one rendered path at a time misses conflicts created through
overlapping roots. This planner first validates every declaration, renders every
candidate, and then rejects collisions and any path below a role declared as a
file. Only a globally consistent tuple of frozen records is returned.

## When to Use

Use this algorithm when a small trusted profile declares several related paths
under explicit logical POSIX roots. Role suffixes are exact tuples of
conservative ASCII components, and each role states whether its rendered path
represents a file or directory. Declaration order becomes output order.

The roots and suffixes are lexical configuration, not host filesystem paths
that this function discovers. Raw strings are not coerced into enum members,
lists are not accepted in place of tuples, and path-like objects are not
converted implicitly.

## Implementation

```python
import re
from dataclasses import dataclass
from enum import StrEnum
from pathlib import PurePosixPath

_MAX_ROOTS = 6
_MAX_ROLES = 24
_MAX_ROOT_DEPTH = 10
_MAX_SUFFIX_DEPTH = 7
_MAX_RENDERED_DEPTH = 14
_MAX_COMPONENT_BYTES = 72
_MAX_PATH_BYTES = 640
_MAX_PROFILE_COMPONENTS = 144
_NAME = re.compile(r"[a-z][a-z0-9_]{0,31}", re.ASCII).fullmatch
_SUFFIX_COMPONENT = re.compile(r"[A-Za-z0-9][A-Za-z0-9._-]{0,71}", re.ASCII).fullmatch


class LayoutPlanError(ValueError):
    pass


class RoleKind(StrEnum):
    DIRECTORY = "directory"
    FILE = "file"


@dataclass(frozen=True, slots=True)
class LogicalRoot:
    name: str
    path: str


@dataclass(frozen=True, slots=True)
class RoleDeclaration:
    name: str
    root: str
    suffix: tuple[str, ...]
    kind: RoleKind


@dataclass(frozen=True, slots=True)
class LayoutProfile:
    roots: tuple[LogicalRoot, ...]
    roles: tuple[RoleDeclaration, ...]


@dataclass(frozen=True, slots=True)
class PlannedRolePath:
    role: str
    kind: RoleKind
    path: PurePosixPath


def _utf8_size(value: str, error_message: str) -> int:
    try:
        return len(value.encode("utf-8"))
    except UnicodeEncodeError:
        raise LayoutPlanError(error_message) from None


def _checked_name(value: object, label: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{label} must be an exact string")
    if _NAME(value) is None:
        raise LayoutPlanError(f"{label} has invalid syntax")
    return value


def _checked_root_path(value: object) -> tuple[PurePosixPath, int]:
    if type(value) is not str:
        raise TypeError("logical root paths must be exact strings")
    size = _utf8_size(value, "a logical root path is not valid UTF-8")
    if not 1 <= size <= _MAX_PATH_BYTES:
        raise LayoutPlanError("a logical root path exceeds its byte bound")
    if not value.startswith("/") or value.startswith("//"):
        raise LayoutPlanError("logical root paths must be canonical absolute POSIX paths")
    if value != "/" and value.endswith("/"):
        raise LayoutPlanError("logical root paths must not end with a separator")
    if "\x00" in value or "\\" in value:
        raise LayoutPlanError("a logical root path contains a forbidden character")

    parts = () if value == "/" else tuple(value[1:].split("/"))
    if len(parts) > _MAX_ROOT_DEPTH:
        raise LayoutPlanError("a logical root path exceeds its depth bound")
    if any(part in ("", ".", "..") for part in parts):
        raise LayoutPlanError("a logical root path is not canonical")
    if any(
        _utf8_size(part, "a logical root component is not valid UTF-8") > _MAX_COMPONENT_BYTES
        for part in parts
    ):
        raise LayoutPlanError("a logical root component exceeds its byte bound")

    path = PurePosixPath(value)
    if path.as_posix() != value:
        raise LayoutPlanError("a logical root path is not canonical")
    return path, len(parts)


def _checked_suffix(value: object) -> tuple[str, ...]:
    if type(value) is not tuple:
        raise TypeError("role suffixes must be exact tuples")
    if not 1 <= len(value) <= _MAX_SUFFIX_DEPTH:
        raise LayoutPlanError("a role suffix exceeds its depth bound")

    checked: list[str] = []
    for component in value:
        if type(component) is not str:
            raise TypeError("role suffix components must be exact strings")
        if component in (".", "..") or "/" in component or "\\" in component:
            raise LayoutPlanError("role suffixes must be relative and non-traversing")
        if _SUFFIX_COMPONENT(component) is None:
            raise LayoutPlanError("a role suffix component has invalid syntax")
        if _utf8_size(component, "a role suffix component is not valid UTF-8") > (
            _MAX_COMPONENT_BYTES
        ):
            raise LayoutPlanError("a role suffix component exceeds its byte bound")
        checked.append(component)
    return tuple(checked)


def _is_strict_prefix(ancestor: PurePosixPath, descendant: PurePosixPath) -> bool:
    ancestor_parts = ancestor.parts
    descendant_parts = descendant.parts
    return len(ancestor_parts) < len(descendant_parts) and (
        descendant_parts[: len(ancestor_parts)] == ancestor_parts
    )


def plan_named_posix_layout(profile: LayoutProfile) -> tuple[PlannedRolePath, ...]:
    if type(profile) is not LayoutProfile:
        raise TypeError("profile must be an exact LayoutProfile record")
    if type(profile.roots) is not tuple:
        raise TypeError("profile roots must be an exact tuple")
    if type(profile.roles) is not tuple:
        raise TypeError("profile roles must be an exact tuple")
    if not 1 <= len(profile.roots) <= _MAX_ROOTS:
        raise LayoutPlanError("profile root count is outside its bound")
    if not 1 <= len(profile.roles) <= _MAX_ROLES:
        raise LayoutPlanError("profile role count is outside its bound")

    roots: dict[str, PurePosixPath] = {}
    root_depths: dict[str, int] = {}
    component_count = 0
    for declaration in profile.roots:
        if type(declaration) is not LogicalRoot:
            raise TypeError("profile roots must contain exact LogicalRoot records")
        name = _checked_name(declaration.name, "logical root name")
        if name in roots:
            raise LayoutPlanError("logical root names must be unique")
        path, depth = _checked_root_path(declaration.path)
        roots[name] = path
        root_depths[name] = depth
        component_count += depth

    role_names: set[str] = set()
    candidates: list[PlannedRolePath] = []
    for declaration in profile.roles:
        if type(declaration) is not RoleDeclaration:
            raise TypeError("profile roles must contain exact RoleDeclaration records")
        name = _checked_name(declaration.name, "role name")
        if name in role_names:
            raise LayoutPlanError("role names must be unique")
        role_names.add(name)

        root_name = _checked_name(declaration.root, "role root reference")
        if root_name not in roots:
            raise LayoutPlanError("a role refers to an undeclared logical root")
        if type(declaration.kind) is not RoleKind:
            raise TypeError("role kinds must be exact RoleKind members")
        suffix = _checked_suffix(declaration.suffix)
        component_count += len(suffix)
        if component_count > _MAX_PROFILE_COMPONENTS:
            raise LayoutPlanError("profile components exceed their aggregate bound")

        if root_depths[root_name] + len(suffix) > _MAX_RENDERED_DEPTH:
            raise LayoutPlanError("a rendered role path exceeds its depth bound")
        path = roots[root_name].joinpath(*suffix)
        if _utf8_size(path.as_posix(), "a rendered role path is not valid UTF-8") > (
            _MAX_PATH_BYTES
        ):
            raise LayoutPlanError("a rendered role path exceeds its byte bound")
        candidates.append(PlannedRolePath(name, declaration.kind, path))

    rendered_paths: set[PurePosixPath] = set()
    for candidate in candidates:
        if candidate.path in rendered_paths:
            raise LayoutPlanError("two roles render to the same path")
        rendered_paths.add(candidate.path)

    for index, left in enumerate(candidates):
        for right in candidates[index + 1 :]:
            if left.kind is RoleKind.FILE and _is_strict_prefix(left.path, right.path):
                raise LayoutPlanError("a file role is a prefix of another role path")
            if right.kind is RoleKind.FILE and _is_strict_prefix(right.path, left.path):
                raise LayoutPlanError("a file role is a prefix of another role path")

    return tuple(candidates)
```

## Example

```python
profile = LayoutProfile(
    roots=(
        LogicalRoot("durable", "/srv/wrenbox"),
        LogicalRoot("transient", "/run/wrenbox"),
    ),
    roles=(
        RoleDeclaration(
            "catalog_file",
            "durable",
            ("records", "catalog.bin"),
            RoleKind.FILE,
        ),
        RoleDeclaration("arrival_dir", "durable", ("arrivals",), RoleKind.DIRECTORY),
        RoleDeclaration(
            "lease_file",
            "transient",
            ("coordination", "writer.lock"),
            RoleKind.FILE,
        ),
    ),
)

planned = plan_named_posix_layout(profile)
assert planned == (
    PlannedRolePath(
        "catalog_file",
        RoleKind.FILE,
        PurePosixPath("/srv/wrenbox/records/catalog.bin"),
    ),
    PlannedRolePath(
        "arrival_dir",
        RoleKind.DIRECTORY,
        PurePosixPath("/srv/wrenbox/arrivals"),
    ),
    PlannedRolePath(
        "lease_file",
        RoleKind.FILE,
        PurePosixPath("/run/wrenbox/coordination/writer.lock"),
    ),
)

colliding = LayoutProfile(
    roots=(
        LogicalRoot("outer", "/srv/starling"),
        LogicalRoot("inner", "/srv/starling/cache"),
    ),
    roles=(
        RoleDeclaration("outer_stamp", "outer", ("cache", "stamp.bin"), RoleKind.FILE),
        RoleDeclaration("inner_stamp", "inner", ("stamp.bin",), RoleKind.FILE),
    ),
)
try:
    plan_named_posix_layout(colliding)
except LayoutPlanError:
    collision_rejected = True
else:
    collision_rejected = False

assert collision_rejected
assert tuple(item.role for item in planned) == (
    "catalog_file",
    "arrival_dir",
    "lease_file",
)
```

## Trade-offs and Limitations

Validation costs `O(b + r^2 d)` time for encoded input size `b`, role count
`r`, and bounded path depth `d`; the pairwise global ambiguity check favors
clarity for at most 24 roles. Working memory is `O(b + r)`. Root and role
counts, component counts, root and suffix depth, individual component bytes,
and rendered UTF-8 path bytes all have explicit limits.

`PurePosixPath` supplies platform-independent lexical joins and performs no I/O.
The result does not prove existence or filesystem containment, resolve
symlinks, inspect permissions or mounts, authorize access, or handle race
conditions. It also does not install, deploy, mutate, or roll back anything;
the caller owns those policies and any later resource lifecycle.

## Related Snippets

<!-- catalog:related:start -->
- [Map a Namespaced POSIX Path Beneath a Logical Local Root](map-a-namespaced-posix-path-beneath-a-logical-local-root.md)
- [Build a Bounded File Manifest with Internal Symlink Aliases](build-a-bounded-file-manifest-with-internal-symlink-aliases.md)
- [Validate a Conservative Unicode Filename Component](../security-privacy/validate-a-conservative-unicode-filename-component.md)
<!-- catalog:related:end -->

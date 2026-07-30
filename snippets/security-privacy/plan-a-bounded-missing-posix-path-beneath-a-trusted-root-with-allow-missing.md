---
title: "Plan a Bounded Missing POSIX Path Beneath a Trusted Root with Allow Missing"
snippet_type: recipe
use_cases:
  - resource-management
  - security
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../storage-databases/map-a-namespaced-posix-path-beneath-a-logical-local-root.md
  - read-one-descriptor-relative-regular-file-without-following-its-final-symlink.md
  - audit-symlinks-in-a-bounded-directory-tree.md
---

# Plan a Bounded Missing POSIX Path Beneath a Trusted Root with Allow Missing

## Idea and Problem

Resolve existing symbolic links in a future POSIX path while allowing a missing suffix, then reject a result outside one trusted canonical root.

`os.path.ALLOW_MISSING` makes `realpath()` report errors other than missing
components while still producing a symlink-free canonical plan. Validating a
small conservative component tuple before resolution removes separator and
parent-traversal ambiguity. The immutable result remains advisory data rather
than authority to create or open anything.

## When to Use

Use this planner on POSIX with Python 3.14 when a trusted process owns an
existing absolute directory root and needs to describe a not-yet-created leaf
below it. Supply from 1 through 16 exact ASCII components, each at most 255
bytes, with a combined relative-path limit of 1,024 bytes. Keep the root and
each resolved absolute result within 4,096 filesystem-encoded bytes. Keep the
root and its existing path topology under the same trusted administrative
boundary.

Do not use the plan to authorize a later operation in a mutable or hostile
directory tree. Perform the real creation or open through an operating-system
primitive and policy that protects the entire check/use interval.

## Implementation

```python
import os
import re
from dataclasses import dataclass
from pathlib import Path, PurePosixPath
from tempfile import TemporaryDirectory

_MAX_COMPONENTS = 16
_MAX_COMPONENT_BYTES = 255
_MAX_RELATIVE_PATH_BYTES = 1_024
_MAX_RESOLVED_PATH_BYTES = 4_096
_COMPONENT_PATTERN = re.compile(r"[A-Za-z0-9][A-Za-z0-9._-]*", flags=re.ASCII)


class MissingPathPlanError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class MissingPathPlan:
    canonical_root: Path
    planned_path: Path
    canonical_relative_path: PurePosixPath


def _validate_missing_path_components(
    components: tuple[object, ...],
) -> tuple[str, ...]:
    if not 1 <= len(components) <= _MAX_COMPONENTS:
        raise MissingPathPlanError("component count is outside the supported range")

    accepted: list[str] = []
    for component in components:
        if type(component) is not str:
            raise TypeError("each component must be exact text")
        try:
            encoded = component.encode("ascii")
        except UnicodeEncodeError as error:
            raise MissingPathPlanError("components must contain only ASCII") from error
        if not 1 <= len(encoded) <= _MAX_COMPONENT_BYTES:
            raise MissingPathPlanError("component byte length is outside the supported range")
        if _COMPONENT_PATTERN.fullmatch(component) is None:
            raise MissingPathPlanError("component is outside the conservative name grammar")
        accepted.append(component)

    relative_text = "/".join(accepted)
    if len(relative_text.encode("ascii")) > _MAX_RELATIVE_PATH_BYTES:
        raise MissingPathPlanError("relative path exceeds the byte limit")
    return tuple(accepted)


def plan_missing_posix_path(
    root: str | os.PathLike[str],
    *components: str,
) -> MissingPathPlan:
    if os.name != "posix" or not hasattr(os.path, "ALLOW_MISSING"):
        raise NotImplementedError("ALLOW_MISSING path planning requires POSIX Python 3.14")

    root_path = Path(root)
    if not root_path.is_absolute():
        raise MissingPathPlanError("root must be an absolute path")
    if len(os.fsencode(root_path)) > _MAX_RESOLVED_PATH_BYTES:
        raise MissingPathPlanError("root exceeds the filesystem byte limit")
    canonical_root = Path(os.path.realpath(root_path, strict=True))
    if len(os.fsencode(canonical_root)) > _MAX_RESOLVED_PATH_BYTES:
        raise MissingPathPlanError("canonical root exceeds the filesystem byte limit")
    if not canonical_root.is_dir():
        raise NotADirectoryError("root must resolve to an existing directory")

    accepted = _validate_missing_path_components(components)
    planned_path = Path(
        os.path.realpath(
            canonical_root.joinpath(*accepted),
            strict=os.path.ALLOW_MISSING,
        ),
    )
    if len(os.fsencode(planned_path)) > _MAX_RESOLVED_PATH_BYTES:
        raise MissingPathPlanError("planned path exceeds the filesystem byte limit")
    common_root = Path(os.path.commonpath((canonical_root, planned_path)))
    if common_root != canonical_root or planned_path == canonical_root:
        raise MissingPathPlanError("resolved path is not strictly beneath the trusted root")
    canonical_relative_path = PurePosixPath(
        planned_path.relative_to(canonical_root).as_posix()
    )

    return MissingPathPlan(
        canonical_root=canonical_root,
        planned_path=planned_path,
        canonical_relative_path=canonical_relative_path,
    )
```

## Example

```python
with TemporaryDirectory() as temporary_directory:
    workspace = Path(temporary_directory)
    root = workspace / "trusted"
    objects = root / "objects"
    outside = workspace / "outside"
    objects.mkdir(parents=True)
    outside.mkdir()
    (root / "current").symlink_to("objects", target_is_directory=True)
    (root / "escape").symlink_to(outside, target_is_directory=True)

    plan = plan_missing_posix_path(root, "current", "incoming", "item.bin")
    canonical_root = Path(os.path.realpath(root, strict=True))

    try:
        plan_missing_posix_path(root, "escape", "item.bin")
    except MissingPathPlanError:
        escaping_symlink_rejected = True
    else:
        escaping_symlink_rejected = False

    (objects / "incoming").symlink_to(outside, target_is_directory=True)
    resolved_after_mutation = Path(
        os.path.realpath(
            root / "current" / "incoming" / "item.bin",
            strict=os.path.ALLOW_MISSING,
        ),
    )

assert (
    plan,
    escaping_symlink_rejected,
    resolved_after_mutation != plan.planned_path,
) == (
    MissingPathPlan(
        canonical_root=canonical_root,
        planned_path=canonical_root / "objects" / "incoming" / "item.bin",
        canonical_relative_path=PurePosixPath("objects/incoming/item.bin"),
    ),
    True,
    True,
)
```

## Trade-offs and Limitations

The component limits bound validation before resolution, and the
filesystem-byte checks bound each returned absolute path after resolution.
`realpath()` can still block or transiently construct a larger expansion before
the post-resolution check rejects it, so the existing topology must remain
trusted. `ALLOW_MISSING` resolves existing links and re-raises errors other than
missing components; it does not prove that any component is absent or available
for creation. Platform path limits can be lower than this policy's output cap;
those filesystem errors propagate. The strict ASCII grammar deliberately
rejects many valid POSIX names, including whitespace and names that begin with
punctuation, in exchange for a small transport-friendly policy.

This is a path plan, not an authorization result or filesystem snapshot. A
missing component can become a symbolic link after validation, as the example
demonstrates, and existing components can be renamed or retargeted. The helper
does not create or open a file, confine mount traversal, inspect permissions,
detect hard links, or close any TOCTOU window. Use descriptor-relative or
platform-specific no-follow operations when later access must resist path
replacement.

## Related Snippets

<!-- catalog:related:start -->
- [Map a Namespaced POSIX Path Beneath a Logical Local Root](../storage-databases/map-a-namespaced-posix-path-beneath-a-logical-local-root.md)
- [Read One Descriptor-Relative Regular File Without Following Its Final Symlink](read-one-descriptor-relative-regular-file-without-following-its-final-symlink.md)
- [Audit Symlinks in a Bounded Directory Tree](audit-symlinks-in-a-bounded-directory-tree.md)
<!-- catalog:related:end -->

---
title: "Map a Namespaced POSIX Path Beneath a Logical Local Root"
snippet_type: recipe
use_cases:
  - data-transformation
  - persistence
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-a-bounded-file-manifest-with-internal-symlink-aliases.md
  - ../security-privacy/plan-a-bounded-notebook-storage-key-with-collision-suggestions.md
  - ../security-privacy/redact-explicit-paths-in-bounded-json-like-data.md
---

# Map a Namespaced POSIX Path Beneath a Logical Local Root

## Idea and Problem

Translate one canonical absolute POSIX namespace path into a pure local-root plan while preserving its relative suffix.

A storage adapter may receive names such as `/archive/objects/2026/item.bin`
and need the lexical counterpart below another logical root. Comparing parsed
components instead of string prefixes rejects sibling lookalikes, while strict
canonical-input rules make traversal and separator ambiguity explicit. The
result remains a plan, so it cannot be mistaken for a filesystem authorization
decision.

## When to Use

Use this function when both namespaces have POSIX semantics even if planning
runs on another operating system. All three paths must already be canonical,
absolute strings, and the source path must name an entry strictly below its
declared namespace root. The consumer may later interpret the returned
`PurePosixPath` under a separately defined trusted filesystem policy.

Do not use lexical translation to authorize hostile filesystem access. Real
containment may require descriptor-relative operating-system operations and a
policy for symlinks, mounts, races, permissions, and case sensitivity.

## Implementation

```python
from dataclasses import dataclass
from pathlib import PurePosixPath


_MAX_PATH_BYTES = 1_024
_MAX_COMPONENTS = 64


@dataclass(frozen=True, slots=True)
class PosixPathMapping:
    relative_path: PurePosixPath
    mapped_path: PurePosixPath


def _absolute_parts(value: object, field: str) -> tuple[str, ...]:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not value or len(value.encode("utf-8")) > _MAX_PATH_BYTES:
        raise ValueError(f"{field} is empty or too large")
    if not value.startswith("/") or value.startswith("//"):
        raise ValueError(f"{field} must be an absolute POSIX path")
    if "\x00" in value or "\\" in value:
        raise ValueError(f"{field} contains a forbidden character")
    if value != "/" and value.endswith("/"):
        raise ValueError(f"{field} must not end with a slash")
    if "//" in value[1:]:
        raise ValueError(f"{field} contains a repeated separator")

    parts = () if value == "/" else tuple(value[1:].split("/"))
    if len(parts) > _MAX_COMPONENTS:
        raise ValueError(f"{field} component limit exceeded")
    if any(part in ("", ".", "..") for part in parts):
        raise ValueError(f"{field} contains a forbidden component")
    return parts


def map_namespaced_posix_path(
    namespace_root: str,
    namespaced_path: str,
    local_root: str,
) -> PosixPathMapping:
    source_root = _absolute_parts(namespace_root, "namespace_root")
    source = _absolute_parts(namespaced_path, "namespaced_path")
    destination = _absolute_parts(local_root, "local_root")

    if len(source) <= len(source_root) or source[: len(source_root)] != source_root:
        raise ValueError("namespaced_path must be strictly below namespace_root")
    suffix = source[len(source_root) :]
    output = (*destination, *suffix)
    if len(output) > _MAX_COMPONENTS:
        raise ValueError("mapped path component limit exceeded")

    relative_path = PurePosixPath(*suffix)
    mapped_path = PurePosixPath("/", *output)
    if len(mapped_path.as_posix().encode("utf-8")) > _MAX_PATH_BYTES:
        raise ValueError("mapped path byte limit exceeded")
    return PosixPathMapping(relative_path, mapped_path)
```

## Example

```python
mapping = map_namespaced_posix_path(
    "/archive/objects",
    "/archive/objects/2026/item.bin",
    "/mirror/cache",
)

assert mapping == PosixPathMapping(
    relative_path=PurePosixPath("2026/item.bin"),
    mapped_path=PurePosixPath("/mirror/cache/2026/item.bin"),
)
```

## Trade-offs and Limitations

The function performs `O(p)` validation and allocation over at most 64 path
components and 1,024 UTF-8 bytes per input and output. Its strict textual
grammar rejects repeated or trailing separators, dot components, backslashes,
and the POSIX implementation-defined double-leading-slash form even when some
systems would accept them. Percent-encoded-looking components remain ordinary
literal names because this is not a URL decoder.

`PurePosixPath` performs no I/O. The returned path is neither proof of
existence nor authorization, and it does not resolve symlinks, detect mounts,
open files, compare filesystem identities, or prevent check/use races. A caller
must apply its security and lifecycle policy at the actual filesystem boundary.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Bounded File Manifest with Internal Symlink Aliases](build-a-bounded-file-manifest-with-internal-symlink-aliases.md)
- [Plan a Bounded Notebook Storage Key with Collision Suggestions](../security-privacy/plan-a-bounded-notebook-storage-key-with-collision-suggestions.md)
- [Redact Explicit Paths in Bounded JSON-Like Data](../security-privacy/redact-explicit-paths-in-bounded-json-like-data.md)
<!-- catalog:related:end -->

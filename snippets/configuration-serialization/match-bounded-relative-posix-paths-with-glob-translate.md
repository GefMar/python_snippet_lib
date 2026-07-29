---
title: "Match Bounded Relative POSIX Paths with glob.translate"
snippet_type: pattern
use_cases:
  - configuration
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../security-privacy/audit-bounded-relative-posix-archive-member-names-under-a-closed-policy.md
  - ../storage-databases/map-a-namespaced-posix-path-beneath-a-logical-local-root.md
  - resolve-declared-input-paths-from-an-explicit-execution-mode.md
---

# Match Bounded Relative POSIX Paths with glob.translate

## Idea and Problem

Classify bounded canonical relative POSIX paths by the first matching segment-aware glob pattern without accessing a filesystem.

`glob.translate` converts shell-style wildcards into a regular expression while
preserving path-separator semantics. Fixing `/` as the separator makes matching
independent of the host platform, and first-match order turns a small declared
pattern tuple into a deterministic classification policy.

## When to Use

Use this pattern for configuration selectors, manifest classification, or
offline tests when both patterns and candidate paths are already relative
POSIX spellings. `**` may cross segments, while an ordinary `*` remains within
one segment. Hidden segments are excluded unless a pattern explicitly begins
the corresponding segment with a dot.

This is lexical classification only. It does not expand patterns, inspect
files, resolve symbolic links, prove containment, or authorize access. Use a
filesystem-aware boundary when the existence, type, ownership, or resolved
location of a path determines correctness.

## Implementation

```python
import glob
import re
from dataclasses import dataclass

_MAX_PATTERNS = 32
_MAX_PATTERN_BYTES = 256
_MAX_PATTERN_BYTES_TOTAL = 8 * 1_024
_MAX_PATHS = 1_024
_MAX_PATH_BYTES = 1_024
_MAX_PATH_BYTES_TOTAL = 256 * 1_024


@dataclass(frozen=True, slots=True)
class PathMatch:
    path: str
    pattern_index: int | None


def _validate_relative_posix_text(
    value: str,
    *,
    name: str,
    maximum_bytes: int,
) -> int:
    if type(value) is not str:
        raise TypeError(f"{name} must be an exact string")
    if len(value) > maximum_bytes:
        raise ValueError(f"{name} exceeds the supported byte count")
    if not value or value.startswith("/") or value.endswith("/"):
        raise ValueError(f"{name} must be a nonempty relative POSIX spelling")
    if "\\" in value:
        raise ValueError(f"{name} must use POSIX separators")
    if any(ord(character) < 32 or ord(character) == 127 for character in value):
        raise ValueError(f"{name} must not contain control characters")
    segments = value.split("/")
    if any(segment in {"", ".", ".."} for segment in segments):
        raise ValueError(f"{name} contains a noncanonical path segment")
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError as error:
        raise ValueError(f"{name} must not contain Unicode surrogates") from error
    if len(encoded) > maximum_bytes:
        raise ValueError(f"{name} exceeds the supported UTF-8 byte count")
    return len(encoded)


def match_relative_posix_paths(
    patterns: tuple[str, ...],
    paths: tuple[str, ...],
) -> tuple[PathMatch, ...]:
    if type(patterns) is not tuple:
        raise TypeError("patterns must be an exact tuple")
    if not 1 <= len(patterns) <= _MAX_PATTERNS:
        raise ValueError("pattern count is outside the supported range")
    if type(paths) is not tuple:
        raise TypeError("paths must be an exact tuple")
    if len(paths) > _MAX_PATHS:
        raise ValueError("paths exceed the supported count")

    pattern_bytes = 0
    seen_patterns: set[str] = set()
    compiled_patterns: list[re.Pattern[str]] = []
    for pattern in patterns:
        pattern_bytes += _validate_relative_posix_text(
            pattern,
            name="pattern",
            maximum_bytes=_MAX_PATTERN_BYTES,
        )
        if pattern_bytes > _MAX_PATTERN_BYTES_TOTAL:
            raise ValueError("patterns exceed the aggregate byte count")
        if pattern in seen_patterns:
            raise ValueError("patterns must be unique")
        seen_patterns.add(pattern)
        translated = glob.translate(
            pattern,
            recursive=True,
            include_hidden=False,
            seps="/",
        )
        compiled_patterns.append(re.compile(translated))

    path_bytes = 0
    decisions: list[PathMatch] = []
    for path in paths:
        path_bytes += _validate_relative_posix_text(
            path,
            name="path",
            maximum_bytes=_MAX_PATH_BYTES,
        )
        if path_bytes > _MAX_PATH_BYTES_TOTAL:
            raise ValueError("paths exceed the aggregate byte count")

        matching_index: int | None = None
        for index, compiled in enumerate(compiled_patterns):
            if compiled.match(path) is not None:
                matching_index = index
                break
        decisions.append(PathMatch(path=path, pattern_index=matching_index))
    return tuple(decisions)
```

## Example

```python
patterns = (
    "src/**",
    "src/**/*.py",
    "docs/*.md",
    ".config/*.toml",
)
paths = (
    "src/main.py",
    "src/package/module.py",
    "docs/index.md",
    "docs/guides/index.md",
    ".config/tool.toml",
    "src/.cache/state.bin",
    "assets/logo.svg",
    "src/main.py",
)

matches = match_relative_posix_paths(patterns, paths)

assert tuple(match.pattern_index for match in matches) == (
    0,
    0,
    2,
    None,
    3,
    None,
    None,
    0,
)
assert tuple(match.path for match in matches) == paths

specific = match_relative_posix_paths(
    ("src/**/*.py",),
    ("src/main.py", "src/package/module.py", "src/readme.md"),
)
assert tuple(match.pattern_index for match in specific) == (0, 0, None)

literal_bracket = match_relative_posix_paths(("[abc",), ("[abc",))
assert literal_bracket[0].pattern_index == 0

pattern_boundary = tuple(f"group-{index}/**" for index in range(32))
assert len(match_relative_posix_paths(pattern_boundary, ())) == 0
try:
    match_relative_posix_paths((*pattern_boundary, "overflow/**"), ())
except ValueError:
    pass
else:
    raise AssertionError("33 patterns must be rejected")

path_boundary = ("item",) * 1_024
assert len(match_relative_posix_paths(("item",), path_boundary)) == 1_024
try:
    match_relative_posix_paths(("item",), (*path_boundary, "item"))
except ValueError:
    pass
else:
    raise AssertionError("1,025 paths must be rejected")

assert match_relative_posix_paths(("x" * 256,), ("x" * 1_024,))[0] == PathMatch(
    "x" * 1_024,
    None,
)
```

## Trade-offs and Limitations

Validation and translation process the bounded pattern and path bytes once.
Classification performs at most `len(patterns) * len(paths)` bounded regular
expression attempts and stops at the first match per path. Python's regex
engine cost is implementation-dependent, so the helper does not promise
portable linear matching time.

Pattern order is policy: placing a broad pattern first prevents later specific
patterns from being reported. Paths may repeat and keep separate result
records, while patterns must be unique so an index identifies one declaration.
`include_hidden=False` follows glob's segment rules; an explicitly dot-prefixed
pattern can still select a hidden segment.

Unmatched bracket metacharacters can be interpreted literally by glob syntax
and are not rejected as malformed. The profile rejects noncanonical path
spellings, but it does not normalize Unicode, case, percent escapes, or
platform-specific paths. No result proves that a path exists, stays beneath a
directory after symlink resolution, or is safe for an operation.

## Related Snippets

<!-- catalog:related:start -->
- [Audit Bounded Relative POSIX Archive Member Names Under a Closed Policy](../security-privacy/audit-bounded-relative-posix-archive-member-names-under-a-closed-policy.md)
- [Map a Namespaced POSIX Path Beneath a Logical Local Root](../storage-databases/map-a-namespaced-posix-path-beneath-a-logical-local-root.md)
- [Resolve Declared Input Paths from an Explicit Execution Mode](resolve-declared-input-paths-from-an-explicit-execution-mode.md)
<!-- catalog:related:end -->

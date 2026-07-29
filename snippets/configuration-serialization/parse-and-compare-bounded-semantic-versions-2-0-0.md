---
title: "Parse and Compare Bounded Semantic Versions 2.0.0"
snippet_type: recipe
use_cases:
  - configuration
  - interoperability
  - parsing
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/sort-dotted-release-labels-with-an-explicit-last-marker.md
  - migrate-one-bounded-json-record-to-a-current-version.md
  - match-a-client-against-a-bounded-platform-availability-rule.md
---

# Parse and Compare Bounded Semantic Versions 2.0.0

## Idea and Problem

Parse a bounded Semantic Versioning 2.0.0 value without losing its spelling, then compare two values by SemVer precedence.

[Semantic Versioning 2.0.0](https://semver.org/) core and numeric pre-release
identifiers can exceed machine-integer ranges.
Canonical digit strings are therefore compared by length and then
lexicographically, without arbitrary-precision integer conversion.

Build metadata survives parsing and formatting but is ignored for precedence.
Consequently, immutable structural equality remains distinct from a comparator
result of zero.

## When to Use

Use this parser at a closed configuration or interoperability boundary that
requires exact SemVer 2.0.0 grammar, round-trip rendering, and precedence
comparison under explicit resource limits.

Keep the original parsed value when build metadata matters for display,
provenance, or artifact identity. Use the separate comparator only where
SemVer precedence, rather than textual identity, is the intended relation.

## Implementation

```python
from dataclasses import dataclass
from itertools import pairwise

_MAX_TEXT_LENGTH = 256
_MAX_IDENTIFIERS = 16
_MAX_IDENTIFIER_LENGTH = 32
_ASCII_DIGITS = frozenset("0123456789")
_IDENTIFIER_CHARACTERS = frozenset(
    "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz-"
)


@dataclass(frozen=True, slots=True)
class SemanticVersion:
    major: str
    minor: str
    patch: str
    prerelease: tuple[str, ...]
    build: tuple[str, ...]


def _is_ascii_numeric(value: str) -> bool:
    return bool(value) and all(character in _ASCII_DIGITS for character in value)


def _validate_core(name: str, value: object) -> str:
    if type(value) is not str:
        raise TypeError(f"{name} must be an exact string")
    if not 1 <= len(value) <= _MAX_TEXT_LENGTH:
        raise ValueError(f"{name} size is outside the supported range")
    if not _is_ascii_numeric(value):
        raise ValueError(f"{name} must be an ASCII numeric identifier")
    if len(value) > 1 and value[0] == "0":
        raise ValueError(f"{name} must not contain a leading zero")
    return value


def _validate_identifiers(
    name: str,
    identifiers: object,
    *,
    numeric_leading_zero_forbidden: bool,
) -> tuple[str, ...]:
    if type(identifiers) is not tuple:
        raise TypeError(f"{name} must be an exact tuple")
    if len(identifiers) > _MAX_IDENTIFIERS:
        raise ValueError(f"{name} contains too many identifiers")

    for identifier in identifiers:
        if type(identifier) is not str:
            raise TypeError(f"{name} identifiers must be exact strings")
        if not 1 <= len(identifier) <= _MAX_IDENTIFIER_LENGTH:
            raise ValueError(f"{name} identifier size is outside the supported range")
        if any(
            character not in _IDENTIFIER_CHARACTERS
            for character in identifier
        ):
            raise ValueError(f"{name} identifier contains a forbidden character")
        if (
            numeric_leading_zero_forbidden
            and _is_ascii_numeric(identifier)
            and len(identifier) > 1
            and identifier[0] == "0"
        ):
            raise ValueError(
                f"numeric {name} identifier must not contain a leading zero"
            )
    return identifiers


def _parse_identifier_text(
    name: str,
    value: str,
    *,
    numeric_leading_zero_forbidden: bool,
) -> tuple[str, ...]:
    return _validate_identifiers(
        name,
        tuple(value.split(".")),
        numeric_leading_zero_forbidden=numeric_leading_zero_forbidden,
    )


def parse_semantic_version(value: str) -> SemanticVersion:
    """Parse one exact value under a bounded SemVer 2.0.0 profile."""
    if type(value) is not str:
        raise TypeError("value must be an exact string")
    if not 1 <= len(value) <= _MAX_TEXT_LENGTH:
        raise ValueError("value length is outside the supported range")
    if not value.isascii():
        raise ValueError("value must contain only ASCII characters")
    if value.count("+") > 1:
        raise ValueError("value contains more than one build separator")

    precedence_text, build_separator, build_text = value.partition("+")
    build = (
        _parse_identifier_text(
            "build",
            build_text,
            numeric_leading_zero_forbidden=False,
        )
        if build_separator
        else ()
    )

    core_text, prerelease_separator, prerelease_text = precedence_text.partition("-")
    prerelease = (
        _parse_identifier_text(
            "prerelease",
            prerelease_text,
            numeric_leading_zero_forbidden=True,
        )
        if prerelease_separator
        else ()
    )

    core = core_text.split(".")
    if len(core) != 3:
        raise ValueError("core version must contain major, minor, and patch")
    major = _validate_core("major", core[0])
    minor = _validate_core("minor", core[1])
    patch = _validate_core("patch", core[2])
    return SemanticVersion(major, minor, patch, prerelease, build)


def format_semantic_version(value: SemanticVersion) -> str:
    """Render one completely revalidated parsed value."""
    if type(value) is not SemanticVersion:
        raise TypeError("value must be an exact SemanticVersion")
    major = _validate_core("major", value.major)
    minor = _validate_core("minor", value.minor)
    patch = _validate_core("patch", value.patch)
    prerelease = _validate_identifiers(
        "prerelease",
        value.prerelease,
        numeric_leading_zero_forbidden=True,
    )
    build = _validate_identifiers(
        "build",
        value.build,
        numeric_leading_zero_forbidden=False,
    )

    rendered = f"{major}.{minor}.{patch}"
    if prerelease:
        rendered += "-" + ".".join(prerelease)
    if build:
        rendered += "+" + ".".join(build)
    if len(rendered) > _MAX_TEXT_LENGTH:
        raise ValueError("rendered value exceeds the supported length")
    return rendered


def _compare_numeric(left: str, right: str) -> int:
    if len(left) != len(right):
        return -1 if len(left) < len(right) else 1
    return (left > right) - (left < right)


def _compare_prerelease(
    left: tuple[str, ...],
    right: tuple[str, ...],
) -> int:
    if not left or not right:
        if left == right:
            return 0
        return -1 if left else 1

    for left_identifier, right_identifier in zip(left, right, strict=False):
        if left_identifier == right_identifier:
            continue
        left_is_numeric = _is_ascii_numeric(left_identifier)
        right_is_numeric = _is_ascii_numeric(right_identifier)
        if left_is_numeric and right_is_numeric:
            return _compare_numeric(left_identifier, right_identifier)
        if left_is_numeric != right_is_numeric:
            return -1 if left_is_numeric else 1
        return (left_identifier > right_identifier) - (
            left_identifier < right_identifier
        )
    return (len(left) > len(right)) - (len(left) < len(right))


def compare_semantic_version_precedence(left: str, right: str) -> int:
    """Return -1, 0, or 1 under SemVer 2.0.0 precedence."""
    left_version = parse_semantic_version(left)
    right_version = parse_semantic_version(right)

    for left_core, right_core in zip(
        (left_version.major, left_version.minor, left_version.patch),
        (right_version.major, right_version.minor, right_version.patch),
        strict=True,
    ):
        comparison = _compare_numeric(left_core, right_core)
        if comparison:
            return comparison
    return _compare_prerelease(
        left_version.prerelease,
        right_version.prerelease,
    )
```

## Example

```python
precedence_chain = (
    "1.0.0-alpha",
    "1.0.0-alpha.1",
    "1.0.0-alpha.beta",
    "1.0.0-beta",
    "1.0.0-beta.2",
    "1.0.0-beta.11",
    "1.0.0-rc.1",
    "1.0.0",
)
assert all(
    compare_semantic_version_precedence(left, right) == -1
    for left, right in pairwise(precedence_chain)
)

labelled = parse_semantic_version("2.7.1-rc.3+linux.001")
assert format_semantic_version(labelled) == "2.7.1-rc.3+linux.001"

left_build = parse_semantic_version("1.0.0+build.1")
right_build = parse_semantic_version("1.0.0+build.2")
assert left_build != right_build
assert compare_semantic_version_precedence(
    format_semantic_version(left_build),
    format_semantic_version(right_build),
) == 0

huge_core = "9999999999999999999999999999999999999999.0.0"
assert compare_semantic_version_precedence(huge_core, "10.0.0") == 1
```

## Trade-offs and Limitations

Parsing, formatting, and comparison use `O(L)` time and state for bounded
input length `L`. This profile further caps each pre-release or build list at
16 identifiers and each such identifier at 32 characters. Those resource
limits intentionally reject some otherwise valid SemVer text.

Precedence equality is not artifact identity: build metadata does not affect
ordering, while the dataclass preserves it for structural equality and
round-trip rendering. SemVer precedence also does not prove API compatibility
or decide whether an upgrade is safe.

No `v` prefix, whitespace coercion, PEP 440 syntax, epoch, wildcard, range,
caret or tilde rule, dependency resolution, automated version bumping, or
normalization that discards build metadata is supported.

## Related Snippets

<!-- catalog:related:start -->
- [Sort Dotted Release Labels with an Explicit Last Marker](../algorithms-data-structures/sort-dotted-release-labels-with-an-explicit-last-marker.md)
- [Migrate One Bounded JSON Record to a Current Version](migrate-one-bounded-json-record-to-a-current-version.md)
- [Match a Client Against a Bounded Platform Availability Rule](match-a-client-against-a-bounded-platform-availability-rule.md)
<!-- catalog:related:end -->

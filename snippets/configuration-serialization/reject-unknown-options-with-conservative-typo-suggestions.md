---
title: "Reject Unknown Options with Conservative Typo Suggestions"
snippet_type: recipe
use_cases:
  - configuration
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - merge-nested-configuration-with-an-explicit-delete-sentinel.md
  - ../python-language/build-a-read-only-mapping-with-canonical-text-keys.md
  - ../reliability-resilience/resolve-incoming-configuration-with-last-known-good-values.md
---

# Reject Unknown Options with Conservative Typo Suggestions

## Idea and Problem

Reject every unknown option name while attaching a small deterministic typo hint when one allowed name is sufficiently similar.

The validator treats suggestions as diagnostics, never corrections. Allowed
names and input keys use one bounded ASCII identifier grammar, candidates are
sorted before matching, and error details contain names only so option values
cannot leak into logs or exception text.

## When to Use

Use this helper at a small configuration or keyword boundary where silently
ignoring a misspelled name would be dangerous. Keep the allowlist explicit and
reviewed, and validate option values separately after the names pass. Use a
schema library when nested structures, aliases, defaults, coercion, rich source
locations, or many hundreds of fields are required.

## Implementation

```python
import re
from collections.abc import Iterable, Mapping
from difflib import get_close_matches
from typing import TypeVar


ValueT = TypeVar("ValueT")
_MAX_ALLOWED_NAMES = 64
_MAX_OPTIONS = 64
_OPTION_NAME = re.compile(r"[a-z][a-z0-9_]{0,31}", re.ASCII)
_SUGGESTION_CUTOFF = 0.82


class UnknownOptionsError(ValueError):
    def __init__(
        self,
        details: tuple[tuple[str, str | None], ...],
    ) -> None:
        self.details = details
        rendered = []
        for name, suggestion in details:
            hint = (
                f" (did you mean {suggestion}?)"
                if suggestion is not None
                else ""
            )
            rendered.append(f"{name}{hint}")
        super().__init__("unknown options: " + ", ".join(rendered))


def _validate_option_name(name: object, *, source: str) -> str:
    if not isinstance(name, str):
        raise TypeError(f"{source} names must be text")
    if _OPTION_NAME.fullmatch(name) is None:
        raise ValueError(f"{source} contains an invalid option name")
    return name


def validate_named_options(
    options: Mapping[str, ValueT],
    allowed_names: Iterable[str],
) -> dict[str, ValueT]:
    if not isinstance(options, Mapping):
        raise TypeError("options must be a mapping")
    if len(options) > _MAX_OPTIONS:
        raise ValueError("options exceed the supported count")
    if isinstance(allowed_names, (str, bytes)):
        raise TypeError("allowed_names must be an iterable of names")

    allowed: list[str] = []
    seen: set[str] = set()
    for allowed_count, raw_name in enumerate(allowed_names, start=1):
        if allowed_count > _MAX_ALLOWED_NAMES:
            raise ValueError("allowed_names exceed the supported count")
        name = _validate_option_name(raw_name, source="allowed_names")
        if name in seen:
            raise ValueError("allowed_names must be unique")
        seen.add(name)
        allowed.append(name)
    if not allowed:
        raise ValueError("allowed_names must not be empty")

    unknown: list[str] = []
    for raw_name in options:
        name = _validate_option_name(raw_name, source="options")
        if name not in seen:
            unknown.append(name)

    if unknown:
        ordered_candidates = sorted(allowed)
        details = []
        for name in sorted(unknown):
            matches = get_close_matches(
                name,
                ordered_candidates,
                n=1,
                cutoff=_SUGGESTION_CUTOFF,
            )
            details.append((name, matches[0] if matches else None))
        raise UnknownOptionsError(tuple(details))

    return dict(options)
```

## Example

```python
allowed = ("required", "description", "default_value")
validated = validate_named_options(
    {"description": "Retry count", "required": True},
    allowed,
)

try:
    validate_named_options(
        {
            "unrelated": "do-not-expose-this-value",
            "reqired": True,
        },
        reversed(allowed),
    )
except UnknownOptionsError as error:
    details = error.details
    rendered_error = str(error)
else:
    details = ()
    rendered_error = ""

assert (
    validated,
    details,
    "do-not-expose-this-value" not in rendered_error,
) == (
    {"description": "Retry count", "required": True},
    (("reqired", "required"), ("unrelated", None)),
    True,
)
```

## Trade-offs and Limitations

`difflib` similarity is a spelling heuristic, not semantic understanding, and
even a high-cutoff suggestion can be wrong. Matching is case-sensitive and does
not detect Unicode confusables because the accepted grammar is intentionally
ASCII-only. Comparing every unknown name with every allowed name is suitable
for these small bounded sets, not a large registry. The helper makes a shallow
copy, validates names only, and does not apply defaults, aliases, coercion,
nested schemas, value validation, or automatic correction. Adding allowed
names can also change future suggestions, so tests should assert only hints
that matter to the interface.

## Related Snippets

<!-- catalog:related:start -->
- [Merge Nested Configuration with an Explicit Delete Sentinel](merge-nested-configuration-with-an-explicit-delete-sentinel.md)
- [Build a Read-Only Mapping with Canonical Text Keys](../python-language/build-a-read-only-mapping-with-canonical-text-keys.md)
- [Resolve Incoming Configuration with Last-Known-Good Values](../reliability-resilience/resolve-incoming-configuration-with-last-known-good-values.md)
<!-- catalog:related:end -->

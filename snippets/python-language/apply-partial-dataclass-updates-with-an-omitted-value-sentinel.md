---
title: "Apply Partial Dataclass Updates with an Omitted-Value Sentinel"
snippet_type: recipe
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - pass-constructor-only-context-with-initvar.md
  - make-a-defensive-copy-at-a-mutable-input-boundary.md
---

# Apply Partial Dataclass Updates with an Omitted-Value Sentinel

## Idea and Problem

Use a unique sentinel to distinguish an omitted dataclass patch field from an explicit None, false, or zero value.

Optional patch values need three states: omitted, explicitly empty, and
populated. A singleton sentinel represents omission. Reading patch values
directly with `dataclasses.fields` preserves sentinel identity, and
`dataclasses.replace` builds a new target object from only the supplied values.

## When to Use

Use this pattern for a small, typed patch model whose field names intentionally
match an immutable target dataclass. It is useful when `None`, `False`, or zero
are valid replacements rather than missing markers. Prefer explicit conversion
code when patch and target names differ, nested fields need recursive merge
rules, or authorization determines which fields may change.

## Implementation

```python
from dataclasses import dataclass, fields, replace
from typing import Final


class _Omitted:
    __slots__ = ()


OMITTED: Final = _Omitted()


@dataclass(frozen=True, slots=True)
class Details:
    code: str


@dataclass(frozen=True, slots=True)
class Entry:
    label: str | None
    enabled: bool
    limit: int
    details: Details | None


@dataclass(frozen=True, slots=True)
class EntryPatch:
    label: str | None | _Omitted = OMITTED
    enabled: bool | _Omitted = OMITTED
    limit: int | _Omitted = OMITTED
    details: Details | None | _Omitted = OMITTED


def apply_entry_patch(current: Entry, patch: EntryPatch) -> Entry:
    changes = {
        item.name: value
        for item in fields(patch)
        if (value := getattr(patch, item.name)) is not OMITTED
    }
    if not changes:
        return current
    return replace(current, **changes)
```

## Example

```python
current = Entry(
    label="active",
    enabled=True,
    limit=5,
    details=Details("original"),
)

unchanged = apply_entry_patch(current, EntryPatch())
updated = apply_entry_patch(
    current,
    EntryPatch(
        label=None,
        enabled=False,
        limit=0,
        details=Details("replacement"),
    ),
)

assert (
    unchanged is current,
    updated.label,
    updated.enabled,
    updated.limit,
    isinstance(updated.details, Details),
    updated.details,
) == (True, None, False, 0, True, Details("replacement"))
```

## Trade-offs and Limitations

The patch and target schemas are coupled and must evolve together. Do not use
`dataclasses.asdict` for this filtering step: it recursively converts nested
dataclasses and deep-copies other values, which can destroy singleton identity.
`replace` calls the target constructor and `__post_init__` again; init-only
variables without defaults need special handling, and `init=False` fields
cannot be supplied as changes. Recursive patching and per-field permissions
should remain explicit policies rather than hidden behavior in this helper.

## Related Snippets

<!-- catalog:related:start -->
- [Pass Constructor-Only Context with dataclasses.InitVar](pass-constructor-only-context-with-initvar.md)
- [Make a Defensive Copy at a Mutable Input Boundary](make-a-defensive-copy-at-a-mutable-input-boundary.md)
<!-- catalog:related:end -->

---
title: "Report Unicode NFC Collisions Before Importing Keys"
snippet_type: recipe
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/build-a-canonical-unicode-caseless-comparison-key.md
  - project-a-bounded-document-snapshot-into-collision-audited-secondary-index-rows.md
---

# Report Unicode NFC Collisions Before Importing Keys

## Idea and Problem

Report distinct original keys that would become the same key after Unicode NFC normalization.

A composed character and its decomposed spelling can produce the same NFC key.
Building a dictionary under normalized keys can silently overwrite one of the
rows. Group the originals by normalized key first and report every group with
multiple distinct spellings. Sort the groups and their members for a stable
report regardless of input order.

## When to Use

Use this before importing a finite batch into a destination that uses NFC
keys and compares them by exact string equality. Run it on the original
spellings before normalization or insertion. Use the evidence to reject the
batch or ask the caller to resolve the conflicting rows.

## Implementation

```python
from collections import defaultdict
from collections.abc import Iterable
from dataclasses import dataclass
from unicodedata import normalize


@dataclass(frozen=True, slots=True)
class NfcCollision:
    normalized_key: str
    spellings: tuple[str, ...]


def report_nfc_collisions(keys: Iterable[str]) -> tuple[NfcCollision, ...]:
    if isinstance(keys, (str, bytes)):
        raise TypeError("keys must be a collection or stream of strings")

    originals: dict[str, set[str]] = defaultdict(set)
    for key in keys:
        if type(key) is not str:
            raise TypeError("each key must be a string")
        originals[normalize("NFC", key)].add(key)

    return tuple(
        NfcCollision(normalized, tuple(sorted(spellings)))
        for normalized, spellings in sorted(originals.items())
        if len(spellings) > 1
    )
```

## Example

```python
keys = (
    "zone-\u212b",
    "caf\u00e9",
    "zone-A\u030a",
    "cafe\u0301",
    "zone-\u00c5",
    "zone-\u00e5",
    "zone-\u00c5",
    "CAFE\u0301",
    "spare",
    "spare",
)

expected = (
    NfcCollision("caf\u00e9", ("cafe\u0301", "caf\u00e9")),
    NfcCollision("zone-\u00c5", ("zone-A\u030a", "zone-\u00c5", "zone-\u212b")),
)
assert report_nfc_collisions(keys) == report_nfc_collisions(reversed(keys)) == expected
```

## Trade-offs and Limitations

The report retains distinct original spellings in memory until it has
consumed the batch. Sort order is Python string order rather than a locale's
collation. Preserve the original spellings in any report so reviewers can see
exactly which keys would share a destination key.

Repeated identical spellings represent one member. Apply a separate policy
if duplicate rows with the same exact key are also invalid. This audit is
case-sensitive and implements NFC only. If the destination also folds case,
applies compatibility normalization, or uses custom collation, audit with
the destination's complete equality rule.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Canonical Unicode Caseless Comparison Key](../algorithms-data-structures/build-a-canonical-unicode-caseless-comparison-key.md)
- [Project a Bounded Document Snapshot into Collision-Audited Secondary Index Rows](project-a-bounded-document-snapshot-into-collision-audited-secondary-index-rows.md)
<!-- catalog:related:end -->

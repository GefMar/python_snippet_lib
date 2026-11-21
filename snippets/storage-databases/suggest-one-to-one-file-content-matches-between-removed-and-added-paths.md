---
title: "Suggest One-to-One File Content Matches Between Removed and Added Paths"
snippet_type: recipe
use_cases:
  - automation
  - persistence
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compare-bounded-apparent-sizes-of-two-file-tree-snapshots.md
  - plan-collision-free-renames-in-a-bounded-flat-file-namespace.md
---

# Suggest One-to-One File Content Matches Between Removed and Added Paths

## Idea and Problem

Pair vanished and newly discovered paths when their content signature matches exactly once on each side of a snapshot comparison.

Start with names present in only one snapshot. Group these paths by the
combination of size and content digest. A group with one old path and one new
path is a useful review candidate. Larger groups leave several possible
pairings, so omit them. The result is ordered by the old path.

## When to Use

Use this for a change report over two trusted regular-file inventories that
already contain byte sizes and SHA-256 digests. Both snapshots must use the
same path namespace and digest representation. The input scope is paths that
appeared or disappeared. A name present in both snapshots belongs to a
separate content-change comparison.

## Implementation

```python
from collections import defaultdict
from collections.abc import Mapping, Set


type ContentSignature = tuple[int, bytes]


def _paths_by_signature(
    snapshot: Mapping[str, ContentSignature], paths: Set[str]
) -> dict[ContentSignature, list[str]]:
    grouped: dict[ContentSignature, list[str]] = defaultdict(list)
    for name in paths:
        grouped[snapshot[name]].append(name)
    return grouped


def suggest_content_matches(
    before: Mapping[str, ContentSignature],
    after: Mapping[str, ContentSignature],
) -> tuple[tuple[str, str], ...]:
    """Return old and new paths for signatures unique among changes on each side."""
    removed = _paths_by_signature(before, before.keys() - after.keys())
    added = _paths_by_signature(after, after.keys() - before.keys())

    matches = []
    for signature, old_names in removed.items():
        new_names = added.get(signature)
        if len(old_names) == 1 and new_names is not None and len(new_names) == 1:
            matches.append((old_names[0], new_names[0]))
    return tuple(sorted(matches))
```

## Example

```python
from hashlib import sha256


def signature(data: bytes) -> ContentSignature:
    return len(data), sha256(data).digest()


report = signature(b"report")
shared = signature(b"shared")
before = {
    "draft.txt": report,
    "shared-old.txt": shared,
    "current.txt": signature(b"previous contents"),
}
after = {
    "published.txt": report,
    "shared-copy-a.txt": shared,
    "shared-copy-b.txt": shared,
    "current.txt": report,
}

assert suggest_content_matches(before, after) == (("draft.txt", "published.txt"),)
```

## Trade-offs and Limitations

The function reads the supplied inventories and holds the added and removed
paths in memory. The uniqueness check applies within those two sets, so a
path present in both snapshots does not affect the match. Review same-name
changes, copies and ambiguous groups separately when they matter.

Matching size and digest is evidence of equal content across snapshots.
Deletion followed by an independent copy can produce the same evidence as a
move. Base any filesystem action on the current files and a separate move
policy.

## Related Snippets

<!-- catalog:related:start -->
- [Compare Bounded Apparent Sizes of Two File-Tree Snapshots](compare-bounded-apparent-sizes-of-two-file-tree-snapshots.md)
- [Plan Collision-Free Renames in a Bounded Flat File Namespace](plan-collision-free-renames-in-a-bounded-flat-file-namespace.md)
<!-- catalog:related:end -->

---
title: "Report Missing and Extra POSIX Permission Bits Without Changing a File"
snippet_type: recipe
use_cases:
  - validation
  - observability
tested_python:
  - "3.14"
dependencies: []
related:
  - replace-a-file-atomically-with-a-sibling-temporary-file.md
  - ../security-privacy/audit-symlinks-in-a-bounded-directory-tree.md
---

# Report Missing and Extra POSIX Permission Bits Without Changing a File

## Idea and Problem

Find both missing and unexpected read, write, and execute bits in a POSIX mode snapshot.

Mask out file type and special bits, then compute the bit difference in each direction. A single inequality test cannot tell which permissions are absent and which are extra.

## When to Use

Use this for an audit that already has a stat mode and an expected ordinary permission mask. The return value contains the missing and extra masks, in that order.

## Implementation

```python
from stat import S_IMODE


def permission_drift(mode: int, expected: int) -> tuple[int, int]:
    if type(expected) is not int or not 0 <= expected <= 0o777:
        raise ValueError("expected must contain only ordinary permission bits")

    actual = S_IMODE(mode) & 0o777
    return expected & ~actual, actual & ~expected
```

## Example

```python
from stat import S_IFREG, S_ISUID

observed = S_IFREG | S_ISUID | 0o624
assert (permission_drift(observed, 0o640), permission_drift(observed, 0o624)) == (
    (0o040, 0o024),
    (0, 0),
)
```

## Trade-offs and Limitations

The input is a snapshot, not proof of current access for a particular identity. This intentionally does not audit ownership, ACLs, special mode bits, or symbolic-link policy, and never changes the file.

## Related Snippets

<!-- catalog:related:start -->
- [Replace a File Atomically with a Sibling Temporary File](replace-a-file-atomically-with-a-sibling-temporary-file.md)
- [Audit Symlinks in a Bounded Directory Tree](../security-privacy/audit-symlinks-in-a-bounded-directory-tree.md)
<!-- catalog:related:end -->

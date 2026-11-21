---
title: "Enforce Unique Contiguous Ordered Codes with IntEnum"
snippet_type: idiom
use_cases:
  - interoperability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/decode-a-closed-integer-capability-mask-with-strict-intflag-boundaries.md
---

# Enforce Unique Contiguous Ordered Codes with IntEnum

## Idea and Problem

Reject accidental aliases and gaps when integer codes represent ordered stages in a closed workflow.

`IntEnum` members compare by number. `verify(UNIQUE, CONTINUOUS)` checks
that one name owns each code and that all integers from the lowest to the
highest declared code are present. These checks run when the enum is
defined, so an invalid code table fails before application work starts.

## When to Use

Use this when a small set of stages must occupy one contiguous integer range
and numeric comparisons are part of the application. State the intended
values explicitly so their mapping is visible. Bit positions and code tables
with reserved gaps have different rules.

## Implementation

```python
from enum import CONTINUOUS, UNIQUE, IntEnum, verify


@verify(UNIQUE, CONTINUOUS)
class ReportStage(IntEnum):
    VALIDATED = 41
    RECEIVED = 40
    ARCHIVED = 42
```

## Example

```python
def rejected(members: dict[str, int]) -> bool:
    try:
        verify(UNIQUE, CONTINUOUS)(IntEnum("FixtureStage", members))
    except ValueError:
        return True
    return False


aliases = {"RECEIVED": 40, "QUEUED": 40, "VALIDATED": 41}
gap = {"RECEIVED": 40, "ARCHIVED": 42}

assert (
    tuple(member.value for member in ReportStage),
    ReportStage.RECEIVED < ReportStage.VALIDATED < ReportStage.ARCHIVED,
    ReportStage(41) is ReportStage.VALIDATED,
    rejected(aliases),
    rejected(gap),
) == ((41, 40, 42), True, True, True, True)
```

## Trade-offs and Limitations

Contiguity does not force a zero start or ascending declaration order. The
example starts at 40 and deliberately declares 41 first. Iteration follows
declaration order, while numeric comparisons use the values.

These checks apply to one definition, so renumbering a previously published
table can pass both checks. Assert the exact name-to-code mapping separately
when it is a storage or interface contract. Decide separately which stage
transitions are valid. A numeric comparison alone cannot enforce them.

## Related Snippets

<!-- catalog:related:start -->
- [Decode a Closed Integer Capability Mask with Strict IntFlag Boundaries](../configuration-serialization/decode-a-closed-integer-capability-mask-with-strict-intflag-boundaries.md)
<!-- catalog:related:end -->

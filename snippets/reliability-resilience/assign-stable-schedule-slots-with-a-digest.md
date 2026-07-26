---
title: "Assign Stable Schedule Slots with a Digest"
snippet_type: algorithm
use_cases:
  - performance-optimization
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related: []
---

# Assign Stable Schedule Slots with a Digest

## Idea and Problem

Assign a string key reproducibly to one of a fixed number of schedule slots using a domain-separated digest instead of Python's process-randomized hash.

Stable slot assignment spreads recurring work without storing a separate
schedule for every key. UTF-8 encoding, a fixed BLAKE2b configuration, and an
explicit byte order make the result reproducible across interpreter processes
as long as the algorithm contract remains unchanged.

## When to Use

Use this algorithm when the same key should repeatedly select the same slot and
occasional imbalance is acceptable. It suits coarse load spreading for
periodic work, not strict capacity allocation. Normalize keys before calling
only when the application intentionally considers multiple Unicode spellings
equivalent.

## Implementation

```python
from hashlib import blake2b


_SLOT_PERSONALIZATION = b"schedule-slot-v1"
_MAX_SLOT_COUNT = 1 << 64


def stable_schedule_slot(key: str, *, slot_count: int) -> int:
    if not isinstance(key, str):
        raise TypeError("key must be text")
    if isinstance(slot_count, bool) or not isinstance(slot_count, int):
        raise TypeError("slot_count must be an integer")
    if slot_count <= 0:
        raise ValueError("slot_count must be positive")
    if slot_count > _MAX_SLOT_COUNT:
        raise ValueError("slot_count exceeds the 64-bit digest space")

    digest = blake2b(
        key.encode("utf-8"),
        digest_size=8,
        person=_SLOT_PERSONALIZATION,
    ).digest()
    return int.from_bytes(digest, byteorder="big") % slot_count
```

## Example

```python
known_slots = (
    stable_schedule_slot("", slot_count=24),
    stable_schedule_slot("alpha", slot_count=24),
    stable_schedule_slot("schedule-key", slot_count=24),
)
unicode_slots = (
    stable_schedule_slot("\u00e9", slot_count=24),
    stable_schedule_slot("e\u0301", slot_count=24),
)
range_is_valid = all(
    0 <= stable_schedule_slot(f"item-{index}", slot_count=7) < 7
    for index in range(100)
)

try:
    stable_schedule_slot("key", slot_count=0)
except ValueError:
    zero_slots_rejected = True
else:
    zero_slots_rejected = False

try:
    stable_schedule_slot("key", slot_count=True)
except TypeError:
    bool_slots_rejected = True
else:
    bool_slots_rejected = False

try:
    stable_schedule_slot("key", slot_count=2**64 + 1)
except ValueError:
    excessive_slots_rejected = True
else:
    excessive_slots_rejected = False

assert (
    known_slots,
    unicode_slots,
    stable_schedule_slot("anything", slot_count=1),
    range_is_valid,
    zero_slots_rejected,
    bool_slots_rejected,
    excessive_slots_rejected,
) == ((3, 11, 8), (21, 22), 0, True, True, True, True)
```

## Trade-offs and Limitations

Changing the digest settings, encoding, personalization, or slot count changes
assignments; a rollout may need an explicit migration. Modulo reduction does
not guarantee even occupancy for a small or structured key set, and the
64-bit digest limits the slot count to at most `2**64`. Byte-distinct Unicode
strings remain distinct unless the caller normalizes them. This public,
unkeyed mapping is not suitable for adversarial isolation, authorization, or
secret bucketing.

## Related Snippets

<!-- catalog:related:start -->
No related snippets.
<!-- catalog:related:end -->

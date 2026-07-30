---
title: "Update One Bounded tracestate Member While Preserving Peer Order"
snippet_type: recipe
use_cases:
  - interoperability
  - networking
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - parse-and-format-a-strict-w3c-traceparent-version-00-value.md
  - extract-bounded-singleton-http-headers-with-explicit-rules.md
  - parse-one-bounded-printable-ascii-http-1-field-section-with-exact-crlf-framing.md
---

# Update One Bounded tracestate Member While Preserving Peer Order

## Idea and Problem

Update one validated member of a bounded canonical tracestate while keeping every peer in its existing relative order.

The input is already structured into frozen key-value members rather than raw
HTTP text. The replacement is moved to the left. Replacing an existing key
keeps every peer; adding a new key to a full 32-member state evicts exactly the
rightmost member. The result is either complete and within the local
512-character limit or rejected without a partial value.

## When to Use

Use this recipe at a tracing propagation boundary after another layer has
selected and parsed canonical Trace Context Level 2 state under this exact key
and value profile. It is useful when one participant owns one member and must
prepend its updated value without reordering other participants.

Use a maintained tracing SDK when raw header parsing, multiple HTTP fields,
optional whitespace, compatibility with broader key grammars, `traceparent`
preconditions, privacy controls, or vendor-specific truncation policy matters.
Those responsibilities are intentionally outside this state transformation.

## Implementation

```python
import re
from dataclasses import dataclass

_MAX_MEMBERS = 32
_MAX_RENDERED_CHARACTERS = 512
_KEY = re.compile(
    r"[a-z0-9][a-z0-9_*/@-]{0,255}",
    re.ASCII,
).fullmatch


class TraceStateError(ValueError):
    """Raised when members violate this closed tracestate profile."""


@dataclass(frozen=True, slots=True)
class TraceStateMember:
    key: str
    value: str


@dataclass(frozen=True, slots=True)
class TraceState:
    members: tuple[TraceStateMember, ...]
    rendered: str


def _validate_member(member: TraceStateMember) -> None:
    if type(member) is not TraceStateMember:
        raise TypeError("every member must be an exact TraceStateMember")
    if type(member.key) is not str:
        raise TypeError("member.key must be an exact string")
    if _KEY(member.key) is None:
        raise TraceStateError("member.key is outside the closed key grammar")
    if type(member.value) is not str:
        raise TypeError("member.value must be an exact string")
    if not 1 <= len(member.value) <= 256:
        raise TraceStateError("member.value must contain from 1 through 256 characters")
    if any(not 0x20 <= ord(character) <= 0x7E or character in ",=" for character in member.value):
        raise TraceStateError("member.value is outside the closed ASCII grammar")
    if member.value.endswith(" "):
        raise TraceStateError("member.value must end in a non-space character")


def _render_members(members: tuple[TraceStateMember, ...]) -> str:
    return ",".join(f"{member.key}={member.value}" for member in members)


def _validate_current(
    members: tuple[TraceStateMember, ...],
) -> None:
    if type(members) is not tuple:
        raise TypeError("members must be an exact tuple")
    if len(members) > _MAX_MEMBERS:
        raise TraceStateError("members exceed the 32-member limit")

    seen: set[str] = set()
    for member in members:
        _validate_member(member)
        if member.key in seen:
            raise TraceStateError("member keys must be unique")
        seen.add(member.key)
    if len(_render_members(members)) > _MAX_RENDERED_CHARACTERS:
        raise TraceStateError("current state exceeds the local character limit")


def update_tracestate_member(
    members: tuple[TraceStateMember, ...],
    replacement: TraceStateMember,
) -> TraceState:
    """Prepend one member and preserve the relative order of its peers."""
    _validate_current(members)
    _validate_member(replacement)

    replacing = any(member.key == replacement.key for member in members)
    peers = tuple(member for member in members if member.key != replacement.key)
    if not replacing and len(members) == _MAX_MEMBERS:
        peers = peers[:-1]

    updated = (replacement, *peers)
    rendered = _render_members(updated)
    if len(rendered) > _MAX_RENDERED_CHARACTERS:
        raise TraceStateError("updated state exceeds the local character limit")
    return TraceState(members=updated, rendered=rendered)
```

## Example

```python


current = (
    TraceStateMember("peer_a", "opaque-a"),
    TraceStateMember("mine", "old"),
    TraceStateMember("peer_b", "opaque-b"),
)
replaced = update_tracestate_member(
    current,
    TraceStateMember("mine", "new"),
)
assert replaced == TraceState(
    members=(
        TraceStateMember("mine", "new"),
        TraceStateMember("peer_a", "opaque-a"),
        TraceStateMember("peer_b", "opaque-b"),
    ),
    rendered="mine=new,peer_a=opaque-a,peer_b=opaque-b",
)

full = tuple(TraceStateMember(f"peer{i}", "v") for i in range(32))
extended = update_tracestate_member(
    full,
    TraceStateMember("mine", "new"),
)
assert len(extended.members) == 32
assert extended.members[:2] == (
    TraceStateMember("mine", "new"),
    TraceStateMember("peer0", "v"),
)
assert extended.members[-1] == TraceStateMember("peer30", "v")
assert TraceStateMember("peer31", "v") not in extended.members

near_limit = (
    TraceStateMember("a", "x" * 250),
    TraceStateMember("b", "y" * 250),
)
assert len(_render_members(near_limit)) == 505
try:
    update_tracestate_member(
        near_limit,
        TraceStateMember("c", "z" * 10),
    )
except TraceStateError:
    oversized_result_rejected = True
else:
    oversized_result_rejected = False

assert oversized_result_rejected
assert near_limit == (
    TraceStateMember("a", "x" * 250),
    TraceStateMember("b", "y" * 250),
)
```

## Trade-offs and Limitations

Validation and rendering use `O(N + C)` time and `O(N + C)` result memory for
at most 32 members and 512 rendered characters. A replacement scans the small
tuple and allocates a new immutable result. On new-key overflow, evicting only
the rightmost member makes the order rule explicit; the character cap never
causes additional eviction or partial output.

This is a closed local propagation profile, not a raw `tracestate` parser or a
complete implementation of every standard key form and truncation rule. It
does not select HTTP fields, normalize whitespace, validate `traceparent`,
handle baggage, assign member ownership, inspect opaque values, or decide what
may cross a trust or privacy boundary.

## Related Snippets

<!-- catalog:related:start -->
- [Parse and Format a Strict W3C traceparent Version 00 Value](parse-and-format-a-strict-w3c-traceparent-version-00-value.md)
- [Extract Bounded Singleton HTTP Headers with Explicit Rules](extract-bounded-singleton-http-headers-with-explicit-rules.md)
- [Parse One Bounded Printable-ASCII HTTP/1 Field Section with Exact CRLF Framing](parse-one-bounded-printable-ascii-http-1-field-section-with-exact-crlf-framing.md)
<!-- catalog:related:end -->

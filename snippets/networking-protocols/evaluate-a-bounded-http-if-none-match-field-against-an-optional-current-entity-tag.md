---
title: "Evaluate a Bounded HTTP If-None-Match Field Against an Optional Current Entity Tag"
snippet_type: recipe
use_cases:
  - interoperability
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - apply-an-http-patch-only-when-a-strong-etag-still-matches.md
  - extract-bounded-singleton-http-headers-with-explicit-rules.md
  - normalize-a-closed-http-byte-range-value-into-bounded-half-open-spans.md
---

# Evaluate a Bounded HTTP If-None-Match Field Against an Optional Current Entity Tag

## Idea and Problem

Parse one deliberately closed HTTP If-None-Match field value and evaluate its weak-comparison condition against an optional current entity tag.

Entity-tag opaque values may contain commas, so splitting the field on every
comma is incorrect. A small structural scanner recognizes separators only
outside quoted opaque tags. Weak comparison then compares opaque values while
ignoring whether either tag carries the `W/` weakness prefix.

The wildcard has separate semantics: it fails the condition exactly when a
current selected representation exists. The result reports only whether the
condition is satisfied and which source member matched; response status and
method semantics remain the caller's responsibility.

## When to Use

Use this recipe at a bounded HTTP adapter that has already combined policy into
one field value and needs a reviewable conditional-match decision. It is useful
for protocol fixtures, cache-adjacent components, and narrow handlers that can
represent current-resource absence explicitly as `None`.

Use a maintained HTTP implementation for general message processing. Repeated
fields, legacy bytes, method-dependent status selection, precedence among
preconditions, and cache behavior require a wider protocol boundary.

## Implementation

```python
from dataclasses import dataclass

_MAX_FIELD_CHARACTERS = 8_192
_MAX_MEMBERS = 64
_MAX_OPAQUE_CHARACTERS = 128
_MAX_ENTITY_TAG_CHARACTERS = 132


class IfNoneMatchError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class IfNoneMatchResult:
    condition_satisfied: bool
    matched_member: str | None


@dataclass(frozen=True, slots=True)
class _EntityTag:
    opaque: str
    source: str


def _require_printable_ascii(value: str, *, field: str) -> None:
    if any(not 0x20 <= ord(character) <= 0x7E for character in value):
        raise IfNoneMatchError(f"{field} must contain printable ASCII only")


def _is_opaque_character(character: str) -> bool:
    code_point = ord(character)
    return code_point == 0x21 or 0x23 <= code_point <= 0x7E


def _scan_entity_tag(text: str, position: int) -> tuple[_EntityTag, int]:
    member_start = position
    if text.startswith("W/", position):
        position += 2
    if position >= len(text) or text[position] != '"':
        raise IfNoneMatchError("an entity tag must start with an exact quote")
    position += 1
    opaque_start = position

    while position < len(text) and text[position] != '"':
        if not _is_opaque_character(text[position]):
            raise IfNoneMatchError("an entity tag has an invalid opaque character")
        if position - opaque_start >= _MAX_OPAQUE_CHARACTERS:
            raise IfNoneMatchError("an entity tag opaque value is too long")
        position += 1

    if position >= len(text):
        raise IfNoneMatchError("an entity tag is missing its closing quote")
    opaque = text[opaque_start:position]
    position += 1
    return _EntityTag(opaque, text[member_start:position]), position


def _parse_current_entity_tag(current: object) -> _EntityTag | None:
    if current is None:
        return None
    if type(current) is not str:
        raise TypeError("current_entity_tag must be an exact string or None")
    if not 2 <= len(current) <= _MAX_ENTITY_TAG_CHARACTERS:
        raise ValueError("current_entity_tag length is outside the grammar")
    _require_printable_ascii(current, field="current_entity_tag")
    parsed, stop = _scan_entity_tag(current, 0)
    if stop != len(current):
        raise IfNoneMatchError("current_entity_tag must contain one exact tag")
    return parsed


def _parse_if_none_match_list(text: str) -> tuple[_EntityTag, ...]:
    members: list[_EntityTag] = []
    position = 0

    while True:
        member, position = _scan_entity_tag(text, position)
        members.append(member)
        if len(members) > _MAX_MEMBERS:
            raise IfNoneMatchError("the field contains too many entity tags")

        while position < len(text) and text[position] == " ":
            position += 1
        if position == len(text):
            return tuple(members)
        if text[position] != ",":
            raise IfNoneMatchError("entity tags must be separated by commas")
        position += 1
        while position < len(text) and text[position] == " ":
            position += 1
        if position == len(text):
            raise IfNoneMatchError("the field must not end with an empty member")


def evaluate_if_none_match(
    field_value: str,
    current_entity_tag: str | None,
) -> IfNoneMatchResult:
    if type(field_value) is not str:
        raise TypeError("field_value must be an exact string")
    if not 1 <= len(field_value) <= _MAX_FIELD_CHARACTERS:
        raise ValueError("field_value length is outside the supported range")
    _require_printable_ascii(field_value, field="field_value")

    text = field_value.strip(" ")
    if not text:
        raise IfNoneMatchError("field_value must not be empty after trimming")
    current = _parse_current_entity_tag(current_entity_tag)

    if text == "*":
        return IfNoneMatchResult(
            condition_satisfied=current is None,
            matched_member="*" if current is not None else None,
        )

    for member in _parse_if_none_match_list(text):
        if current is not None and member.opaque == current.opaque:
            return IfNoneMatchResult(False, member.source)
    return IfNoneMatchResult(True, None)
```

## Example

```python
field = r' W/"rev,1" , "other\tag" '

assert evaluate_if_none_match(field, '"rev,1"') == IfNoneMatchResult(
    condition_satisfied=False,
    matched_member='W/"rev,1"',
)
assert evaluate_if_none_match(field, r'W/"other\tag"') == IfNoneMatchResult(
    condition_satisfied=False,
    matched_member=r'"other\tag"',
)
assert evaluate_if_none_match(field, '"missing"') == IfNoneMatchResult(True, None)
assert evaluate_if_none_match("*", None) == IfNoneMatchResult(True, None)
assert evaluate_if_none_match("*", 'W/"present"') == IfNoneMatchResult(
    condition_satisfied=False,
    matched_member="*",
)
```

## Trade-offs and Limitations

Scanning and comparison take `O(f)` time for at most 8,192 field characters.
The parser retains at most 64 small entity-tag records, so auxiliary space is
bounded by the admitted field size. Matching stops at the first source member
whose opaque value weakly matches the current tag.

This is intentionally narrower than the complete HTTP grammar. It accepts
only printable ASCII, exact uppercase `W/`, spaces rather than tabs around
separators, and opaque values of at most 128 characters. Commas and backslashes
inside quotes are literal; backslash does not escape the following character.
Empty opaque values are valid, while empty list members are not.

The function does not combine repeated fields, accept obsolete text, generate
entity tags, select a response status, distinguish safe from unsafe methods,
evaluate other preconditions, or implement cache validation. The matched
member is syntax evidence from the supplied field, not proof that any payloads
are byte-for-byte identical.

## Related Snippets

<!-- catalog:related:start -->
- [Apply an HTTP Patch Only When a Strong ETag Still Matches](apply-an-http-patch-only-when-a-strong-etag-still-matches.md)
- [Extract Bounded Singleton HTTP Headers with Explicit Rules](extract-bounded-singleton-http-headers-with-explicit-rules.md)
- [Normalize a Closed HTTP Byte-Range Value into Bounded Half-Open Spans](normalize-a-closed-http-byte-range-value-into-bounded-half-open-spans.md)
<!-- catalog:related:end -->

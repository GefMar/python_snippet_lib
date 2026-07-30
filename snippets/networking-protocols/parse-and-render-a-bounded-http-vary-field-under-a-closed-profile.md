---
title: "Parse and Render a Bounded HTTP Vary Field Under a Closed Profile"
snippet_type: recipe
use_cases:
  - interoperability
  - networking
  - parsing
  - serialization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-response-cache-control-field-under-a-closed-profile.md
  - parse-and-render-a-bounded-server-timing-field-under-a-closed-profile.md
  - extract-bounded-singleton-http-headers-with-explicit-rules.md
---

# Parse and Render a Bounded HTTP Vary Field Under a Closed Profile

## Idea and Problem

Normalize one already-combined bounded HTTP Vary field into either a wildcard or a sorted immutable tuple of unique lowercase field names.

The parser accepts only ASCII field-name tokens with optional space or tab
around comma members. It rejects empty and case-insensitively duplicate members,
while the renderer checks the complete normalized model before producing one
deterministic spelling.

## When to Use

Use this recipe at a controlled HTTP boundary where the field values have
already been combined and a strict local representation simplifies fixtures,
configuration comparisons, cache metadata, or deterministic serialization.
Rejecting duplicate names exposes producer mistakes instead of silently hiding
them during normalization.

Use a maintained HTTP cache implementation for general traffic. `Vary` affects
which selected request fields extend a cache key, but parsing the field alone
does not determine cacheability, freshness, request matching, validation, or
whether a representation may be reused.

## Implementation

```python
from dataclasses import dataclass

_MAX_FIELD_LENGTH = 2_048
_MAX_MEMBERS = 32
_MAX_FIELD_NAME_LENGTH = 64
_TOKEN_CHARACTERS = frozenset(
    "!#$%&'*+-.^_`|~0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
)


class VaryProfileError(ValueError):
    """The value is outside this closed Vary-field profile."""


@dataclass(frozen=True, slots=True)
class VaryProfile:
    members: tuple[str, ...]


def _validate_field_name(name: str) -> None:
    if type(name) is not str:
        raise TypeError("every member must be an exact string")
    if not 1 <= len(name) <= _MAX_FIELD_NAME_LENGTH:
        raise VaryProfileError("field-name length is outside the profile")
    if any(character not in _TOKEN_CHARACTERS for character in name):
        raise VaryProfileError("field name is not an ASCII token")


def render_vary_field(profile: VaryProfile) -> str:
    if type(profile) is not VaryProfile:
        raise TypeError("profile must be an exact VaryProfile")
    if type(profile.members) is not tuple:
        raise TypeError("profile.members must be an exact tuple")
    if not 1 <= len(profile.members) <= _MAX_MEMBERS:
        raise VaryProfileError("member count is outside the profile")
    for member in profile.members:
        _validate_field_name(member)
    if profile.members == ("*",):
        return "*"

    for member in profile.members:
        if member == "*":
            raise VaryProfileError("wildcard must be the sole member")
        if member != member.lower():
            raise VaryProfileError("normalized field names must be lowercase")
    if any(
        left >= right for left, right in zip(profile.members, profile.members[1:], strict=False)
    ):
        raise VaryProfileError("field names must be strictly sorted and unique")

    rendered = ", ".join(profile.members)
    if len(rendered) > _MAX_FIELD_LENGTH:
        raise VaryProfileError("rendered field exceeds the supported size")
    return rendered


def parse_vary_field(field_value: str) -> VaryProfile:
    if type(field_value) is not str:
        raise TypeError("field_value must be an exact string")
    if not 1 <= len(field_value) <= _MAX_FIELD_LENGTH:
        raise VaryProfileError("field length is outside the supported range")
    if not field_value.isascii():
        raise VaryProfileError("field must contain ASCII only")

    raw_members = field_value.split(",")
    if not 1 <= len(raw_members) <= _MAX_MEMBERS:
        raise VaryProfileError("member count is outside the profile")
    members = tuple(raw_member.strip(" \t") for raw_member in raw_members)
    if any(not member for member in members):
        raise VaryProfileError("members must not be empty")
    if "*" in members:
        if members != ("*",):
            raise VaryProfileError("wildcard must be the sole member")
        return VaryProfile(("*",))

    normalized: list[str] = []
    seen: set[str] = set()
    for member in members:
        _validate_field_name(member)
        lowercase = member.lower()
        if lowercase in seen:
            raise VaryProfileError("field names must be case-insensitively unique")
        seen.add(lowercase)
        normalized.append(lowercase)

    profile = VaryProfile(tuple(sorted(normalized)))
    render_vary_field(profile)
    return profile
```

## Example

```python


profile = parse_vary_field(" User-Agent\t, Accept-Encoding , X-View ")
wildcard = parse_vary_field("\t* ")

invalid_values = (
    "",
    "Accept,,User-Agent",
    "Accept, ACCEPT",
    "*, Accept-Encoding",
    "Accept Encoding",
    "Accept\r",
    "x" * 65,
)
rejected = []
for invalid_value in invalid_values:
    try:
        parse_vary_field(invalid_value)
    except VaryProfileError:
        rejected.append(invalid_value)

invalid_models = (
    VaryProfile(("user-agent", "accept-encoding")),
    VaryProfile(("Accept",)),
    VaryProfile(("accept", "accept")),
)
for invalid_model in invalid_models:
    try:
        render_vary_field(invalid_model)
    except VaryProfileError:
        rejected.append(invalid_model)

assert (
    profile,
    render_vary_field(profile),
    wildcard,
    render_vary_field(wildcard),
    tuple(rejected),
) == (
    VaryProfile(("accept-encoding", "user-agent", "x-view")),
    "accept-encoding, user-agent, x-view",
    VaryProfile(("*",)),
    "*",
    invalid_values + invalid_models,
)
```

## Trade-offs and Limitations

Parsing and rendering are linear in the bounded field length apart from sorting
at most 32 names. Lowercasing and lexical sorting are local canonicalization
rules, not HTTP requirements. This profile also deliberately rejects duplicate
names, empty members, non-ASCII input, and otherwise valid deployments whose
field names or combined values exceed its fixed bounds.

The parser expects one already-combined field value and does not combine
repeated HTTP field lines. It does not read the selected request fields, build
cache keys, compare request values, choose representations, enforce privacy
policy, or implement any other cache behavior. A proxy must not generate the
wildcard form even though a recipient can parse it and this model can render it
for deterministic storage or testing.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded Response Cache-Control Field Under a Closed Profile](parse-a-bounded-response-cache-control-field-under-a-closed-profile.md)
- [Parse and Render a Bounded Server-Timing Field Under a Closed Profile](parse-and-render-a-bounded-server-timing-field-under-a-closed-profile.md)
- [Extract Bounded Singleton HTTP Headers with Explicit Rules](extract-bounded-singleton-http-headers-with-explicit-rules.md)
<!-- catalog:related:end -->

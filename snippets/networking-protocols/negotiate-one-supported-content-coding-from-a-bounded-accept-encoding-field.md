---
title: "Negotiate One Supported Content Coding from a Bounded Accept-Encoding Field"
snippet_type: recipe
use_cases:
  - interoperability
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - negotiate-one-supported-media-type-from-a-bounded-accept-field.md
  - parse-and-rank-a-bounded-accept-language-value.md
  - extract-bounded-singleton-http-headers-with-explicit-rules.md
---

# Negotiate One Supported Content Coding from a Bounded Accept-Encoding Field

## Idea and Problem

Choose one server-supported content coding from a strict bounded HTTP Accept-Encoding field while preserving absence, empty-value, wildcard, and identity semantics.

Every quality is represented as exact integer thousandths. An explicit coding
member overrides the wildcard. The unencoded `identity` representation remains
quality 1000 unless an exact identity member overrides it or a zero-quality
wildcard excludes it. Highest positive quality wins, with declared server order
as the deterministic tie break.

## When to Use

Use this recipe after HTTP field combination at a boundary that supports a
small fixed set of content codings and can always serve `identity`. Pass offers
in server-preference order and include `identity` exactly once. The return value
tells representation-selection code which coding is acceptable; it does not
perform compression.

Use a full HTTP framework when repeated-field combination, representation
metadata, content transformation, cache variation, response construction, or
protocol-version policy must be coordinated. Transfer codings belong to a
different field and are deliberately outside this function.

## Implementation

```python
import re
from dataclasses import dataclass

_MAX_FIELD_CHARACTERS = 8_192
_MAX_RAW_SLOTS = 128
_MAX_MEMBERS = 64
_MAX_OFFERS = 16
_MAX_TOKEN_CHARACTERS = 32
_TOKEN_TEXT = rf"[!#$%&'*+\-.^_`|~0-9A-Za-z]{{1,{_MAX_TOKEN_CHARACTERS}}}"
_TOKEN = re.compile(_TOKEN_TEXT, re.ASCII).fullmatch
_QVALUE = r"(?:0(?:\.[0-9]{0,3})?|1(?:\.0{0,3})?)"
_MEMBER = re.compile(
    rf"(?P<coding>{_TOKEN_TEXT}|\*)"
    rf"(?: *; *(?i:q)=(?P<quality>{_QVALUE}))?",
    re.ASCII,
).fullmatch


class AcceptEncodingError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class NegotiatedContentCoding:
    coding: str
    quality: int
    matched_member: str


def _quality_thousandths(raw: str | None) -> int:
    if raw is None:
        return 1_000
    whole, separator, fraction = raw.partition(".")
    if whole == "1":
        return 1_000
    if not separator or not fraction:
        return 0
    return int((fraction + "000")[:3])


def _validated_offers(value: object) -> tuple[str, ...]:
    if type(value) is not tuple:
        raise TypeError("supported must be an exact tuple")
    if not 1 <= len(value) <= _MAX_OFFERS:
        raise ValueError("supported count is outside 1..16")

    seen: set[str] = set()
    identity_count = 0
    for index, offer in enumerate(value):
        if type(offer) is not str:
            raise TypeError(f"supported[{index}] must be an exact string")
        if offer != offer.lower() or _TOKEN(offer) is None or offer == "*":
            raise AcceptEncodingError(
                "offers must be lowercase concrete HTTP tokens"
            )
        if offer in seen:
            raise AcceptEncodingError("offers must be unique")
        seen.add(offer)
        identity_count += offer == "identity"
    if identity_count != 1:
        raise AcceptEncodingError("offers must contain identity exactly once")
    return value


def _parse_members(field: object) -> dict[str, int] | None:
    if field is None:
        return None
    if type(field) is not str:
        raise TypeError("field must be exactly str or None")
    if len(field) > _MAX_FIELD_CHARACTERS:
        raise ValueError("field exceeds the supported length")
    if any(not 0x20 <= ord(character) <= 0x7E for character in field):
        raise ValueError("field must contain printable ASCII")

    raw_slots = field.split(",")
    if len(raw_slots) > _MAX_RAW_SLOTS:
        raise ValueError("field has too many raw comma slots")
    members: dict[str, int] = {}
    for raw_slot in raw_slots:
        member = raw_slot.strip(" ")
        if not member:
            continue
        if len(members) >= _MAX_MEMBERS:
            raise ValueError("field has too many non-empty members")
        match = _MEMBER(member)
        if match is None:
            raise AcceptEncodingError("a member is outside the closed grammar")
        coding = match.group("coding").lower()
        if coding in members:
            raise AcceptEncodingError(
                "members must be unique case-insensitively"
            )
        members[coding] = _quality_thousandths(match.group("quality"))
    return members


def negotiate_content_coding(
    field: str | None,
    supported: tuple[str, ...],
) -> NegotiatedContentCoding | None:
    offers = _validated_offers(supported)
    members = _parse_members(field)
    selected: NegotiatedContentCoding | None = None

    for offer in offers:
        if members is None:
            quality = 1_000
            matched = "<absent>"
        elif offer in members:
            quality = members[offer]
            matched = offer
        elif offer == "identity":
            wildcard_quality = members.get("*")
            quality = 0 if wildcard_quality == 0 else 1_000
            matched = "*" if wildcard_quality == 0 else "<identity-default>"
        elif "*" in members:
            quality = members["*"]
            matched = "*"
        else:
            quality = 0
            matched = "<unlisted>"

        if quality == 0:
            continue
        candidate = NegotiatedContentCoding(offer, quality, matched)
        if selected is None or candidate.quality > selected.quality:
            selected = candidate
    return selected
```

## Example

```python
def expected_choice(
    members: dict[str, int],
    offers: tuple[str, ...],
) -> tuple[str, int] | None:
    ranked: list[tuple[int, int, str]] = []
    for order, offer in enumerate(offers):
        if offer in members:
            quality = members[offer]
        elif offer == "identity":
            quality = 0 if members.get("*") == 0 else 1_000
        else:
            quality = members.get("*", 0)
        if quality > 0:
            ranked.append((-quality, order, offer))
    if not ranked:
        return None
    negative_quality, _, coding = min(ranked)
    return coding, -negative_quality


def exercise_small_preference_tables() -> int:
    from itertools import permutations, product

    quality_text = {0: "0", 500: "0.5", 1_000: "1"}
    names = ("br", "gzip", "identity", "*")
    offers_permutations = tuple(permutations(("br", "gzip", "identity")))
    checked = 0
    for choices in product((None, 0, 500, 1_000), repeat=len(names)):
        members = {
            name: quality
            for name, quality in zip(names, choices, strict=True)
            if quality is not None
        }
        field = ", ".join(
            f"{name};q={quality_text[quality]}"
            for name, quality in members.items()
        )
        for offers in offers_permutations:
            observed = negotiate_content_coding(field, offers)
            expected = expected_choice(members, offers)
            assert (
                None if observed is None else (observed.coding, observed.quality)
            ) == expected
            checked += 1
    return checked


checked = exercise_small_preference_tables()

absent = negotiate_content_coding(None, ("gzip", "identity", "br"))
empty = negotiate_content_coding("", ("gzip", "identity", "br"))

invalid_fields = (
    "gzip;q=1.1",
    "gzip;q=1;level=9",
    "gzip, GZIP",
    "gzip\t;q=1",
)
rejected = 0
for invalid in invalid_fields:
    try:
        negotiate_content_coding(invalid, ("gzip", "identity"))
    except (AcceptEncodingError, ValueError):
        rejected += 1

assert (
    checked == 1_536
    and absent == NegotiatedContentCoding("gzip", 1_000, "<absent>")
    and empty == NegotiatedContentCoding(
        "identity",
        1_000,
        "<identity-default>",
    )
    and negotiate_content_coding("*;q=0", ("br", "identity")) is None
    and negotiate_content_coding(
        "*;q=0, identity;q=0.5",
        ("br", "identity"),
    )
    == NegotiatedContentCoding("identity", 500, "identity")
    and rejected == len(invalid_fields)
)
```

## Trade-offs and Limitations

Parsing and selection take `O(F + S)` time and memory for field length `F` and
supported-offer count `S`. Integer thousandths avoid binary-float ordering
surprises. Rejecting duplicate members and extensions creates one auditable
interpretation instead of guessing how to combine ambiguous preferences.

An absent field makes every offer acceptable and therefore selects the first
server preference. A present field with no non-empty members selects only
`identity`. Implicit identity ignores a positive wildcard and keeps quality
1000, but a zero wildcard excludes it unless an exact identity member says
otherwise. The function does not combine repeated fields, compress content,
manage `Vary`, select transfer codings, or construct an HTTP response.

## Related Snippets

<!-- catalog:related:start -->
- [Negotiate One Supported Media Type from a Bounded Accept Field](negotiate-one-supported-media-type-from-a-bounded-accept-field.md)
- [Parse and Rank a Bounded Accept-Language Value](parse-and-rank-a-bounded-accept-language-value.md)
- [Extract Bounded Singleton HTTP Headers with Explicit Rules](extract-bounded-singleton-http-headers-with-explicit-rules.md)
<!-- catalog:related:end -->

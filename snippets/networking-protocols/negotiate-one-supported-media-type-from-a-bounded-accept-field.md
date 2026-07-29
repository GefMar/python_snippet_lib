---
title: "Negotiate One Supported Media Type from a Bounded Accept Field"
snippet_type: recipe
use_cases:
  - interoperability
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-ascii-media-type-value.md
  - parse-and-rank-a-bounded-accept-language-value.md
  - extract-bounded-singleton-http-headers-with-explicit-rules.md
---

# Negotiate One Supported Media Type from a Bounded Accept Field

## Idea and Problem

Choose one concrete server-supported media type from a strict bounded HTTP Accept field without using binary floating point for quality values.

Each supported offer first receives the quality of its most-specific matching
range. This ordering matters: an exact `q=0` exclusion overrides a positive
`type/*` or `*/*` range. Only after effective qualities are known does explicit
server offer order break equal positive-quality ties.

## When to Use

Use this recipe at a small HTTP boundary whose supported representations have
parameter-free concrete media types and whose accepted request profile needs
only exact ranges, type wildcards, global wildcards and qvalues. Pass offers in
the server's preferred order.

Use a standards-complete negotiation library when media parameters,
extensions, repeated field combination, structured suffix policy, client
order, representation metadata or cache-key management influence selection.
An absent `Accept` field should be handled before this function according to
the application's explicit default policy.

## Implementation

```python
import re
from dataclasses import dataclass

_MAX_FIELD_CHARACTERS = 8_192
_MAX_RAW_SLOTS = 128
_MAX_RANGES = 64
_MAX_OFFERS = 64
_MAX_TOKEN_CHARACTERS = 64
_MAX_OFFER_CHARACTERS = 129
_MAX_AGGREGATE_OFFER_CHARACTERS = 8_192
_TOKEN = rf"[!#$%&'*+\-.^_`|~0-9A-Za-z]{{1,{_MAX_TOKEN_CHARACTERS}}}"
_QVALUE = r"(?:0(?:\.[0-9]{0,3})?|1(?:\.0{0,3})?)"
_RANGE = re.compile(
    rf"(?P<type>{_TOKEN}|\*)/(?P<subtype>{_TOKEN}|\*)"
    rf"(?: *; *(?i:q)=(?P<quality>{_QVALUE}))?",
    re.ASCII,
).fullmatch
_CONCRETE_TYPE = re.compile(
    rf"(?P<type>{_TOKEN})/(?P<subtype>{_TOKEN})",
    re.ASCII,
).fullmatch


class AcceptNegotiationError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class NegotiatedMediaType:
    media_type: str
    quality: int
    matched_range: str


@dataclass(frozen=True, slots=True)
class _MediaRange:
    main_type: str
    subtype: str
    quality: int

    @property
    def text(self) -> str:
        return f"{self.main_type}/{self.subtype}"


def _quality_thousandths(raw_quality: str | None) -> int:
    if raw_quality is None:
        return 1_000
    whole, separator, fraction = raw_quality.partition(".")
    if whole == "1":
        return 1_000
    if not separator or not fraction:
        return 0
    return int((fraction + "000")[:3])


def _parse_ranges(text: str) -> tuple[_MediaRange, ...]:
    if type(text) is not str:
        raise TypeError("Accept field value must be an exact string")
    if len(text) > _MAX_FIELD_CHARACTERS:
        raise ValueError("Accept field exceeds the supported length")
    if any(not 0x20 <= ord(character) <= 0x7E for character in text):
        raise ValueError("Accept field must contain printable ASCII")

    raw_slots = text.split(",")
    if len(raw_slots) > _MAX_RAW_SLOTS:
        raise ValueError("Accept field has too many raw list slots")

    ranges: list[_MediaRange] = []
    seen: set[str] = set()
    for raw_slot in raw_slots:
        member = raw_slot.strip(" ")
        if not member:
            continue
        if len(ranges) >= _MAX_RANGES:
            raise ValueError("Accept field has too many non-empty ranges")
        match = _RANGE(member)
        if match is None:
            raise AcceptNegotiationError("a media range is outside the closed grammar")

        main_type = match.group("type").lower()
        subtype = match.group("subtype").lower()
        if main_type == "*" and subtype != "*":
            raise AcceptNegotiationError("a wildcard type requires a wildcard subtype")
        normalized = f"{main_type}/{subtype}"
        if normalized in seen:
            raise AcceptNegotiationError("media ranges must be unique case-insensitively")
        seen.add(normalized)
        ranges.append(
            _MediaRange(
                main_type,
                subtype,
                _quality_thousandths(match.group("quality")),
            )
        )
    return tuple(ranges)


def _parse_offers(offers: tuple[str, ...]) -> tuple[tuple[str, str, str], ...]:
    if type(offers) is not tuple:
        raise TypeError("offers must be an exact tuple")
    if not 1 <= len(offers) <= _MAX_OFFERS:
        raise ValueError("offer count is outside the supported range")

    aggregate_characters = 0
    parsed: list[tuple[str, str, str]] = []
    seen: set[str] = set()
    for offer in offers:
        if type(offer) is not str:
            raise TypeError("offers must contain exact strings")
        if not 1 <= len(offer) <= _MAX_OFFER_CHARACTERS:
            raise ValueError("an offer length is outside the supported range")
        aggregate_characters += len(offer)
        if aggregate_characters > _MAX_AGGREGATE_OFFER_CHARACTERS:
            raise ValueError("aggregate offer text exceeds the supported length")

        match = _CONCRETE_TYPE(offer)
        if match is None:
            raise AcceptNegotiationError("an offer is not a concrete media type")
        main_type = match.group("type").lower()
        subtype = match.group("subtype").lower()
        if main_type == "*" or subtype == "*":
            raise AcceptNegotiationError("offers cannot contain wildcards")
        normalized = f"{main_type}/{subtype}"
        if normalized in seen:
            raise AcceptNegotiationError("offers must be unique case-insensitively")
        seen.add(normalized)
        parsed.append((main_type, subtype, normalized))
    return tuple(parsed)


def _specificity(media_range: _MediaRange, main_type: str, subtype: str) -> int:
    if media_range.main_type == main_type and media_range.subtype == subtype:
        return 2
    if media_range.main_type == main_type and media_range.subtype == "*":
        return 1
    if media_range.main_type == "*" and media_range.subtype == "*":
        return 0
    return -1


def negotiate_media_type(
    accept: str,
    supported: tuple[str, ...],
) -> NegotiatedMediaType | None:
    ranges = _parse_ranges(accept)
    offers = _parse_offers(supported)
    selected: NegotiatedMediaType | None = None

    for main_type, subtype, normalized in offers:
        matches = tuple(
            (specificity, media_range)
            for media_range in ranges
            if (specificity := _specificity(media_range, main_type, subtype)) >= 0
        )
        if not matches:
            continue
        _, effective_range = max(matches, key=lambda item: item[0])
        if effective_range.quality == 0:
            continue
        candidate = NegotiatedMediaType(
            normalized,
            effective_range.quality,
            effective_range.text,
        )
        if selected is None or candidate.quality > selected.quality:
            selected = candidate
    return selected
```

## Example

```python
selected = negotiate_media_type(
    "text/*;q=0.7, text/html;q=0, */*;q=0.5",
    ("TEXT/HTML", "application/json", "text/plain"),
)
server_tie = negotiate_media_type(
    "application/*;q=0.8",
    ("application/json", "application/cbor"),
)
excluded = negotiate_media_type("text/html;q=0, */*;q=1", ("text/html",))

assert (selected, server_tie, excluded) == (
    NegotiatedMediaType("text/plain", 700, "text/*"),
    NegotiatedMediaType("application/json", 800, "application/*"),
    None,
)
```

## Trade-offs and Limitations

For field length `L`, `r` ranges and `s` offers, parsing uses `O(L)` work and
matching uses `O(r * s)` comparisons; both sets are capped at 64. Quality is
an exact integer from zero through 1,000. A more-specific range determines an
offer's effective quality even when a broader range has a larger qvalue.
Equal effective qualities retain explicit server offer order.

This is a closed application policy, not complete HTTP content negotiation.
It rejects tabs, media parameters other than `q`, accept extensions, quoted
values, wildcard offers and non-ASCII spellings. It does not combine repeated
field lines, assign standardized meaning to client order, validate registered
media types, emit `Vary`, select encodings or languages, or manage caches.
Returning a representation still requires the surrounding HTTP response logic.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded ASCII Media Type Value](parse-a-bounded-ascii-media-type-value.md)
- [Parse and Rank a Bounded Accept-Language Value](parse-and-rank-a-bounded-accept-language-value.md)
- [Extract Bounded Singleton HTTP Headers with Explicit Rules](extract-bounded-singleton-http-headers-with-explicit-rules.md)
<!-- catalog:related:end -->

---
title: "Parse and Rank a Bounded Accept-Language Value"
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
  - parse-one-bounded-printable-ascii-http-1-field-section-with-exact-crlf-framing.md
  - extract-bounded-singleton-http-headers-with-explicit-rules.md
---

# Parse and Rank a Bounded Accept-Language Value

## Idea and Problem

Parse one HTTP Accept-Language field value into bounded, immutable preferences without using binary floating point for quality weights.

Each non-empty member contains an RFC 4647 basic language range and an
optional RFC 9110 qvalue. Converting its zero-to-three decimal digits to an
integer from 0 through 1,000 makes ordering exact. Ranges compare
case-insensitively and are normalized to lowercase.

HTTP list recipients ignore a reasonable number of empty elements, so this
profile caps raw comma slots separately from non-empty preferences. Equal
qualities retain source position only to make the returned representation
deterministic; that tie rule does not add a standardized HTTP preference.

## When to Use

Use this parser after extracting one bounded singleton field value when an
application needs to inspect or log a strict basic language-priority list.
Quality zero remains visible as an explicit “not acceptable” preference.

Use a standards-complete negotiation component when selecting among actual
language tags. Matching requires an explicit Basic Filtering or Lookup policy,
server fallback behavior, and cache handling beyond parsing the request field.

## Implementation

```python
import re
from dataclasses import dataclass

_MAX_FIELD_LENGTH = 8_192
_MAX_RAW_SLOTS = 128
_MAX_MEMBERS = 64
_BASIC_RANGE = r"(?:[A-Za-z]{1,8}(?:-[A-Za-z0-9]{1,8})*|\*)"
_QVALUE = r"(?:0(?:\.[0-9]{0,3})?|1(?:\.0{0,3})?)"
_MEMBER = re.compile(
    rf"(?P<range>{_BASIC_RANGE})"
    rf"(?:[ \t]*;[ \t]*(?i:q)=(?P<quality>{_QVALUE}))?",
    re.ASCII,
)


class AcceptLanguageParseError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class LanguagePreference:
    language_range: str
    quality: int
    source_position: int


def _quality_thousandths(raw_quality: str | None) -> int:
    if raw_quality is None:
        return 1_000
    whole, separator, fraction = raw_quality.partition(".")
    if whole == "1":
        return 1_000
    if not separator or not fraction:
        return 0
    return int((fraction + "000")[:3])


def parse_accept_language(text: str) -> tuple[LanguagePreference, ...]:
    """Return strict basic language ranges ranked by exact qvalue."""
    if type(text) is not str:
        raise TypeError("Accept-Language value must be an exact string")
    if len(text) > _MAX_FIELD_LENGTH:
        raise ValueError("Accept-Language value exceeds the supported length")
    if any(
        character != "\t" and not 0x20 <= ord(character) <= 0x7E
        for character in text
    ):
        raise ValueError("Accept-Language value must contain admitted ASCII")

    raw_slots = text.split(",")
    if len(raw_slots) > _MAX_RAW_SLOTS:
        raise ValueError("Accept-Language value has too many raw list slots")

    preferences: list[LanguagePreference] = []
    seen_ranges: set[str] = set()
    for source_position, raw_slot in enumerate(raw_slots):
        member = raw_slot.strip(" \t")
        if not member:
            continue
        if len(preferences) >= _MAX_MEMBERS:
            raise ValueError("Accept-Language value has too many members")
        match = _MEMBER.fullmatch(member)
        if match is None:
            raise AcceptLanguageParseError(
                "Accept-Language member is outside the supported grammar"
            )

        language_range = match.group("range").lower()
        if language_range in seen_ranges:
            raise AcceptLanguageParseError(
                "language ranges must be unique case-insensitively"
            )
        seen_ranges.add(language_range)
        preferences.append(
            LanguagePreference(
                language_range,
                _quality_thousandths(match.group("quality")),
                source_position,
            )
        )

    return tuple(
        sorted(
            preferences,
            key=lambda preference: (
                -preference.quality,
                preference.source_position,
            ),
        )
    )
```

## Example

```python
ranked = parse_accept_language("da, en-gb;q=0.8, en;q=0.7")
assert tuple(
    (item.language_range, item.quality, item.source_position)
    for item in ranked
) == (
    ("da", 1_000, 0),
    ("en-gb", 800, 1),
    ("en", 700, 2),
)

with_empty_slots = parse_accept_language(
    ", \t, EN;q=0.5,, fr;Q=0.500, *;q=0,"
)
assert tuple(item.language_range for item in with_empty_slots) == (
    "en",
    "fr",
    "*",
)
assert tuple(item.source_position for item in with_empty_slots) == (2, 4, 5)
assert parse_accept_language("") == ()

legal_qvalues = ("0", "0.", "0.1", "0.125", "1", "1.", "1.000")
assert tuple(
    parse_accept_language(f"de;q={qvalue}")[0].quality
    for qvalue in legal_qvalues
) == (0, 0, 100, 125, 1_000, 1_000, 1_000)

invalid_values = (
    "en;q=1.001",
    "en;q=0.1234",
    "en-*",
    "en;level=1",
    "en, EN;q=0",
    "en\r\nfr",
)
rejected = []
for value in invalid_values:
    try:
        parse_accept_language(value)
    except ValueError:
        rejected.append(value)

assert tuple(rejected) == invalid_values
```

## Trade-offs and Limitations

For field length `L` and `n` non-empty members, parsing uses `O(L)` work and
ranking uses `O(n log n)` comparisons, with `O(L + n)` temporary and returned
state. The 128-slot cap bounds empty-element tolerance independently of the
64-member cap.

The parser implements basic language-range syntax without consulting the IANA
Language Subtag Registry. It preserves qvalue zero and sorts equal qualities
by raw source slot for deterministic inspection, but recipients cannot assume
that equal-q list order carries portable negotiation priority.

This function does not combine repeated field lines, match ranges to available
representations, implement Basic Filtering or Lookup, choose a fallback, emit
`Vary`, or manage caches. Detailed language preferences can reveal private
information and increase fingerprinting surface; parsing them does not remove
that risk.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded ASCII Media Type Value](parse-a-bounded-ascii-media-type-value.md)
- [Parse One Bounded Printable-ASCII HTTP/1 Field Section with Exact CRLF Framing](parse-one-bounded-printable-ascii-http-1-field-section-with-exact-crlf-framing.md)
- [Extract Bounded Singleton HTTP Headers with Explicit Rules](extract-bounded-singleton-http-headers-with-explicit-rules.md)
<!-- catalog:related:end -->

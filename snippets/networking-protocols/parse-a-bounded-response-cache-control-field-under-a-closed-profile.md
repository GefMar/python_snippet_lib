---
title: "Parse a Bounded Response Cache-Control Field Under a Closed Profile"
snippet_type: recipe
use_cases:
  - networking
  - parsing
  - interoperability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-ascii-media-type-value.md
  - interpret-one-bounded-retry-after-value-under-a-closed-modern-sender-profile.md
  - apply-an-http-patch-only-when-a-strong-etag-still-matches.md
---

# Parse a Bounded Response Cache-Control Field Under a Closed Profile

## Idea and Problem

Parse one already-combined response Cache-Control field into an immutable closed-profile value instead of letting unknown or ambiguous directives silently influence application policy.

The profile recognizes six bare flags and two unquoted delta-second directives.
It normalizes directive names, rejects duplicates, and renders one deterministic
lowercase representation. Contradictory known flags remain visible because
parsing syntax is separate from deciding whether or how a response may be
cached.

## When to Use

Use this parser at a controlled HTTP boundary where both producer and consumer
have agreed on this deliberately small response-directive profile. It is useful
for fixtures, gateways, and internal protocol adapters that should fail closed
when an extension or qualified directive appears.

Use an HTTP cache implementation for general web traffic. Parsing this value
does not decide cacheability, freshness, validation, authorization handling,
or `Vary` behavior, and the closed profile is not a complete Cache-Control
grammar.

## Implementation

```python
import re
from dataclasses import dataclass

_MAX_FIELD_LENGTH = 4096
_MAX_DIRECTIVES = 16
_MAX_DELTA_SECONDS = (1 << 31) - 1
_FLAG_DIRECTIVES = frozenset(
    {
        "must-revalidate",
        "no-cache",
        "no-store",
        "private",
        "proxy-revalidate",
        "public",
    }
)
_SECONDS_DIRECTIVES = frozenset({"max-age", "s-maxage"})
_DECIMAL = re.compile(r"[0-9]+", re.ASCII)


class CacheControlParseError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class CacheControlProfile:
    flags: tuple[str, ...]
    seconds: tuple[tuple[str, int], ...]

    def render(self) -> str:
        directives = [*self.flags]
        directives.extend(
            f"{name}={value}" for name, value in self.seconds
        )
        return ", ".join(
            sorted(directives, key=lambda item: item.partition("=")[0])
        )


def parse_response_cache_control(text: str) -> CacheControlProfile:
    if type(text) is not str:
        raise TypeError("field value must be exact text")
    if not 1 <= len(text) <= _MAX_FIELD_LENGTH:
        raise ValueError("field value length is outside the supported range")
    if any(not 0x20 <= ord(character) <= 0x7E for character in text):
        raise ValueError("field value must contain printable ASCII only")

    items = text.split(",")
    if not 1 <= len(items) <= _MAX_DIRECTIVES:
        raise ValueError("directive count is outside the supported range")

    flags: set[str] = set()
    seconds: dict[str, int] = {}
    seen: set[str] = set()

    for raw_item in items:
        item = raw_item.strip(" ")
        if not item:
            raise CacheControlParseError("directives must not be empty")
        if item.count("=") > 1:
            raise CacheControlParseError("directive has too many value separators")

        if "=" in item:
            raw_name, raw_value = item.split("=", 1)
            name = raw_name.strip(" ").lower()
            value = raw_value.strip(" ")
            has_value = True
        else:
            name = item.lower()
            value = ""
            has_value = False

        if name in seen:
            raise CacheControlParseError("directive names must be unique")
        seen.add(name)

        if name in _FLAG_DIRECTIVES:
            if has_value:
                raise CacheControlParseError(
                    "flag directives must not have values"
                )
            flags.add(name)
        elif name in _SECONDS_DIRECTIVES:
            if not has_value or _DECIMAL.fullmatch(value) is None:
                raise CacheControlParseError(
                    "seconds directives need unquoted decimal values"
                )
            parsed = int(value)
            if parsed > _MAX_DELTA_SECONDS:
                raise CacheControlParseError(
                    "seconds value exceeds the supported range"
                )
            seconds[name] = parsed
        else:
            raise CacheControlParseError(
                "directive is outside the closed profile"
            )

    return CacheControlProfile(
        flags=tuple(sorted(flags)),
        seconds=tuple(sorted(seconds.items())),
    )
```

## Example

```python
profile = parse_response_cache_control(
    " PUBLIC, max-age = 60, No-Cache, s-maxage=0, private "
)

invalid_values = (
    "max-age=60, MAX-AGE=30",
    "no-store=true",
    'max-age="60"',
    "max-age=",
    "stale-if-error=30",
    "public,,max-age=60",
    "public\t",
)
rejected = []
for value in invalid_values:
    try:
        parse_response_cache_control(value)
    except ValueError:
        rejected.append(value)

assert (
    profile,
    profile.render(),
    tuple(rejected),
) == (
    CacheControlProfile(
        flags=("no-cache", "private", "public"),
        seconds=(("max-age", 60), ("s-maxage", 0)),
    ),
    "max-age=60, no-cache, private, public, s-maxage=0",
    invalid_values,
)
```

## Trade-offs and Limitations

Parsing is linear in the bounded field length; the immutable output contains at
most the fixed recognized directive set. ASCII spaces are accepted only around
the field, commas, and numeric equals signs. Tabs, quoted values, field-name
qualifiers, request-only directives, extension directives, and multiple field
representations that have not already been combined are rejected.

The result intentionally preserves combinations such as `public, private` or
`no-store, max-age=60`. Their operational meaning depends on the response,
request, cache type, validators, authorization state, and governing HTTP
semantics. This snippet performs no storage or serving decision and offers no
security boundary by itself.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded ASCII Media Type Value](parse-a-bounded-ascii-media-type-value.md)
- [Interpret One Bounded Retry-After Value Under a Closed Modern-Sender Profile](interpret-one-bounded-retry-after-value-under-a-closed-modern-sender-profile.md)
- [Apply an HTTP Patch Only When a Strong ETag Still Matches](apply-an-http-patch-only-when-a-strong-etag-still-matches.md)
<!-- catalog:related:end -->

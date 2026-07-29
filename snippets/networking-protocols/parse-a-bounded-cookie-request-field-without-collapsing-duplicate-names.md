---
title: "Parse a Bounded Cookie Request Field Without Collapsing Duplicate Names"
snippet_type: recipe
use_cases:
  - interoperability
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - extract-bounded-singleton-http-headers-with-explicit-rules.md
  - parse-a-bounded-percent-encoded-query-string-as-strict-utf-8-with-explicit-duplicate-rules.md
  - parse-one-bounded-printable-ascii-http-1-field-section-with-exact-crlf-framing.md
---

# Parse a Bounded Cookie Request Field Without Collapsing Duplicate Names

## Idea and Problem

Parse a strict HTTP Cookie request field into ordered name-value pairs without silently overwriting repeated names.

Cookies with one name can reach a server from different browser path or domain
scopes. Converting the field directly into a dictionary destroys that
ambiguity. This closed profile retains every pair, caps the complete field and
each component, and has one canonical `; ` rendering.

## When to Use

Use this recipe after extracting one bounded `Cookie` request field when a
service accepts unquoted RFC cookie-octet values and wants to preserve the
wire-level pairs for a separate policy decision. Treat names as
case-sensitive and values as opaque ASCII.

Use a maintained HTTP framework for browser-compatible parsing and cookie-jar
behavior. Reject or resolve repeated security-sensitive names under an
application policy that has enough request context; their source order alone
does not reveal which browser scope should win.

## Implementation

```python
import re
from dataclasses import dataclass

_MAX_FIELD_CHARACTERS = 4_096
_MAX_PAIRS = 64
_MAX_NAME_CHARACTERS = 64
_MAX_VALUE_CHARACTERS = 1_024
_COOKIE_NAME = re.compile(
    rf"[!#$%&'*+\-.^_`|~0-9A-Za-z]{{1,{_MAX_NAME_CHARACTERS}}}",
    re.ASCII,
).fullmatch
_COOKIE_VALUE = re.compile(
    rf"[\x21\x23-\x2B\x2D-\x3A\x3C-\x5B\x5D-\x7E]"
    rf"{{0,{_MAX_VALUE_CHARACTERS}}}",
    re.ASCII,
).fullmatch


class CookieFieldError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class CookiePair:
    name: str
    value: str


def _checked_pair(name: object, value: object) -> CookiePair:
    if type(name) is not str or type(value) is not str:
        raise TypeError("cookie names and values must be exact strings")
    if _COOKIE_NAME(name) is None:
        raise CookieFieldError("a cookie name is outside the closed grammar")
    if _COOKIE_VALUE(value) is None:
        raise CookieFieldError("a cookie value is outside the closed grammar")
    return CookiePair(name, value)


def parse_cookie_request_field(text: str) -> tuple[CookiePair, ...]:
    if type(text) is not str:
        raise TypeError("Cookie field value must be an exact string")
    if not 1 <= len(text) <= _MAX_FIELD_CHARACTERS:
        raise CookieFieldError("Cookie field length is outside the supported range")
    if any(not 0x20 <= ord(character) <= 0x7E for character in text):
        raise CookieFieldError("Cookie field must contain printable ASCII")

    raw_pairs = text.split("; ")
    if not 1 <= len(raw_pairs) <= _MAX_PAIRS:
        raise CookieFieldError("cookie pair count is outside the supported range")

    pairs: list[CookiePair] = []
    for raw_pair in raw_pairs:
        name, separator, value = raw_pair.partition("=")
        if not separator:
            raise CookieFieldError("every cookie pair must contain an equals sign")
        pairs.append(_checked_pair(name, value))
    return tuple(pairs)


def render_cookie_request_field(pairs: tuple[CookiePair, ...]) -> str:
    if type(pairs) is not tuple:
        raise TypeError("pairs must be an exact tuple")
    if not 1 <= len(pairs) <= _MAX_PAIRS:
        raise ValueError("cookie pair count is outside the supported range")

    checked: list[CookiePair] = []
    for pair in pairs:
        if type(pair) is not CookiePair:
            raise TypeError("pairs must contain exact CookiePair records")
        checked.append(_checked_pair(pair.name, pair.value))

    rendered = "; ".join(f"{pair.name}={pair.value}" for pair in checked)
    if len(rendered) > _MAX_FIELD_CHARACTERS:
        raise ValueError("rendered Cookie field exceeds the supported length")
    return rendered
```

## Example

```python
field = "theme=dark; id=first; empty=; id=second"
pairs = parse_cookie_request_field(field)
duplicate_values = tuple(pair.value for pair in pairs if pair.name == "id")

invalid_fields = (
    "theme=dark;id=compact",
    "theme=\"dark\"",
    "theme=dark, id=second",
    "theme=dark;  id=second",
)
rejected = []
for invalid in invalid_fields:
    try:
        parse_cookie_request_field(invalid)
    except CookieFieldError:
        rejected.append(invalid)

assert (
    duplicate_values,
    render_cookie_request_field(pairs),
    tuple(rejected),
) == (
    ("first", "second"),
    field,
    invalid_fields,
)
```

## Trade-offs and Limitations

Parsing and rendering use `O(L)` work and memory for a field of length `L`.
The immutable tuple preserves source order and duplicate names, but the frozen
records contain ordinary strings and do not confer trust on their values.
Errors describe only the violated rule and never include a cookie value.

This deliberately narrow grammar requires exactly one ASCII space after every
semicolon and rejects quoted values, escapes, commas, tabs, non-ASCII text and
other tolerant user-agent spellings. It parses neither `Set-Cookie` nor its
attributes, performs no percent decoding, and knows nothing about browser
storage, expiry, origin, path, domain, `Secure`, or `HttpOnly`. It does not
select a repeated name, authenticate a session, authorize a request, or make
cookies safe to log.

## Related Snippets

<!-- catalog:related:start -->
- [Extract Bounded Singleton HTTP Headers with Explicit Rules](extract-bounded-singleton-http-headers-with-explicit-rules.md)
- [Parse a Bounded Percent-Encoded Query String as Strict UTF-8 with Explicit Duplicate Rules](parse-a-bounded-percent-encoded-query-string-as-strict-utf-8-with-explicit-duplicate-rules.md)
- [Parse One Bounded Printable-ASCII HTTP/1 Field Section with Exact CRLF Framing](parse-one-bounded-printable-ascii-http-1-field-section-with-exact-crlf-framing.md)
<!-- catalog:related:end -->

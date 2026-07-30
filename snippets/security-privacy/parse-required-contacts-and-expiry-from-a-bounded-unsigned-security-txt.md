---
title: "Parse Required Contacts and Expiry from a Bounded Unsigned security.txt"
snippet_type: recipe
use_cases:
  - interoperability
  - parsing
  - security
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/parse-a-closed-rfc-3339-timestamp-subset-into-an-aware-datetime.md
  - resolve-a-bounded-relative-http-reference-under-a-same-origin-policy.md
  - verify-a-bounded-byte-stream-before-returning-its-payload.md
---

# Parse Required Contacts and Expiry from a Bounded Unsigned security.txt

## Idea and Problem

Extract the required contact and expiry evidence from one bounded unsigned security.txt document while preserving its ordered fields.

RFC 9116 requires at least one `Contact` field and exactly one `Expires` field.
This closed ASCII subset of UTF-8 validates complete LF or CRLF framing, field
syntax, component-aware URI grammar for six frozen core names, and a bounded
RFC 3339 timestamp subset. Field names are case-insensitive, while repeated
contact order remains significant.

## When to Use

Use this parser after a caller has retrieved one small unsigned `security.txt`
resource and needs deterministic core fields before presenting disclosure
options. It is suitable for inventory, validation, and diagnostics where a
closed accepted subset is preferable to permissive text splitting.

Use a standards-complete implementation when OpenPGP cleartext signatures,
full Net-Unicode, every RFC 3986 URI edge case, or complete BCP 47 language-tag
validation is required. Retrieval policy must separately enforce HTTPS,
redirect, origin, content-type, freshness, and trust decisions.

## Implementation

```python
import re
from dataclasses import dataclass
from datetime import UTC, datetime, timedelta, timezone
from string import ascii_letters, digits

_MAX_DOCUMENT_BYTES = 65_536
_MAX_LINE_BYTES = 2_048
_MAX_LINES = 256
_MAX_FIELDS = 128
_MAX_FIELD_NAME_CHARACTERS = 64
_MAX_CONTACTS = 16
_HEX_DIGITS = frozenset("0123456789abcdefABCDEF")
_UNRESERVED = frozenset(ascii_letters + digits + "-._~")
_SUB_DELIMITERS = frozenset("!$&'()*+,;=")
_PATH_CHARACTERS = _UNRESERVED | _SUB_DELIMITERS | frozenset(":@/")
_QUERY_FRAGMENT_CHARACTERS = _PATH_CHARACTERS | frozenset("?")
_URI_SCHEME = re.compile(r"[A-Za-z][A-Za-z0-9+.-]*", re.ASCII)
_URI_SCHEMES = frozenset({"https", "mailto", "tel", "dns", "openpgp4fpr"})
_HTTPS_REMAINDER = re.compile(
    r"//(?P<authority>[^/?#]+)(?P<path>/[^?#]*)?"
    r"(?:\?(?P<query>[^#]*))?(?:#(?P<fragment>.*))?",
    re.ASCII,
)
_DNS_LABEL = re.compile(
    r"[A-Za-z0-9](?:[A-Za-z0-9-]{0,61}[A-Za-z0-9])?",
    re.ASCII,
)
_RFC3339_EXPIRY = re.compile(
    r"(?P<year>[0-9]{4})-(?P<month>[0-9]{2})-(?P<day>[0-9]{2})"
    r"[Tt](?P<hour>[0-9]{2}):(?P<minute>[0-9]{2}):"
    r"(?P<second>[0-9]{2})(?:\.(?P<fraction>[0-9]{1,6}))?"
    r"(?P<zone>[Zz]|(?P<sign>[+-])(?P<offset_hour>[0-9]{2}):"
    r"(?P<offset_minute>[0-9]{2}))",
    re.ASCII,
)
_URI_FIELDS = frozenset(
    {
        "acknowledgments",
        "canonical",
        "contact",
        "encryption",
        "hiring",
        "policy",
    }
)


class SecurityTxtError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class SecurityTxtField:
    name: str
    value: str


@dataclass(frozen=True, slots=True)
class SecurityTxtCore:
    fields: tuple[SecurityTxtField, ...]
    contacts: tuple[str, ...]
    expires_at: datetime


def _malformed(message: str) -> SecurityTxtError:
    return SecurityTxtError(message)


def _validate_uri_component(
    value: str,
    *,
    allowed: frozenset[str],
    field_name: str,
) -> None:
    index = 0
    while index < len(value):
        character = value[index]
        if character == "%":
            if (
                index + 2 >= len(value)
                or value[index + 1] not in _HEX_DIGITS
                or value[index + 2] not in _HEX_DIGITS
            ):
                raise _malformed(f"{field_name} URI has a malformed percent escape")
            index += 3
            continue
        if character not in allowed:
            raise _malformed(f"{field_name} URI contains a disallowed character")
        index += 1


def _validate_https_remainder(remainder: str, *, field_name: str) -> None:
    match = _HTTPS_REMAINDER.fullmatch(remainder)
    if match is None:
        raise _malformed(f"{field_name} HTTPS URI has invalid component structure")

    authority = match.group("authority")
    if "@" in authority:
        raise _malformed(f"{field_name} HTTPS URI must not contain userinfo")
    if "[" in authority or "]" in authority or authority.count(":") > 1:
        raise _malformed(f"{field_name} HTTPS URI needs a DNS-style host")
    if ":" in authority:
        host, port_text = authority.rsplit(":", 1)
        if not port_text.isascii() or not port_text.isdecimal():
            raise _malformed(f"{field_name} HTTPS URI port must be decimal")
        if not 1 <= int(port_text) <= 65_535:
            raise _malformed(f"{field_name} HTTPS URI port is outside the supported range")
    else:
        host = authority

    if not 1 <= len(host) <= 253:
        raise _malformed(f"{field_name} HTTPS URI host length is invalid")
    if any(_DNS_LABEL.fullmatch(label) is None for label in host.split(".")):
        raise _malformed(f"{field_name} HTTPS URI host is outside the DNS-style profile")

    _validate_uri_component(
        match.group("path") or "",
        allowed=_PATH_CHARACTERS,
        field_name=field_name,
    )
    _validate_uri_component(
        match.group("query") or "",
        allowed=_QUERY_FRAGMENT_CHARACTERS,
        field_name=field_name,
    )
    _validate_uri_component(
        match.group("fragment") or "",
        allowed=_QUERY_FRAGMENT_CHARACTERS,
        field_name=field_name,
    )


def _validate_non_web_remainder(remainder: str, *, field_name: str) -> None:
    before_fragment, marker, fragment = remainder.partition("#")
    if marker and "#" in fragment:
        raise _malformed(f"{field_name} URI contains more than one fragment marker")
    path, query_marker, query = before_fragment.partition("?")
    if not path or path.startswith("/"):
        raise _malformed(f"{field_name} non-web URI needs a rootless path")
    _validate_uri_component(
        path,
        allowed=_PATH_CHARACTERS,
        field_name=field_name,
    )
    if query_marker:
        _validate_uri_component(
            query,
            allowed=_QUERY_FRAGMENT_CHARACTERS,
            field_name=field_name,
        )
    if marker:
        _validate_uri_component(
            fragment,
            allowed=_QUERY_FRAGMENT_CHARACTERS,
            field_name=field_name,
        )


def _validate_absolute_uri(value: str, *, field_name: str) -> None:
    if not value.isascii():
        raise _malformed(f"{field_name} URI must use ASCII")
    colon = value.find(":")
    if colon <= 0 or _URI_SCHEME.fullmatch(value[:colon]) is None:
        raise _malformed(f"{field_name} must contain an absolute URI")
    scheme = value[:colon].lower()
    if scheme not in _URI_SCHEMES:
        raise _malformed(f"{field_name} URI scheme is outside the closed profile")

    remainder = value[colon + 1 :]
    if scheme == "https":
        _validate_https_remainder(remainder, field_name=field_name)
    else:
        _validate_non_web_remainder(remainder, field_name=field_name)


def _parse_expiry(value: str) -> datetime:
    match = _RFC3339_EXPIRY.fullmatch(value)
    if match is None:
        raise _malformed("Expires does not match the supported RFC 3339 profile")

    year = int(match.group("year"))
    hour = int(match.group("hour"))
    minute = int(match.group("minute"))
    second = int(match.group("second"))
    if year == 0 or hour > 23 or minute > 59 or second > 59:
        raise _malformed("Expires contains an out-of-range date or time")

    fraction = match.group("fraction")
    microsecond = int(fraction.ljust(6, "0")) if fraction is not None else 0
    if match.group("zone") in {"Z", "z"}:
        zone = UTC
    else:
        offset_hour = int(match.group("offset_hour"))
        offset_minute = int(match.group("offset_minute"))
        if offset_hour > 23 or offset_minute > 59:
            raise _malformed("Expires contains an out-of-range UTC offset")
        sign = match.group("sign")
        if sign == "-" and offset_hour == 0 and offset_minute == 0:
            raise _malformed("Expires cannot use the unknown offset -00:00")
        offset = timedelta(hours=offset_hour, minutes=offset_minute)
        zone = timezone(-offset if sign == "-" else offset)

    try:
        parsed = datetime(
            year,
            int(match.group("month")),
            int(match.group("day")),
            hour,
            minute,
            second,
            microsecond,
            tzinfo=zone,
        )
    except ValueError:
        raise _malformed("Expires contains an invalid calendar date") from None
    try:
        return parsed.astimezone(UTC)
    except OverflowError:
        raise _malformed("Expires cannot be represented as a UTC datetime") from None


def parse_unsigned_security_txt(data: bytes) -> SecurityTxtCore:
    """Parse the required core of one unsigned RFC 9116 document subset."""
    if type(data) is not bytes:
        raise TypeError("data must be exact bytes")
    if not 1 <= len(data) <= _MAX_DOCUMENT_BYTES:
        raise ValueError("document size is outside the supported range")
    if not data.endswith(b"\n"):
        raise _malformed("every physical line must end with LF or CRLF")

    encoded_lines = data.split(b"\n")[:-1]
    if len(encoded_lines) > _MAX_LINES:
        raise _malformed("document contains too many physical lines")

    fields: list[SecurityTxtField] = []
    contacts: list[str] = []
    expires_values: list[str] = []
    preferred_language_count = 0

    for line_number, encoded_line in enumerate(encoded_lines, start=1):
        physical_size = len(encoded_line) + 1
        if physical_size > _MAX_LINE_BYTES:
            raise _malformed(f"line {line_number} exceeds the byte limit")
        if encoded_line.endswith(b"\r"):
            encoded_line = encoded_line[:-1]
        if b"\r" in encoded_line:
            raise _malformed(f"line {line_number} contains a bare CR")

        try:
            line = encoded_line.decode("ascii", errors="strict")
        except UnicodeDecodeError as error:
            raise _malformed(f"line {line_number} is outside the ASCII subset") from error
        if any(character != "\t" and not 0x20 <= ord(character) <= 0x7E for character in line):
            raise _malformed(f"line {line_number} contains a disallowed control")

        content = line.rstrip(" \t")
        if not content or content.startswith("#"):
            continue
        if content == "-----BEGIN PGP SIGNED MESSAGE-----":
            raise _malformed("OpenPGP cleartext signatures are outside this profile")

        colon = content.find(":")
        if colon <= 0 or content[colon : colon + 2] != ": ":
            raise _malformed(f"line {line_number} is not a field with ': '")
        name = content[:colon]
        value = content[colon + 2 :]
        if not 1 <= len(name) <= _MAX_FIELD_NAME_CHARACTERS:
            raise _malformed(f"line {line_number} field name length is invalid")
        if any(not 0x21 <= ord(character) <= 0x7E or character == ":" for character in name):
            raise _malformed(f"line {line_number} field name is invalid")
        if not value:
            raise _malformed(f"line {line_number} field value is empty")
        if len(fields) == _MAX_FIELDS:
            raise _malformed("document contains too many fields")

        normalized_name = name.lower()
        if normalized_name in _URI_FIELDS:
            _validate_absolute_uri(value, field_name=name)
        if normalized_name == "contact":
            if len(contacts) == _MAX_CONTACTS:
                raise _malformed("document contains too many Contact fields")
            contacts.append(value)
        elif normalized_name == "expires":
            expires_values.append(value)
        elif normalized_name == "preferred-languages":
            preferred_language_count += 1

        fields.append(SecurityTxtField(normalized_name, value))

    if not contacts:
        raise _malformed("document needs at least one Contact field")
    if len(expires_values) != 1:
        raise _malformed("document needs exactly one Expires field")
    if preferred_language_count > 1:
        raise _malformed("Preferred-Languages may appear at most once")

    return SecurityTxtCore(
        fields=tuple(fields),
        contacts=tuple(contacts),
        expires_at=_parse_expiry(expires_values[0]),
    )


```

## Example

```python
document = (
    b"# Preferred disclosure channels\r\n"
    b"Contact: mailto:security@example.test\n"
    b"cOnTaCt: https://example.test/report\r\n"
    b"Policy: https://example.test/policy \t\n"
    b"Expires: 2030-01-02T03:04:05.5+02:00\n"
    b"X-Note: coordinated disclosure\n"
)

parsed = parse_unsigned_security_txt(document)

invalid_documents = (
    document[:-1],
    b"Contact: http://example.test/report\nExpires: 2030-01-01T00:00:00Z\n",
    b"Contact: mailto:security%2@example.test\nExpires: 2030-01-01T00:00:00Z\n",
    b"Contact: mailto:security@example.test\n",
    b"-----BEGIN PGP SIGNED MESSAGE-----\r\n",
)
rejected = 0
for invalid in invalid_documents:
    try:
        parse_unsigned_security_txt(invalid)
    except SecurityTxtError:
        rejected += 1

assert (
    parsed.contacts,
    parsed.expires_at,
    parsed.fields[-1],
    rejected,
) == (
    (
        "mailto:security@example.test",
        "https://example.test/report",
    ),
    datetime(2030, 1, 2, 1, 4, 5, 500_000, tzinfo=UTC),
    SecurityTxtField("x-note", "coordinated disclosure"),
    5,
)
```

## Trade-offs and Limitations

Validation is `O(b)` time and memory for at most 64 KiB of input. The parser
retains at most 128 normalized field/value pairs and 16 ordered contacts.
Mixed LF and CRLF terminators are accepted because RFC 9116 permits either on
each line; a missing final terminator and every bare CR fail closed.

This is an explicit interoperable subset, not a claim of complete RFC 9116
conformance. ASCII is valid UTF-8 and already NFC, but the profile rejects the
Unicode comments that RFC 9116 can represent. It also rejects signed documents,
year zero, leap seconds, fractions beyond microseconds, and unknown offset
`-00:00`; endpoint values whose offset would cross Python's year 1-9999 UTC
range are also rejected. It preserves `Preferred-Languages` but enforces only
its single-occurrence rule, not full BCP 47 syntax.

HTTPS accepts a DNS-style host, an optional port from 1 through 65,535, and
component-appropriate RFC 3986 path, query, and fragment characters. It
deliberately excludes userinfo, bracketed IPv6 and IPvFuture literals,
internationalized host names, and registered-name sub-delimiters. Dotted
numeric label text is admitted without deciding whether it denotes an IPv4
address. The non-web schemes use a rootless path plus optional query and
fragment; their deeper scheme-specific semantics remain opaque after component
and percent-escape validation.

Parsing does not fetch or dereference a URI, verify a signature, compare a
`Canonical` field with the retrieval location, decide whether `expires_at` is
stale, authenticate the publisher, or grant permission to test any system.
Treat every field as untrusted display and routing data until independent
retrieval and trust policy succeeds.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Closed RFC 3339 Timestamp Subset into an Aware Datetime](../configuration-serialization/parse-a-closed-rfc-3339-timestamp-subset-into-an-aware-datetime.md)
- [Resolve a Bounded Relative HTTP Reference under a Same-Origin Policy](resolve-a-bounded-relative-http-reference-under-a-same-origin-policy.md)
- [Verify a Bounded Byte Stream Before Returning Its Payload](verify-a-bounded-byte-stream-before-returning-its-payload.md)
<!-- catalog:related:end -->

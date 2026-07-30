---
title: "Parse One Bounded RFC 5424 Syslog Message under a Closed Version-One Profile"
snippet_type: recipe
use_cases:
  - observability
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../networking-protocols/parse-one-bounded-proxy-protocol-version-one-line.md
  - ../networking-protocols/parse-one-bounded-printable-ascii-http-1-field-section-with-exact-crlf-framing.md
  - ../data-processing/split-quoted-and-bracketed-log-fields.md
---

# Parse One Bounded RFC 5424 Syslog Message under a Closed Version-One Profile

## Idea and Problem

Parse one already-isolated RFC 5424 message from exact immutable bytes while preserving the distinctions that a whitespace split would erase.

The header uses one canonical priority spelling, version `1`, and exact spaces.
Structured-data elements and their parameters remain ordered immutable tuples,
while duplicate identifiers fail instead of being overwritten. The optional
message is represented separately as absent, opaque bytes, or BOM-marked strict
UTF-8 text. A present but empty opaque message therefore differs from no
message at all.

## When to Use

Use this parser after a transport-specific layer has isolated one complete
message of 1 through 8,192 bytes. It fits bounded ingestion, protocol fixtures,
and diagnostics that deliberately require this exact version-one profile and
need typed header and structured-data values before applying local policy.

The parser enforces the RFC's uniqueness rule for structured-data identifiers.
Its closed profile is narrower by also requiring unique parameter names within
each element and by allowing a backslash in a parameter value to introduce
only `\"`, `\\`, or `\]`.
Choose a maintained syslog implementation when tolerant interoperability,
stream or datagram framing, RFC 3164 input, or relay behavior is required.

## Implementation

```python
import re
from dataclasses import dataclass
from datetime import UTC, datetime, timedelta, timezone
from enum import StrEnum

_MAX_MESSAGE_BYTES = 8_192
_MAX_STRUCTURED_DATA_ELEMENTS = 32
_MAX_PARAMETERS_PER_ELEMENT = 64
_UTF8_BOM = b"\xef\xbb\xbf"
_SD_NAME_FORBIDDEN = frozenset(b'=]"')
_PARAMETER_ESCAPES = frozenset(b'"\\]')
_TIMESTAMP = re.compile(
    rb"(?P<year>[0-9]{4})-(?P<month>[0-9]{2})-(?P<day>[0-9]{2})"
    rb"T(?P<hour>[0-9]{2}):(?P<minute>[0-9]{2}):(?P<second>[0-9]{2})"
    rb"(?:\.(?P<fraction>[0-9]{1,6}))?"
    rb"(?P<offset>Z|[+-][0-9]{2}:[0-9]{2})"
)


class SyslogMessageError(ValueError):
    """The bytes are outside the closed RFC 5424 version-one profile."""


class SyslogMessageKind(StrEnum):
    ABSENT = "absent"
    OPAQUE = "opaque"
    UTF8 = "utf8"


@dataclass(frozen=True, slots=True)
class SyslogMessageBody:
    kind: SyslogMessageKind
    value: bytes | str | None


@dataclass(frozen=True, slots=True)
class SyslogStructuredDataParameter:
    name: str
    value: str


@dataclass(frozen=True, slots=True)
class SyslogStructuredDataElement:
    sd_id: str
    parameters: tuple[SyslogStructuredDataParameter, ...]


@dataclass(frozen=True, slots=True)
class ParsedSyslogMessage:
    priority: int
    facility: int
    severity: int
    timestamp: datetime | None
    hostname: str | None
    app_name: str | None
    procid: str | None
    msgid: str | None
    structured_data: tuple[SyslogStructuredDataElement, ...]
    message: SyslogMessageBody


def _parse_priority(message: bytes) -> tuple[int, int]:
    if not message.startswith(b"<"):
        raise SyslogMessageError("PRI must start with '<'")
    closing = message.find(b">", 1)
    if closing < 0:
        raise SyslogMessageError("PRI is missing its closing '>'")

    digits = message[1:closing]
    if not 1 <= len(digits) <= 3 or not digits.isdigit():
        raise SyslogMessageError("PRI must contain one to three decimal digits")
    if len(digits) > 1 and digits.startswith(b"0"):
        raise SyslogMessageError("PRI must not contain a leading zero")
    priority = int(digits)
    if priority > 191:
        raise SyslogMessageError("PRI exceeds 191")
    return priority, closing + 1


def _read_header_token(
    message: bytes,
    position: int,
    *,
    field: str,
) -> tuple[bytes, int]:
    delimiter = message.find(b" ", position)
    if delimiter < 0:
        raise SyslogMessageError(f"{field} must be followed by one space")
    return message[position:delimiter], delimiter + 1


def _parse_header_value(token: bytes, *, field: str, maximum: int) -> str | None:
    if token == b"-":
        return None
    if not 1 <= len(token) <= maximum:
        raise SyslogMessageError(f"{field} length is outside the supported range")
    if any(octet < 0x21 or octet > 0x7E for octet in token):
        raise SyslogMessageError(f"{field} must contain only PRINTUSASCII bytes")
    return token.decode("ascii")


def _parse_timestamp(token: bytes) -> datetime | None:
    if token == b"-":
        return None
    match = _TIMESTAMP.fullmatch(token)
    if match is None:
        raise SyslogMessageError("TIMESTAMP has invalid version-one syntax")

    second = int(match["second"])
    if second > 59:
        raise SyslogMessageError("TIMESTAMP seconds must be between 00 and 59")
    fraction = match["fraction"]
    microsecond = int(fraction.ljust(6, b"0")) if fraction is not None else 0

    offset_token = match["offset"]
    if offset_token == b"Z":
        zone = UTC
    else:
        if offset_token == b"-00:00":
            raise SyslogMessageError("TIMESTAMP must use a known numeric offset")
        offset_hour = int(offset_token[1:3])
        offset_minute = int(offset_token[4:6])
        if offset_hour > 23 or offset_minute > 59:
            raise SyslogMessageError("TIMESTAMP offset is outside the supported range")
        delta = timedelta(hours=offset_hour, minutes=offset_minute)
        if offset_token.startswith(b"-"):
            delta = -delta
        zone = timezone(delta)

    try:
        local = datetime(
            int(match["year"]),
            int(match["month"]),
            int(match["day"]),
            int(match["hour"]),
            int(match["minute"]),
            second,
            microsecond,
            tzinfo=zone,
        )
    except ValueError as error:
        raise SyslogMessageError("TIMESTAMP contains an invalid calendar value") from error
    try:
        return local.astimezone(UTC)
    except (OverflowError, ValueError) as error:
        raise SyslogMessageError("TIMESTAMP overflows while normalizing to UTC") from error


def _is_sd_name_octet(octet: int) -> bool:
    return 0x21 <= octet <= 0x7E and octet not in _SD_NAME_FORBIDDEN


def _parse_sd_name(token: bytes, *, field: str) -> str:
    if not 1 <= len(token) <= 32:
        raise SyslogMessageError(f"{field} length is outside the supported range")
    if any(not _is_sd_name_octet(octet) for octet in token):
        raise SyslogMessageError(f"{field} contains a forbidden byte")
    return token.decode("ascii")


def _parse_parameter_value(message: bytes, position: int) -> tuple[str, int]:
    unescaped = bytearray()
    while position < len(message):
        octet = message[position]
        if octet == ord('"'):
            try:
                return unescaped.decode("utf-8", errors="strict"), position + 1
            except UnicodeDecodeError as error:
                raise SyslogMessageError(
                    "structured-data parameter value is not strict UTF-8"
                ) from error
        if octet == ord("\\"):
            position += 1
            if position == len(message):
                raise SyslogMessageError("structured-data escape is incomplete")
            escaped = message[position]
            if escaped not in _PARAMETER_ESCAPES:
                raise SyslogMessageError("structured-data escape is outside the closed profile")
            unescaped.append(escaped)
            position += 1
            continue
        if octet == ord("]"):
            raise SyslogMessageError("']' must be escaped inside a parameter value")
        unescaped.append(octet)
        position += 1
    raise SyslogMessageError("structured-data parameter value is missing its closing quote")


def _parse_structured_data_element(
    message: bytes,
    position: int,
) -> tuple[SyslogStructuredDataElement, int]:
    position += 1
    name_start = position
    while position < len(message) and message[position] not in (ord(" "), ord("]")):
        position += 1
    sd_id = _parse_sd_name(message[name_start:position], field="SD-ID")

    parameters: list[SyslogStructuredDataParameter] = []
    parameter_names: set[str] = set()
    while position < len(message) and message[position] == ord(" "):
        if len(parameters) >= _MAX_PARAMETERS_PER_ELEMENT:
            raise SyslogMessageError("structured-data element exceeds 64 parameters")
        position += 1
        parameter_start = position
        while position < len(message) and _is_sd_name_octet(message[position]):
            position += 1
        parameter_name = _parse_sd_name(
            message[parameter_start:position],
            field="PARAM-NAME",
        )
        if parameter_name in parameter_names:
            raise SyslogMessageError("PARAM-NAME values must be unique within an element")
        if position >= len(message) or message[position] != ord("="):
            raise SyslogMessageError("PARAM-NAME must be followed by '='")
        position += 1
        if position >= len(message) or message[position] != ord('"'):
            raise SyslogMessageError("parameter value must start with a quote")
        value, position = _parse_parameter_value(message, position + 1)
        parameter_names.add(parameter_name)
        parameters.append(SyslogStructuredDataParameter(parameter_name, value))

    if position >= len(message) or message[position] != ord("]"):
        raise SyslogMessageError("structured-data element is missing its closing ']'")
    return SyslogStructuredDataElement(sd_id, tuple(parameters)), position + 1


def _parse_structured_data(
    message: bytes,
    position: int,
) -> tuple[tuple[SyslogStructuredDataElement, ...], int]:
    if message[position] == ord("-"):
        return (), position + 1
    if message[position] != ord("["):
        raise SyslogMessageError("STRUCTURED-DATA must start with '-' or '['")

    elements: list[SyslogStructuredDataElement] = []
    sd_ids: set[str] = set()
    while position < len(message) and message[position] == ord("["):
        if len(elements) >= _MAX_STRUCTURED_DATA_ELEMENTS:
            raise SyslogMessageError("STRUCTURED-DATA exceeds 32 elements")
        element, position = _parse_structured_data_element(message, position)
        if element.sd_id in sd_ids:
            raise SyslogMessageError("SD-ID values must be unique")
        sd_ids.add(element.sd_id)
        elements.append(element)
    return tuple(elements), position


def _parse_message_body(message: bytes, position: int) -> SyslogMessageBody:
    if position == len(message):
        return SyslogMessageBody(SyslogMessageKind.ABSENT, None)
    if message[position] != ord(" "):
        raise SyslogMessageError("STRUCTURED-DATA must be followed by one space before MSG")

    payload = message[position + 1 :]
    if not payload.startswith(_UTF8_BOM):
        return SyslogMessageBody(SyslogMessageKind.OPAQUE, payload)
    try:
        text = payload[len(_UTF8_BOM) :].decode("utf-8", errors="strict")
    except UnicodeDecodeError as error:
        raise SyslogMessageError("BOM-prefixed MSG is not strict UTF-8") from error
    return SyslogMessageBody(SyslogMessageKind.UTF8, text)


def parse_rfc5424_message(message: bytes) -> ParsedSyslogMessage:
    """Parse one isolated message under the closed RFC 5424 profile."""
    if type(message) is not bytes:
        raise TypeError("message must be exact immutable bytes")
    if not 1 <= len(message) <= _MAX_MESSAGE_BYTES:
        raise SyslogMessageError("message length is outside the supported range")

    priority, position = _parse_priority(message)
    if message[position : position + 2] != b"1 ":
        raise SyslogMessageError("PRI must be followed by version 1 and one space")
    position += 2

    timestamp_token, position = _read_header_token(
        message,
        position,
        field="TIMESTAMP",
    )
    hostname_token, position = _read_header_token(
        message,
        position,
        field="HOSTNAME",
    )
    app_name_token, position = _read_header_token(
        message,
        position,
        field="APP-NAME",
    )
    procid_token, position = _read_header_token(
        message,
        position,
        field="PROCID",
    )
    msgid_token, position = _read_header_token(
        message,
        position,
        field="MSGID",
    )
    if position >= len(message):
        raise SyslogMessageError("STRUCTURED-DATA is missing")

    structured_data, position = _parse_structured_data(message, position)
    return ParsedSyslogMessage(
        priority=priority,
        facility=priority // 8,
        severity=priority % 8,
        timestamp=_parse_timestamp(timestamp_token),
        hostname=_parse_header_value(hostname_token, field="HOSTNAME", maximum=255),
        app_name=_parse_header_value(app_name_token, field="APP-NAME", maximum=48),
        procid=_parse_header_value(procid_token, field="PROCID", maximum=128),
        msgid=_parse_header_value(msgid_token, field="MSGID", maximum=32),
        structured_data=structured_data,
        message=_parse_message_body(message, position),
    )
```

## Example

```python


wire = (
    b"<165>1 2003-10-11T22:14:15.003Z mymachine.example.com evntslog - ID47 "
    b'[exampleSDID@32473 iut="3" eventSource="Application" eventID="1011" '
    b'detail="quote:\\" slash:\\\\ bracket:\\]"] '
    + _UTF8_BOM
    + b"An application event log entry..."
)
parsed = parse_rfc5424_message(wire)

no_message = parse_rfc5424_message(b"<13>1 - host app proc event -")
empty_opaque = parse_rfc5424_message(b"<13>1 - host app proc event - ")
opaque = parse_rfc5424_message(b"<13>1 - host app proc event - \xff")
offset_time = parse_rfc5424_message(
    b"<165>1 2003-08-24T05:14:15.000003-07:00 host app proc event -"
)


def is_rejected(candidate: object) -> bool:
    try:
        parse_rfc5424_message(candidate)  # type: ignore[arg-type]
    except TypeError:
        return True
    except SyslogMessageError:
        return True
    return False


prefix = b"<13>1 - host app proc event "
too_many_elements = prefix + b"".join(f"[id{index}]".encode("ascii") for index in range(33))
too_many_parameters = (
    prefix + b"[id" + b"".join(f' p{index}="x"'.encode("ascii") for index in range(65)) + b"]"
)
invalid = (
    b"",
    b"x" * (_MAX_MESSAGE_BYTES + 1),
    bytearray(b"<13>1 - host app proc event -"),
    b"<013>1 - host app proc event -",
    b"<13>2 - host app proc event -",
    b"<13>1 2003-10-11t22:14:15z host app proc event -",
    b"<13>1 2003-10-11T22:14:60Z host app proc event -",
    b"<13>1 2003-10-11T22:14:15.1234567Z host app proc event -",
    b"<13>1 2003-10-11T22:14:15-00:00 host app proc event -",
    b"<13>1 0001-01-01T00:00:00+23:59 host app proc event -",
    prefix + b'[id value="bad\\q"]',
    prefix + b'[id value="one" value="two"]',
    prefix + b'[id value="\xff"]',
    prefix + b"[id][id]",
    prefix + b"- " + _UTF8_BOM + b"\xff",
    too_many_elements,
    too_many_parameters,
)

assert parsed == ParsedSyslogMessage(
    priority=165,
    facility=20,
    severity=5,
    timestamp=datetime(2003, 10, 11, 22, 14, 15, 3_000, tzinfo=UTC),
    hostname="mymachine.example.com",
    app_name="evntslog",
    procid=None,
    msgid="ID47",
    structured_data=(
        SyslogStructuredDataElement(
            "exampleSDID@32473",
            (
                SyslogStructuredDataParameter("iut", "3"),
                SyslogStructuredDataParameter("eventSource", "Application"),
                SyslogStructuredDataParameter("eventID", "1011"),
                SyslogStructuredDataParameter(
                    "detail",
                    'quote:" slash:\\ bracket:]',
                ),
            ),
        ),
    ),
    message=SyslogMessageBody(
        SyslogMessageKind.UTF8,
        "An application event log entry...",
    ),
)
assert (
    no_message.message,
    empty_opaque.message,
    opaque.message,
    offset_time.timestamp,
) == (
    SyslogMessageBody(SyslogMessageKind.ABSENT, None),
    SyslogMessageBody(SyslogMessageKind.OPAQUE, b""),
    SyslogMessageBody(SyslogMessageKind.OPAQUE, b"\xff"),
    datetime(2003, 8, 24, 12, 14, 15, 3, tzinfo=UTC),
)
assert all(is_rejected(candidate) for candidate in invalid)
```

## Trade-offs and Limitations

Parsing takes linear time in at most 8,192 already-materialized bytes. The
result retains decoded header strings, decoded parameter values, opaque message
bytes, and at most 32 elements with 64 parameters each. Every equality case at
a declared byte or count limit is accepted. Timestamp fractions are padded to
microseconds, and numeric offsets are normalized to an aware UTC `datetime`;
normalization that would cross Python's supported year range fails.

The parser intentionally accepts only canonical PRI without leading zeroes,
version `1`, uppercase `T` and `Z`, seconds from `00` through `59`, at most six
fractional digits, and known numeric offsets other than `-00:00`. Header values
are syntax-checked PRINTUSASCII rather than interpreted as hostnames, process
identifiers, or registered message identifiers. Structured-data names are
case-sensitive; element and parameter order is preserved. RFC 5424 prohibits
repeated element identifiers but permits an `SD-PARAM` to occur multiple times
inside one element. This profile rejects those repeated parameter names and
also rejects unknown backslash sequences instead of applying the receiver
recovery specified by [RFC 5424](https://www.rfc-editor.org/rfc/rfc5424.html).

The function parses no transport framing, RFC 3164 form, truncated-message
repair, relay recovery, structured-data registry semantics, authentication,
authorization, storage, or retention policy. An opaque message may contain any
bytes, including controls and additional spaces. A BOM claims UTF-8 in this
profile, so invalid bytes after that marker reject the complete input.

## Related Snippets

<!-- catalog:related:start -->
- [Parse One Bounded PROXY Protocol Version 1 Line](../networking-protocols/parse-one-bounded-proxy-protocol-version-one-line.md)
- [Parse One Bounded Printable-ASCII HTTP/1 Field Section with Exact CRLF Framing](../networking-protocols/parse-one-bounded-printable-ascii-http-1-field-section-with-exact-crlf-framing.md)
- [Split Quoted and Bracketed Log Fields](../data-processing/split-quoted-and-bracketed-log-fields.md)
<!-- catalog:related:end -->

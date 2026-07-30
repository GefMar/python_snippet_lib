---
title: "Parse One Bounded RFC 9651 Structured Field Item"
snippet_type: recipe
use_cases:
  - interoperability
  - networking
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - verify-one-rfc-9530-sha-256-content-digest-under-a-closed-profile.md
  - parse-and-render-a-bounded-server-timing-field-under-a-closed-profile.md
  - parse-a-bounded-ascii-media-type-value.md
---

# Parse One Bounded RFC 9651 Structured Field Item

## Idea and Problem

Parse one bounded RFC 9651 Structured Field Item from exact ASCII bytes while preserving every bare-item type and the order of its parameters.

Structured Fields deliberately use strict parsing: accepting an almost-valid
spelling can make two HTTP implementations disagree about the same field. This
parser follows the Item algorithms, consumes the complete field value, and
returns immutable values. Tokens, Dates, and Display Strings have wrappers so
they cannot be confused with ordinary Strings or Integers.

## When to Use

Use this recipe after an HTTP layer has combined every field line for a field
whose definition selects the RFC 9651 Item type. The exact field value must be
between 1 and 65,536 ASCII bytes. Only leading and trailing space characters
are discarded; horizontal tabs are not accepted as Item padding.

The parser handles generic Structured Field syntax, not the meaning assigned
by a particular field. The caller must still enforce the field's allowed bare
types, recognized parameters, value ranges, and error policy. Keep Lists,
Dictionaries, field-line combination, and message acquisition outside this
function.

## Implementation

```python
from base64 import b64decode
from binascii import Error as Base64Error
from dataclasses import dataclass
from decimal import Decimal

_MAX_FIELD_BYTES = 65_536
_MAX_PARAMETERS = 256
_MAX_PARAMETER_KEY_CHARACTERS = 64
_MAX_STRING_CHARACTERS = 1_024
_MAX_TOKEN_CHARACTERS = 512
_MAX_BYTE_SEQUENCE_BYTES = 16_384
_MAX_DISPLAY_STRING_CHARACTERS = 1_024
_MAX_DISPLAY_STRING_UTF8_BYTES = 4 * _MAX_DISPLAY_STRING_CHARACTERS

_ASCII_ALPHA = frozenset("ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz")
_ASCII_DIGITS = frozenset("0123456789")
_KEY_FIRST = frozenset("abcdefghijklmnopqrstuvwxyz*")
_KEY_REST = frozenset("abcdefghijklmnopqrstuvwxyz0123456789_-.*")
_TOKEN_REST = frozenset(
    "!#$%&'*+-.^_`|~:/0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
)
_BASE64_CHARACTERS = frozenset("ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/=")
_LOWER_HEXADECIMAL = frozenset("0123456789abcdef")


class StructuredFieldItemError(ValueError):
    """The bytes are not one bounded RFC 9651 Structured Field Item."""


@dataclass(frozen=True, slots=True)
class StructuredFieldToken:
    text: str


@dataclass(frozen=True, slots=True)
class StructuredFieldDate:
    seconds: int


@dataclass(frozen=True, slots=True)
class StructuredFieldDisplayString:
    text: str


type StructuredFieldBareItem = (
    int
    | Decimal
    | str
    | StructuredFieldToken
    | bytes
    | bool
    | StructuredFieldDate
    | StructuredFieldDisplayString
)


@dataclass(frozen=True, slots=True)
class StructuredFieldItem:
    value: StructuredFieldBareItem
    parameters: tuple[tuple[str, StructuredFieldBareItem], ...]


class _ItemParser:
    def __init__(self, text: str) -> None:
        self.text = text
        self.position = 0

    def _peek(self) -> str | None:
        if self.position == len(self.text):
            return None
        return self.text[self.position]

    def skip_spaces(self) -> None:
        while self._peek() == " ":
            self.position += 1

    def parse_item(self) -> StructuredFieldItem:
        value = self.parse_bare_item()
        parameters = self._parse_parameters()
        return StructuredFieldItem(value, parameters)

    def parse_bare_item(self) -> StructuredFieldBareItem:
        first = self._peek()
        if first is None:
            raise StructuredFieldItemError("expected a bare Item")
        if first == "-" or first in _ASCII_DIGITS:
            return self._parse_number()
        if first == '"':
            return self._parse_string()
        if first in _ASCII_ALPHA or first == "*":
            return self._parse_token()
        if first == ":":
            return self._parse_byte_sequence()
        if first == "?":
            return self._parse_boolean()
        if first == "@":
            return self._parse_date()
        if first == "%":
            return self._parse_display_string()
        raise StructuredFieldItemError("bare Item type is not recognized")

    def _parse_parameters(
        self,
    ) -> tuple[tuple[str, StructuredFieldBareItem], ...]:
        parameters: dict[str, StructuredFieldBareItem] = {}
        parameter_count = 0
        while self._peek() == ";":
            parameter_count += 1
            if parameter_count > _MAX_PARAMETERS:
                raise StructuredFieldItemError("parameter count exceeds 256")
            self.position += 1
            self.skip_spaces()
            key = self._parse_key()
            value: StructuredFieldBareItem = True
            if self._peek() == "=":
                self.position += 1
                value = self.parse_bare_item()
            parameters[key] = value
        return tuple(parameters.items())

    def _parse_key(self) -> str:
        first = self._peek()
        if first is None or first not in _KEY_FIRST:
            raise StructuredFieldItemError("parameter key has an invalid first character")
        start = self.position
        self.position += 1
        while (character := self._peek()) is not None and character in _KEY_REST:
            self.position += 1
        key = self.text[start : self.position]
        if len(key) > _MAX_PARAMETER_KEY_CHARACTERS:
            raise StructuredFieldItemError("parameter key exceeds 64 characters")
        return key

    def _parse_number(self) -> int | Decimal:
        start = self.position
        if self._peek() == "-":
            self.position += 1

        integral_start = self.position
        while (character := self._peek()) is not None and character in _ASCII_DIGITS:
            self.position += 1
        integral_digits = self.position - integral_start
        if integral_digits == 0:
            raise StructuredFieldItemError("number requires an integer component")

        if self._peek() != ".":
            if integral_digits > 15:
                raise StructuredFieldItemError("Integer exceeds 15 digits")
            return int(self.text[start : self.position])

        if integral_digits > 12:
            raise StructuredFieldItemError("Decimal integer component exceeds 12 digits")
        self.position += 1
        fractional_start = self.position
        while (character := self._peek()) is not None and character in _ASCII_DIGITS:
            self.position += 1
        fractional_digits = self.position - fractional_start
        if not 1 <= fractional_digits <= 3:
            raise StructuredFieldItemError(
                "Decimal requires between one and three fractional digits"
            )
        return Decimal(self.text[start : self.position])

    def _parse_string(self) -> str:
        self.position += 1
        output: list[str] = []
        while (character := self._peek()) is not None:
            self.position += 1
            if character == '"':
                return "".join(output)
            if character == "\\":
                escaped = self._peek()
                if escaped not in ('"', "\\"):
                    raise StructuredFieldItemError("String escape is invalid")
                self.position += 1
                character = escaped
            elif not 0x20 <= ord(character) <= 0x7E:
                raise StructuredFieldItemError("String contains a non-printable character")
            output.append(character)
            if len(output) > _MAX_STRING_CHARACTERS:
                raise StructuredFieldItemError("String exceeds 1,024 characters")
        raise StructuredFieldItemError("String is missing its closing quote")

    def _parse_token(self) -> StructuredFieldToken:
        start = self.position
        self.position += 1
        while (character := self._peek()) is not None and character in _TOKEN_REST:
            self.position += 1
        text = self.text[start : self.position]
        if len(text) > _MAX_TOKEN_CHARACTERS:
            raise StructuredFieldItemError("Token exceeds 512 characters")
        return StructuredFieldToken(text)

    def _parse_byte_sequence(self) -> bytes:
        self.position += 1
        end = self.text.find(":", self.position)
        if end < 0:
            raise StructuredFieldItemError("Byte Sequence is missing its closing colon")
        encoded = self.text[self.position : end]
        self.position = end + 1
        if any(character not in _BASE64_CHARACTERS for character in encoded):
            raise StructuredFieldItemError("Byte Sequence contains a non-Base64 character")

        padded = encoded + "=" * (-len(encoded) % 4)
        try:
            decoded = b64decode(padded, validate=True)
        except Base64Error as error:
            raise StructuredFieldItemError("Byte Sequence is not valid Base64") from error
        if len(decoded) > _MAX_BYTE_SEQUENCE_BYTES:
            raise StructuredFieldItemError("Byte Sequence exceeds 16,384 decoded bytes")
        return decoded

    def _parse_boolean(self) -> bool:
        self.position += 1
        value = self._peek()
        if value not in ("0", "1"):
            raise StructuredFieldItemError("Boolean must be ?0 or ?1")
        self.position += 1
        return value == "1"

    def _parse_date(self) -> StructuredFieldDate:
        self.position += 1
        value = self._parse_number()
        if type(value) is not int:
            raise StructuredFieldItemError("Date must contain integer seconds")
        return StructuredFieldDate(value)

    def _parse_display_string(self) -> StructuredFieldDisplayString:
        if not self.text.startswith('%"', self.position):
            raise StructuredFieldItemError("Display String must start with percent and quote")
        self.position += 2
        encoded = bytearray()

        while (character := self._peek()) is not None:
            self.position += 1
            if not 0x20 <= ord(character) <= 0x7E:
                raise StructuredFieldItemError(
                    "Display String contains a non-printable wire character"
                )
            if character == "%":
                if self.position + 2 > len(self.text):
                    raise StructuredFieldItemError("Display String escape is truncated")
                hexadecimal = self.text[self.position : self.position + 2]
                if any(digit not in _LOWER_HEXADECIMAL for digit in hexadecimal):
                    raise StructuredFieldItemError(
                        "Display String escape must use lowercase hexadecimal"
                    )
                encoded.append(int(hexadecimal, 16))
                self.position += 2
            elif character == '"':
                try:
                    text = bytes(encoded).decode("utf-8")
                except UnicodeDecodeError as error:
                    raise StructuredFieldItemError("Display String is not valid UTF-8") from error
                if len(text) > _MAX_DISPLAY_STRING_CHARACTERS:
                    raise StructuredFieldItemError("Display String exceeds 1,024 characters")
                return StructuredFieldDisplayString(text)
            else:
                encoded.append(ord(character))

            if len(encoded) > _MAX_DISPLAY_STRING_UTF8_BYTES:
                raise StructuredFieldItemError(
                    "Display String exceeds the decoded UTF-8 byte limit"
                )

        raise StructuredFieldItemError("Display String is missing its closing quote")


def parse_structured_field_item(field_value: bytes) -> StructuredFieldItem:
    """Parse one complete bounded RFC 9651 Item field value."""
    if type(field_value) is not bytes:
        raise TypeError("field_value must be exact bytes")
    if not 1 <= len(field_value) <= _MAX_FIELD_BYTES:
        raise StructuredFieldItemError("field length is outside 1..65,536 bytes")
    try:
        text = field_value.decode("ascii")
    except UnicodeDecodeError as error:
        raise StructuredFieldItemError("field_value must contain only ASCII bytes") from error

    parser = _ItemParser(text)
    parser.skip_spaces()
    item = parser.parse_item()
    parser.skip_spaces()
    if parser.position != len(text):
        raise StructuredFieldItemError("unconsumed bytes follow the Item")
    return item
```

## Example

```python


values = tuple(
    parse_structured_field_item(field_value).value
    for field_value in (
        b"42",
        b"-12.340",
        b'"quoted \\"text\\""',
        b"token/one",
        b":AQI:",
        b"?0",
        b"@1659578233",
        b'%"This is for %c3%bcsers"',
    )
)

parameterized = parse_structured_field_item(b'example;a=1;flag;a=%"last"')
nonzero_pad_bits = parse_structured_field_item(b":iZ==:")

malformed_values = (
    b"",
    b"\t42",
    b"42\t",
    b"42, 43",
    b"1234567890123.0",
    b'"bad\\n-escape"',
    b":base64-_:",
    b'%"uppercase-%C3%BC"',
    b'%"invalid-%ff"',
    b"?1extra",
)
rejected = 0
for malformed_value in malformed_values:
    try:
        parse_structured_field_item(malformed_value)
    except StructuredFieldItemError:
        rejected += 1

assert values == (
    42,
    Decimal("-12.340"),
    'quoted "text"',
    StructuredFieldToken("token/one"),
    b"\x01\x02",
    False,
    StructuredFieldDate(1_659_578_233),
    StructuredFieldDisplayString("This is for üsers"),
)
assert parameterized == StructuredFieldItem(
    StructuredFieldToken("example"),
    (
        ("a", StructuredFieldDisplayString("last")),
        ("flag", True),
    ),
)
assert nonzero_pad_bits.value == b"\x89"
assert rejected == len(malformed_values)
```

## Trade-offs and Limitations

Parsing takes `O(n)` time in the 65,536-byte input bound. The immutable result
retains the decoded Item value, at most 256 parameter entries, and their
decoded values. Strings are capped at 1,024 characters, Tokens at 512,
Display Strings at 1,024 Unicode scalar values and 4,096 UTF-8 bytes, and Byte
Sequences at 16,384 decoded bytes. These caps meet RFC 9651's stated minimums
for Strings, Tokens, Byte Sequences, parameter count, and parameter-key length.

Base64 syntax and alphabet placement are strict, but missing trailing `=`
padding is synthesized and nonzero pad bits are accepted as RFC 9651 advises
for recipients. Integer and Date syntax is limited to 15 digits. Decimals use
`Decimal`, so their value is not rounded through binary floating point; wire
distinctions such as trailing zeroes are not a serialization promise.

The parser follows the [RFC 9651 Item parsing algorithms](https://www.rfc-editor.org/rfc/rfc9651.html),
but it does not combine repeated HTTP field lines, parse Lists or Dictionaries,
serialize values, or enforce any field-specific meaning. Display Strings are
decoded without normalization, language detection, escaping, or security
filtering; callers must make untrusted Unicode safe for its eventual display
context.

## Related Snippets

<!-- catalog:related:start -->
- [Verify One RFC 9530 SHA-256 Content-Digest Under a Closed Profile](verify-one-rfc-9530-sha-256-content-digest-under-a-closed-profile.md)
- [Parse and Render a Bounded Server-Timing Field Under a Closed Profile](parse-and-render-a-bounded-server-timing-field-under-a-closed-profile.md)
- [Parse a Bounded ASCII Media Type Value](parse-a-bounded-ascii-media-type-value.md)
<!-- catalog:related:end -->

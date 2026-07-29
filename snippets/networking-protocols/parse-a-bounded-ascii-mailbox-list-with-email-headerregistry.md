---
title: "Parse a Bounded ASCII Mailbox List with email.headerregistry"
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
  - extract-bounded-singleton-http-headers-with-explicit-rules.md
  - parse-a-bounded-ascii-media-type-value.md
  - interpret-one-bounded-retry-after-value-under-a-closed-modern-sender-profile.md
---

# Parse a Bounded ASCII Mailbox List with email.headerregistry

## Idea and Problem

Project one bounded, unfolded address-header value into immutable mailbox components while rejecting syntax defects and address groups outside a closed profile.

`HeaderRegistry` supplies the structured mailbox grammar, including quoted
display names and encoded words. The wrapper inspects defects before using the
flattened address view, rejects named groups explicitly, and returns only the
display name, username, and domain needed by the caller.

## When to Use

Use this after isolating one trusted-size `To`-style field value when an
application needs an ordered mailbox list rather than ad hoc comma splitting.
The input profile is printable ASCII, but RFC encoded words may produce a
Unicode display name. Address specifications remain ASCII in this recipe.

Use a dedicated mail system when SMTPUTF8 addresses, policy-dependent header
handling, delivery, DNS, internationalized domains, or complete RFC behavior
is required. Do not use a successfully parsed address as authentication,
authorization, ownership proof, or evidence that delivery will succeed.

## Implementation

```python
from dataclasses import dataclass
from email.headerregistry import HeaderRegistry

_MAX_HEADER_BYTES = 4_096
_MAX_MAILBOXES = 32
_MAX_DISPLAY_NAME_BYTES = 256
_MAX_USERNAME_BYTES = 128
_MAX_DOMAIN_BYTES = 255
_MAX_PROJECTED_BYTES = 8_192


class MailboxListError(ValueError):
    """Raised when a field is outside the closed mailbox-list profile."""


@dataclass(frozen=True, slots=True)
class Mailbox:
    display_name: str
    username: str
    domain: str


def parse_mailbox_list(value: str) -> tuple[Mailbox, ...]:
    """Return ordered mailbox components from one unfolded ASCII field."""
    if type(value) is not str:
        raise TypeError("value must be an exact string")
    try:
        raw_value = value.encode("ascii")
    except UnicodeEncodeError:
        raise MailboxListError(
            "value must contain only ASCII characters"
        ) from None
    if not 1 <= len(raw_value) <= _MAX_HEADER_BYTES:
        raise MailboxListError("value size is outside the supported range")
    if any(byte < 0x20 or byte > 0x7E for byte in raw_value):
        raise MailboxListError(
            "value must be one unfolded printable-ASCII field"
        )

    try:
        header = HeaderRegistry()("To", value)
        defects = header.defects
        groups = header.groups
        addresses = header.addresses
    except (AttributeError, IndexError, RecursionError, ValueError):
        raise MailboxListError(
            "value is not a supported mailbox list"
        ) from None

    if defects:
        raise MailboxListError("value contains a mailbox syntax defect")
    if any(group.display_name is not None for group in groups):
        raise MailboxListError(
            "named address groups are outside the supported profile"
        )
    if not 1 <= len(addresses) <= _MAX_MAILBOXES:
        raise MailboxListError(
            "mailbox count is outside the supported range"
        )

    projected: list[Mailbox] = []
    total_projected_bytes = 0
    for address in addresses:
        if not address.username or not address.domain:
            raise MailboxListError(
                "each mailbox must have a username and domain"
            )
        if not address.username.isascii() or not address.domain.isascii():
            raise MailboxListError("address specifications must be ASCII")

        fields = (
            (address.display_name, _MAX_DISPLAY_NAME_BYTES),
            (address.username, _MAX_USERNAME_BYTES),
            (address.domain, _MAX_DOMAIN_BYTES),
        )
        for field, byte_limit in fields:
            try:
                field_bytes = len(field.encode("utf-8"))
            except UnicodeEncodeError:
                raise MailboxListError(
                    "projected fields must be UTF-8 encodable"
                ) from None
            if field_bytes > byte_limit:
                raise MailboxListError(
                    "a projected mailbox field exceeds its byte limit"
                )
            total_projected_bytes += field_bytes
            if total_projected_bytes > _MAX_PROJECTED_BYTES:
                raise MailboxListError(
                    "projected mailboxes exceed the aggregate byte limit"
                )

        projected.append(
            Mailbox(
                display_name=address.display_name,
                username=address.username,
                domain=address.domain,
            )
        )

    return tuple(projected)
```

## Example

```python
mailboxes = parse_mailbox_list(
    '"Doe, Jane" <jane@example.com>, ops@example.net'
)
encoded_display_name = parse_mailbox_list(
    "=?utf-8?q?Jos=C3=A9?= <jose@example.com>"
)
duplicates = parse_mailbox_list(
    "ops@example.net, ops@example.net"
)

rejected = 0
for invalid in (
    "Team: one@example.com, two@example.com;",
    "bad@@example.com",
    "postmaster",
    "one@example.com\r\nBcc: hidden@example.net",
    "josé@example.com",
):
    try:
        parse_mailbox_list(invalid)
    except MailboxListError:
        rejected += 1

assert (
    mailboxes
    == (
        Mailbox("Doe, Jane", "jane", "example.com"),
        Mailbox("", "ops", "example.net"),
    )
    and encoded_display_name
    == (Mailbox("José", "jose", "example.com"),)
    and duplicates
    == (
        Mailbox("", "ops", "example.net"),
        Mailbox("", "ops", "example.net"),
    )
    and rejected == 5
)
```

## Trade-offs and Limitations

Parsing and projection are bounded by the 4,096-byte field and 32-mailbox
limits. Projected storage is linear in the accepted component bytes. The
parser may allocate internal syntax objects before the wrapper projects the
result, so these limits reduce exposure but do not turn `HeaderRegistry` into
a hardened hostile-input parser.

The email package is intentionally tolerant. This recipe rejects every defect
it reports and normalizes several malformed-input exceptions, but it does not
claim complete RFC rejection. `addresses` also flattens groups, which is why
the group view is checked first. Comments and quoting may be normalized during
projection, while order and duplicate mailboxes are preserved.

Component byte limits are application profile limits, not assertions about
universal mailbox validity. No network lookup, delivery attempt, canonical
identity comparison, logging, persistence, authentication, or authorization
occurs.

## Related Snippets

<!-- catalog:related:start -->
- [Extract Bounded Singleton HTTP Headers with Explicit Rules](extract-bounded-singleton-http-headers-with-explicit-rules.md)
- [Parse a Bounded ASCII Media Type Value](parse-a-bounded-ascii-media-type-value.md)
- [Interpret One Bounded Retry-After Value Under a Closed Modern-Sender Profile](interpret-one-bounded-retry-after-value-under-a-closed-modern-sender-profile.md)
<!-- catalog:related:end -->

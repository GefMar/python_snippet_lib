---
title: "Select Preferred Inline Text from a MIME Message"
snippet_type: recipe
use_cases:
  - parsing
  - interoperability
tested_python:
  - "3.14"
dependencies: []
related:
  - ../networking-protocols/parse-a-bounded-ascii-mailbox-list-with-email-headerregistry.md
---

# Select Preferred Inline Text from a MIME Message

## Idea and Problem

Select an inline text body from a complete MIME message without treating a text attachment as the body.

Ask the email package to choose the body with plain text preferred over HTML. The MIME tree and content disposition matter: finding the first text part in a generic walk does not express the same rule.

## When to Use

Use this for trusted, size-limited email bytes when a reader can handle either a plain body or an HTML fallback. Preserve the returned content type so the caller knows how to handle the text.

## Implementation

```python
from email import policy
from email.parser import BytesParser


def preferred_inline_text(raw: bytes) -> tuple[str, str] | None:
    message = BytesParser(policy=policy.default).parsebytes(raw)
    body = message.get_body(preferencelist=("plain", "html"))
    if body is None:
        return None
    return body.get_content_type(), body.get_content()
```

## Example

```python
from email.message import EmailMessage

message = EmailMessage()
message.set_content("plain body")
message.add_alternative("<p>HTML body</p>", subtype="html")
message.add_attachment("text attachment", filename="notes.txt")

html_only = EmailMessage()
html_only.set_content("<p>fallback</p>", subtype="html")
attachment_only = EmailMessage()
attachment_only.set_content("not an inline body", disposition="attachment")

assert (
    preferred_inline_text(message.as_bytes()),
    preferred_inline_text(html_only.as_bytes()),
    preferred_inline_text(attachment_only.as_bytes()),
) == (("text/plain", "plain body\n"), ("text/html", "<p>fallback</p>\n"), None)
```

## Trade-offs and Limitations

This parses the entire object and does not validate arbitrary mail, authenticate its sender, or sanitize HTML. Body selection follows the standard library's MIME rules, including fallback for missing content type. Set byte and message depth limits as appropriate before accepting untrusted input.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded ASCII Mailbox List with email.headerregistry](../networking-protocols/parse-a-bounded-ascii-mailbox-list-with-email-headerregistry.md)
<!-- catalog:related:end -->

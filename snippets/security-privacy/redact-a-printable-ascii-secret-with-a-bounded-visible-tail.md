---
title: "Redact a Printable ASCII Secret with a Bounded Visible Tail"
snippet_type: recipe
use_cases:
  - observability
  - security
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../observability-operations/format-log-records-as-json-with-explicit-extra-fields.md
  - ../observability-operations/scope-structured-log-fields-with-context-variables.md
---

# Redact a Printable ASCII Secret with a Bounded Visible Tail

## Idea and Problem

Mask every character of a short secret and reveal only a bounded suffix of longer values while keeping at least eight characters hidden.

The function accepts a deliberately narrow printable-ASCII boundary, keeps the
output length equal to the input length, and uses a fixed mask character.
Generic validation errors never interpolate the sensitive value.

## When to Use

Use this helper only after a threat review has approved suffix display for one
specific credential format and audience, such as distinguishing manually
rotated values in a restricted diagnostic view. Keep the unredacted value out
of broad structured metadata before formatting. Use opaque IDs or keyed
fingerprints when even a suffix or the original length would reveal too much.

## Implementation

```python
_MAX_SECRET_LENGTH = 256
_MINIMUM_HIDDEN = 8
_MAX_VISIBLE_TAIL = 8


def redact_secret_tail(secret: str, *, visible_tail: int = 4) -> str:
    if not isinstance(secret, str):
        raise TypeError("secret must be text")
    if not 1 <= len(secret) <= _MAX_SECRET_LENGTH:
        raise ValueError("secret length is outside the accepted range")
    if any(not 32 <= ord(character) <= 126 for character in secret):
        raise ValueError("secret must contain printable ASCII only")
    if isinstance(visible_tail, bool) or not isinstance(visible_tail, int):
        raise TypeError("visible_tail must be an integer")
    if not 0 <= visible_tail <= _MAX_VISIBLE_TAIL:
        raise ValueError("visible_tail is outside the accepted range")

    visible_count = min(
        visible_tail,
        max(0, len(secret) - _MINIMUM_HIDDEN),
    )
    hidden_count = len(secret) - visible_count
    return "*" * hidden_count + secret[hidden_count:]
```

## Example

```python
sample_value = "violet?harbor#731"
redacted = redact_secret_tail(sample_value)
short_value = redact_secret_tail("a-b?c")
boundary_value = redact_secret_tail("abcdefghi", visible_tail=4)
fully_hidden = redact_secret_tail(sample_value, visible_tail=0)


def is_rejected(value: str) -> bool:
    try:
        redact_secret_tail(value)
    except ValueError:
        return True
    return False


assert (
    redacted,
    short_value,
    boundary_value,
    fully_hidden,
    len(redacted) == len(sample_value),
    sample_value[:-4] not in redacted,
    is_rejected("line\nbreak"),
    is_rejected("caf\u00e9-value"),
) == (
    "*************#731",
    "*****",
    "********i",
    "*****************",
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

The output leaks the input length and, for values longer than eight characters,
the requested suffix; both can act as fingerprints. Redaction does not encrypt,
erase, authenticate, or make a value safe for unrestricted logging, and the
original string remains in memory. The ASCII policy excludes Unicode secrets,
while the fixed asterisk mask can be visually ambiguous when the original also
contains asterisks. This helper neither walks nested metadata nor removes other
sensitive fields: apply an allowlist at the data boundary and keep access to
diagnostic output restricted.

## Related Snippets

<!-- catalog:related:start -->
- [Format Log Records as JSON with Explicit Extra Fields](../observability-operations/format-log-records-as-json-with-explicit-extra-fields.md)
- [Scope Structured Log Fields with Context Variables](../observability-operations/scope-structured-log-fields-with-context-variables.md)
<!-- catalog:related:end -->

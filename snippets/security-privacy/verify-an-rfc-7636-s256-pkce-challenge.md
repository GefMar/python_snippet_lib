---
title: "Verify an RFC 7636 S256 PKCE Challenge"
snippet_type: recipe
use_cases:
  - security
  - validation
  - interoperability
tested_python:
  - "3.14"
dependencies: []
related: []
---

# Verify an RFC 7636 S256 PKCE Challenge

## Idea and Problem

Validate an OAuth PKCE verifier and compare its RFC 7636 S256 transformation with an exact unpadded challenge.

PKCE syntax is deliberately narrow: a verifier contains 43 to 128 ASCII
unreserved characters, while an S256 challenge is the 43-character unpadded
base64url encoding of the verifier's SHA-256 digest. Validating both values
before a constant-time comparison avoids accepting padding, Unicode, weakly
normalized method names, or other accidental protocol variants.

## When to Use

Use this recipe inside the token-exchange boundary of an OAuth authorization
server that has already recorded `S256` for the authorization request. The
stored challenge, authorization code, client, redirect URI, and user session
must already be bound by the surrounding protocol state. Use a maintained
OAuth implementation when you need the complete authorization flow rather
than this isolated comparison primitive.

## Implementation

```python
import base64
import hashlib
import hmac
import re


_VERIFIER_PATTERN = re.compile(r"[A-Za-z0-9._~-]{43,128}")
_S256_CHALLENGE_PATTERN = re.compile(r"[A-Za-z0-9_-]{43}")


def _validated_pkce_text(value: str, *, name: str, pattern: re.Pattern[str]) -> str:
    if not isinstance(value, str):
        raise TypeError(f"{name} must be text")
    if pattern.fullmatch(value) is None:
        raise ValueError(f"{name} has invalid RFC 7636 syntax")
    return value


def derive_s256_pkce_challenge(verifier: str) -> str:
    verifier = _validated_pkce_text(
        verifier,
        name="verifier",
        pattern=_VERIFIER_PATTERN,
    )
    digest = hashlib.sha256(verifier.encode("ascii")).digest()
    return base64.urlsafe_b64encode(digest).decode("ascii").rstrip("=")


def verify_s256_pkce_challenge(verifier: str, challenge: str) -> bool:
    challenge = _validated_pkce_text(
        challenge,
        name="challenge",
        pattern=_S256_CHALLENGE_PATTERN,
    )
    expected = derive_s256_pkce_challenge(verifier)
    return hmac.compare_digest(expected, challenge)
```

## Example

```python
verifier = "local-example-" + "A" * 29
challenge = "s8LrJiYqkTpbdqSTL2mv4uwsHAQ1NHXirWKge88qz1g"

derived = derive_s256_pkce_challenge(verifier)
matches = verify_s256_pkce_challenge(verifier, challenge)
mismatch = verify_s256_pkce_challenge("A" * 43, challenge)

try:
    verify_s256_pkce_challenge(verifier, challenge + "=")
except ValueError:
    padded_challenge_rejected = True
else:
    padded_challenge_rejected = False

try:
    derive_s256_pkce_challenge("\u00e9" * 43)
except ValueError:
    non_ascii_verifier_rejected = True
else:
    non_ascii_verifier_rejected = False

assert (
    derived,
    matches,
    mismatch,
    padded_challenge_rejected,
    non_ascii_verifier_rejected,
) == (challenge, True, False, True, True)
```

## Trade-offs and Limitations

This implements only the `S256` transformation and comparison defined by RFC
7636. It intentionally excludes `plain`, method negotiation, absent-PKCE
policy, padding tolerance, and downgrade behavior. Constant-time comparison
does not protect an otherwise incorrect OAuth flow: the caller still needs
TLS, high-entropy verifier generation, one-time authorization codes, secure
challenge storage, transaction binding, expiry, replay prevention, and
careful error responses. The function raises for malformed protocol values but
returns `False` for a well-formed mismatch so the surrounding endpoint can map
both outcomes to its own non-revealing response policy.

## Related Snippets

<!-- catalog:related:start -->
No related snippets.
<!-- catalog:related:end -->

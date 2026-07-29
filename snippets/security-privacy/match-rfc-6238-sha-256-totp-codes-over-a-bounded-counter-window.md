---
title: "Match RFC 6238 SHA-256 TOTP Codes over a Bounded Counter Window"
snippet_type: recipe
use_cases:
  - interoperability
  - security
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - authenticate-bounded-payloads-with-versioned-hmac-keys.md
  - create-and-verify-a-short-lived-hmac-download-url.md
  - verify-an-rfc-7636-s256-pkce-challenge.md
---

# Match RFC 6238 SHA-256 TOTP Codes over a Bounded Counter Window

## Idea and Problem

Match one fixed-profile TOTP code against every explicitly allowed time-step counter without hiding which counter matched.

[RFC 6238](https://www.rfc-editor.org/rfc/rfc6238.html) defines TOTP as HOTP
with a time-derived counter and permits SHA-256. Each counter is encoded as
eight big-endian bytes, authenticated with HMAC-SHA-256, dynamically
truncated, and reduced to eight decimal digits.

The matcher receives the counter rather than reading a clock. It evaluates the
complete bounded window and uses `hmac.compare_digest` for every candidate,
returning all matching counters in ascending order.

## When to Use

Use this primitive inside a system that already owns time-step calculation,
secret protection, replay state, rate limits, and authentication policy. The
closed profile is useful for interoperating with a provisioned 32-byte
SHA-256, eight-digit TOTP configuration.

Choose the narrowest acceptance window that tolerates the system's measured
clock skew and delivery delay. The supported zero-to-eight-step bounds are
implementation limits, not a recommendation to accept broad windows.

## Implementation

```python
import hmac

_MAX_UINT64 = (1 << 64) - 1
_MAX_WINDOW_STEPS = 8


def match_totp_sha256_counters(
    secret: bytes,
    code: str,
    counter: int,
    *,
    past_steps: int,
    future_steps: int,
) -> tuple[int, ...]:
    """Return every allowed counter whose eight-digit SHA-256 TOTP matches."""
    if type(secret) is not bytes:
        raise TypeError("secret must be exact bytes")
    if len(secret) != 32:
        raise ValueError("secret must contain exactly 32 bytes")
    if type(code) is not str:
        raise TypeError("code must be an exact string")
    if len(code) != 8 or any(character < "0" or character > "9" for character in code):
        raise ValueError("code must contain exactly eight ASCII digits")
    if type(counter) is not int:
        raise TypeError("counter must be an exact integer")
    if not 0 <= counter <= _MAX_UINT64:
        raise ValueError("counter is outside the unsigned 64-bit range")

    for name, steps in (("past_steps", past_steps), ("future_steps", future_steps)):
        if type(steps) is not int:
            raise TypeError(f"{name} must be an exact integer")
        if not 0 <= steps <= _MAX_WINDOW_STEPS:
            raise ValueError(f"{name} is outside the supported range")
    if past_steps > counter:
        raise ValueError("past_steps would cross counter zero")
    if future_steps > _MAX_UINT64 - counter:
        raise ValueError("future_steps would cross the uint64 boundary")

    matches: list[int] = []
    for candidate_counter in range(
        counter - past_steps,
        counter + future_steps + 1,
    ):
        digest = hmac.digest(
            secret,
            candidate_counter.to_bytes(8, "big"),
            "sha256",
        )
        offset = digest[-1] & 0x0F
        truncated = int.from_bytes(digest[offset : offset + 4], "big")
        candidate_code = f"{(truncated & 0x7FFFFFFF) % 100_000_000:08d}"
        if hmac.compare_digest(candidate_code, code):
            matches.append(candidate_counter)

    return tuple(matches)
```

## Example

```python
rfc_sha256_secret = b"12345678901234567890123456789012"
rfc_sha256_vectors = (
    (1, "46119246"),
    (37_037_036, "68084774"),
    (37_037_037, "67062674"),
    (41_152_263, "91819424"),
    (66_666_666, "90698825"),
    (666_666_666, "77737706"),
)

assert all(
    match_totp_sha256_counters(
        rfc_sha256_secret,
        expected_code,
        expected_counter,
        past_steps=0,
        future_steps=0,
    )
    == (expected_counter,)
    for expected_counter, expected_code in rfc_sha256_vectors
)
assert match_totp_sha256_counters(
    rfc_sha256_secret,
    "46119246",
    2,
    past_steps=1,
    future_steps=1,
) == (1,)
assert (
    match_totp_sha256_counters(
        rfc_sha256_secret,
        "46119247",
        1,
        past_steps=0,
        future_steps=0,
    )
    == ()
)
```

## Trade-offs and Limitations

The function performs exactly one SHA-256 HMAC and one fixed-length comparison
per admitted counter, at most 17 of each. Returned state is proportional to the
window. It does not exit after the first match, so the selected counter's
position does not change the number of candidate computations.

Input type, length, digit, and boundary validation occurs before HMAC work.
`compare_digest` reduces content-based comparison timing differences but does
not make the surrounding Python function, HMAC implementation, process, or
authentication system constant-time.

A code match is not a complete authentication decision. The caller must
derive counters from an agreed epoch and period, protect and provision keys,
consume successful counters once, rate-limit attempts, and record replay or
resynchronization state. This snippet provides no clock access, recovery code,
secret storage, persistence, replay prevention, or transport security.

## Related Snippets

<!-- catalog:related:start -->
- [Authenticate Bounded Payloads with Versioned HMAC Keys](authenticate-bounded-payloads-with-versioned-hmac-keys.md)
- [Create and Verify a Short-Lived HMAC Download URL](create-and-verify-a-short-lived-hmac-download-url.md)
- [Verify an RFC 7636 S256 PKCE Challenge](verify-an-rfc-7636-s256-pkce-challenge.md)
<!-- catalog:related:end -->

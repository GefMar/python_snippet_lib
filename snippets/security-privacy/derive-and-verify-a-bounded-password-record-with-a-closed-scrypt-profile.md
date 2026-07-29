---
title: "Derive and Verify a Bounded Password Record with a Closed scrypt Profile"
snippet_type: pattern
use_cases:
  - security
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - derive-bounded-key-material-with-rfc-5869-hkdf-sha-256.md
  - authenticate-bounded-payloads-with-versioned-hmac-keys.md
  - match-rfc-6238-sha-256-totp-codes-over-a-bounded-counter-window.md
---

# Derive and Verify a Bounded Password Record with a Closed scrypt Profile

## Idea and Problem

Derive a password-verification record with one fixed memory-hard profile so an untrusted stored record cannot choose excessive work factors.

The profile name maps to reviewed `hashlib.scrypt` parameters in code rather
than storing arbitrary cost values. The record carries only that name, a
16-byte salt, and a 32-byte derived value. Verification validates every field
before running the key derivation once and compares equal-length results with
`hmac.compare_digest`.

Salt generation remains explicit. The caller supplies fresh random bytes when
creating a record and persists the returned fields in its own authenticated
storage format.

## When to Use

Use this pattern as a narrow building block when a Python deployment has
calibrated the fixed profile for its authentication workload and its OpenSSL
build provides `hashlib.scrypt`. Generate a unique salt for every new record,
for example with `secrets.token_bytes(16)`.

Recalibrate memory and latency on the actual deployment before adopting this
profile. Use a mature password-authentication library when interoperable record
formats, automatic upgrades, peppers, breach workflows, or centralized policy
management are required.

## Implementation

```python
from dataclasses import dataclass
from hashlib import scrypt
from hmac import compare_digest

_PROFILE = "scrypt-v1"
_PASSWORD_MAX_BYTES = 1_024
_SALT_BYTES = 16
_DIGEST_BYTES = 32
_SCRYPT_N = 1 << 14
_SCRYPT_R = 8
_SCRYPT_P = 1
_SCRYPT_MAX_MEMORY = 32 << 20


@dataclass(frozen=True, slots=True)
class ScryptPasswordRecord:
    profile: str
    salt: bytes
    digest: bytes


def _validate_password(password: object) -> bytes:
    if type(password) is not bytes:
        raise TypeError("password must be exact bytes")
    if not 1 <= len(password) <= _PASSWORD_MAX_BYTES:
        raise ValueError("password length is outside the supported range")
    return password


def _validate_salt(salt: object) -> bytes:
    if type(salt) is not bytes:
        raise TypeError("salt must be exact bytes")
    if len(salt) != _SALT_BYTES:
        raise ValueError("salt must contain exactly 16 bytes")
    return salt


def _validate_record(record: object) -> ScryptPasswordRecord:
    if type(record) is not ScryptPasswordRecord:
        raise TypeError("record must be an exact ScryptPasswordRecord")
    if type(record.profile) is not str or record.profile != _PROFILE:
        raise ValueError("record has an unsupported scrypt profile")
    salt = _validate_salt(record.salt)
    if type(record.digest) is not bytes:
        raise TypeError("record digest must be exact bytes")
    if len(record.digest) != _DIGEST_BYTES:
        raise ValueError("record digest must contain exactly 32 bytes")
    return ScryptPasswordRecord(_PROFILE, salt, record.digest)


def _derive_scrypt(password: bytes, salt: bytes) -> bytes:
    return scrypt(
        password,
        salt=salt,
        n=_SCRYPT_N,
        r=_SCRYPT_R,
        p=_SCRYPT_P,
        maxmem=_SCRYPT_MAX_MEMORY,
        dklen=_DIGEST_BYTES,
    )


def derive_password_record(
    password: bytes,
    *,
    salt: bytes,
) -> ScryptPasswordRecord:
    """Derive one record under the fixed scrypt-v1 profile."""
    validated_password = _validate_password(password)
    validated_salt = _validate_salt(salt)
    return ScryptPasswordRecord(
        _PROFILE,
        validated_salt,
        _derive_scrypt(validated_password, validated_salt),
    )


def verify_password(
    password: bytes,
    record: ScryptPasswordRecord,
) -> bool:
    """Validate an untrusted record, derive once, and compare its digest."""
    validated_password = _validate_password(password)
    validated_record = _validate_record(record)
    candidate = _derive_scrypt(validated_password, validated_record.salt)
    return compare_digest(candidate, validated_record.digest)
```

## Example

```python
salt = bytes(range(16))
record = derive_password_record(
    b"correct horse battery staple",
    salt=salt,
)

assert record.digest.hex() == (
    "d7590aca2c9801cf06eeba772a69dc31"
    "ce3862591d96522ac4e6bba6ad1f31a5"
)
assert verify_password(b"correct horse battery staple", record)
assert not verify_password(b"incorrect horse battery staple", record)

changed_digest = record.digest[:-1] + bytes([record.digest[-1] ^ 1])
tampered = ScryptPasswordRecord(record.profile, record.salt, changed_digest)
assert not verify_password(b"correct horse battery staple", tampered)
```

## Trade-offs and Limitations

For the fixed parameters, scrypt has approximately `O(n * r * p)` derivation
work and `O(n * r)` memory, with implementation-dependent constants. Here that
means `n=16,384`, `r=8`, `p=1`, a 32 MiB OpenSSL memory ceiling, and one 32-byte
result. Verification deliberately performs no attacker-selected number of
rounds.

The chosen costs are an example deployment profile, not a permanent universal
recommendation. Authentication endpoints still need request limits, account
throttling, monitoring, secure transport, and a migration plan for stronger
future profiles. Python exposes scrypt only when built with suitable OpenSSL
support.

The helper accepts password bytes and therefore does not define text encoding
or Unicode normalization. It also does not generate or store salts, serialize
or authenticate records, manage a secret pepper, erase memory, upgrade old
records automatically, or decide password policy.

## Related Snippets

<!-- catalog:related:start -->
- [Derive Bounded Key Material with RFC 5869 HKDF-SHA-256](derive-bounded-key-material-with-rfc-5869-hkdf-sha-256.md)
- [Authenticate Bounded Payloads with Versioned HMAC Keys](authenticate-bounded-payloads-with-versioned-hmac-keys.md)
- [Match RFC 6238 SHA-256 TOTP Codes over a Bounded Counter Window](match-rfc-6238-sha-256-totp-codes-over-a-bounded-counter-window.md)
<!-- catalog:related:end -->

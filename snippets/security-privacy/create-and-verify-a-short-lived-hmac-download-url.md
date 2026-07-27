---
title: "Create and Verify a Short-Lived HMAC Download URL"
snippet_type: pattern
use_cases:
  - security
  - networking
tested_python:
  - "3.14"
dependencies: []
related:
  - authenticate-bounded-payloads-with-versioned-hmac-keys.md
  - ../networking-protocols/build-a-canonical-http-origin-key.md
  - ../networking-protocols/resume-a-bounded-http-byte-stream-with-validated-range-responses.md
---

# Create and Verify a Short-Lived HMAC Download URL

## Idea and Problem

Issue a replayable HTTPS capability URL whose exact endpoint, resource, audience, issuance time, expiry, and key ID are authenticated together.

A versioned, length-framed HMAC payload prevents field-boundary ambiguity, and
a canonical fixed query grammar prevents alternate encodings from representing
the same claims. Verification uses the declared key directly, enforces both the
maximum issued lifetime and clock-skew bounds, and compares the tag in constant
time.

## When to Use

Use this pattern for a small trusted issuer and verifier that share a bounded
set of high-entropy HMAC keys and expose one fixed HTTPS download endpoint.
Treat the resource ID as an opaque authorization identity rather than deriving
it from a filename stem. Rotate by selecting a new active signing key ID while
retaining an old key for verification only until its last URL expires.

Use a standard capability service or managed signing system when several trust
domains, per-token revocation, audit trails, asymmetric verification, or remote
key management are required. The URL is a bearer credential, so callers must
protect it even though its query fields remain visible.

## Implementation

```python
import hmac
import math
import re
import time
from collections.abc import Callable, Mapping
from dataclasses import dataclass
from ipaddress import IPv4Address, IPv6Address, ip_address
from types import MappingProxyType
from urllib.parse import parse_qsl, urlencode, urlsplit


_DOMAIN = b"short-lived-download-capability-v1\x00"
_MAX_KEYS = 16
_MIN_KEY_BYTES = 32
_MAX_KEY_BYTES = 128
_MAX_URL_LENGTH = 2048
_MAX_ENDPOINT_LENGTH = 512
_MAX_RESOURCE_LENGTH = 128
_MAX_AUDIENCE_LENGTH = 64
_MAX_KEY_ID_LENGTH = 32
_MAX_TIMESTAMP = 9_999_999_999
_TOKEN_TEXT = re.compile(r"[A-Za-z0-9][A-Za-z0-9._~-]*", re.ASCII)
_DNS_LABEL = re.compile(
    r"[a-z0-9](?:[a-z0-9-]{0,61}[a-z0-9])?",
    re.ASCII,
)
_LEGACY_NUMERIC_LABEL = re.compile(r"(?:[0-9]+|0x[0-9a-f]+)", re.ASCII)
_ENDPOINT_PATH = re.compile(r"/[A-Za-z0-9._~/-]{0,127}", re.ASCII)
_HEX_TAG = re.compile(r"[0-9a-f]{64}", re.ASCII)
_QUERY_NAMES = ("v", "kid", "rid", "aud", "iat", "exp", "sig")


@dataclass(frozen=True, slots=True)
class DownloadGrant:
    resource_id: str
    audience: str
    issued_at: int
    expires_at: int
    key_id: str


def _validate_token_text(value: object, *, name: str, maximum: int) -> str:
    if not isinstance(value, str):
        raise TypeError(f"{name} must be text")
    if not 1 <= len(value) <= maximum or _TOKEN_TEXT.fullmatch(value) is None:
        raise ValueError(f"{name} has an invalid format")
    return value


def _canonical_https_endpoint(endpoint: str) -> str:
    if not isinstance(endpoint, str):
        raise TypeError("endpoint must be text")
    if not 1 <= len(endpoint) <= _MAX_ENDPOINT_LENGTH or not endpoint.isascii():
        raise ValueError("endpoint length or encoding is outside the supported range")
    if any(ord(character) <= 0x20 or ord(character) == 0x7F for character in endpoint):
        raise ValueError("endpoint must not contain whitespace or controls")
    if "?" in endpoint or "#" in endpoint:
        raise ValueError("endpoint must not contain a query or fragment")

    try:
        parsed = urlsplit(endpoint)
        host = parsed.hostname
        port = parsed.port
    except ValueError as error:
        raise ValueError("endpoint has an invalid authority") from error
    if parsed.scheme.lower() != "https" or not parsed.netloc or host is None:
        raise ValueError("endpoint must be an absolute HTTPS URL")
    if parsed.username is not None or parsed.password is not None:
        raise ValueError("endpoint must not contain credentials")
    if parsed.netloc.endswith(":") or port == 0:
        raise ValueError("endpoint has an invalid port")
    if _ENDPOINT_PATH.fullmatch(parsed.path) is None:
        raise ValueError("endpoint path has an invalid format")
    segments = parsed.path.split("/")[1:]
    if any(segment in {"", ".", ".."} for segment in segments):
        raise ValueError("endpoint path must use canonical segments")

    if "%" in host:
        raise ValueError("endpoint host must not contain a zone identifier")
    try:
        address = ip_address(host)
    except ValueError:
        normalized_host = host.lower()
        if not normalized_host.isascii() or len(normalized_host) > 253:
            raise ValueError("endpoint DNS name is invalid")
        if any(
            _DNS_LABEL.fullmatch(label) is None
            for label in normalized_host.split(".")
        ):
            raise ValueError("endpoint DNS name is invalid")
        if all(
            _LEGACY_NUMERIC_LABEL.fullmatch(label)
            for label in normalized_host.split(".")
        ):
            raise ValueError("ambiguous numeric endpoint host is not supported")
    else:
        if isinstance(address, IPv4Address):
            normalized_host = str(address)
        elif isinstance(address, IPv6Address):
            normalized_host = f"[{address.compressed}]"
        else:
            raise AssertionError("unexpected IP address family")

    port_text = "" if port in {None, 443} else f":{port}"
    return f"https://{normalized_host}{port_text}{parsed.path}"


def _framed_claims(
    endpoint: str,
    key_id: str,
    resource_id: str,
    audience: str,
    issued_at: int,
    expires_at: int,
) -> bytes:
    parts = (
        "GET",
        endpoint,
        key_id,
        resource_id,
        audience,
        str(issued_at),
        str(expires_at),
    )
    framed = [_DOMAIN]
    for part in parts:
        encoded = part.encode("ascii")
        framed.extend((len(encoded).to_bytes(2, "big"), encoded))
    return b"".join(framed)


def _canonical_query(grant: DownloadGrant, signature: str) -> str:
    return urlencode(
        (
            ("v", "1"),
            ("kid", grant.key_id),
            ("rid", grant.resource_id),
            ("aud", grant.audience),
            ("iat", str(grant.issued_at)),
            ("exp", str(grant.expires_at)),
            ("sig", signature),
        )
    )


class DownloadURLSigner:
    def __init__(
        self,
        endpoint: str,
        audience: str,
        keys: Mapping[str, bytes],
        *,
        active_signing_key_id: str,
        clock: Callable[[], float] = time.time,
        max_lifetime_seconds: int = 900,
        future_skew_seconds: int = 5,
        expiry_skew_seconds: int = 5,
    ) -> None:
        if not isinstance(keys, Mapping) or not 1 <= len(keys) <= _MAX_KEYS:
            raise ValueError("key count is outside the supported range")
        if not callable(clock):
            raise TypeError("clock must be callable")
        for name, value, lower, upper in (
            ("max_lifetime_seconds", max_lifetime_seconds, 1, 86_400),
            ("future_skew_seconds", future_skew_seconds, 0, 300),
            ("expiry_skew_seconds", expiry_skew_seconds, 0, 300),
        ):
            if isinstance(value, bool) or not isinstance(value, int):
                raise TypeError(f"{name} must be an integer")
            if not lower <= value <= upper:
                raise ValueError(f"{name} is outside the supported range")

        copied = {}
        for raw_key_id, raw_key in keys.items():
            key_id = _validate_token_text(
                raw_key_id,
                name="key_id",
                maximum=_MAX_KEY_ID_LENGTH,
            )
            if not isinstance(raw_key, bytes):
                raise TypeError("HMAC keys must be immutable bytes")
            if not _MIN_KEY_BYTES <= len(raw_key) <= _MAX_KEY_BYTES:
                raise ValueError("an HMAC key has an unsafe length")
            copied[key_id] = raw_key

        self._endpoint = _canonical_https_endpoint(endpoint)
        self._audience = _validate_token_text(
            audience,
            name="audience",
            maximum=_MAX_AUDIENCE_LENGTH,
        )
        self._keys = MappingProxyType(copied)
        active_key_id = _validate_token_text(
            active_signing_key_id,
            name="active_signing_key_id",
            maximum=_MAX_KEY_ID_LENGTH,
        )
        if active_key_id not in copied:
            raise ValueError("active signing key ID is not present in keys")
        self._active_signing_key_id = active_key_id
        self._clock = clock
        self._max_lifetime = max_lifetime_seconds
        self._future_skew = future_skew_seconds
        self._expiry_skew = expiry_skew_seconds

    def _now(self) -> int:
        value = self._clock()
        if isinstance(value, bool) or not isinstance(value, (int, float)):
            raise TypeError("clock must return a number")
        timestamp = float(value)
        if not math.isfinite(timestamp) or not 0 <= timestamp <= _MAX_TIMESTAMP:
            raise ValueError("clock returned an invalid timestamp")
        return int(timestamp)

    def issue(
        self,
        resource_id: str,
        *,
        lifetime_seconds: int,
    ) -> str:
        resource_id = _validate_token_text(
            resource_id,
            name="resource_id",
            maximum=_MAX_RESOURCE_LENGTH,
        )
        if isinstance(lifetime_seconds, bool) or not isinstance(lifetime_seconds, int):
            raise TypeError("lifetime_seconds must be an integer")
        if not 1 <= lifetime_seconds <= self._max_lifetime:
            raise ValueError("lifetime_seconds is outside the supported range")
        key_id = self._active_signing_key_id
        key = self._keys[key_id]

        issued_at = self._now()
        expires_at = issued_at + lifetime_seconds
        if expires_at > _MAX_TIMESTAMP:
            raise ValueError("expiry is outside the token format")
        grant = DownloadGrant(
            resource_id,
            self._audience,
            issued_at,
            expires_at,
            key_id,
        )
        tag = hmac.digest(
            key,
            _framed_claims(
                self._endpoint,
                key_id,
                resource_id,
                self._audience,
                issued_at,
                expires_at,
            ),
            "sha256",
        ).hex()
        url = f"{self._endpoint}?{_canonical_query(grant, tag)}"
        if len(url) > _MAX_URL_LENGTH:
            raise ValueError("signed URL exceeds the supported length")
        return url

    def verify(self, url: object, *, method: object) -> DownloadGrant | None:
        if method != "GET" or not isinstance(url, str):
            return None
        if not 1 <= len(url) <= _MAX_URL_LENGTH or not url.isascii():
            return None
        if any(ord(character) <= 0x20 or ord(character) == 0x7F for character in url):
            return None
        try:
            parsed = urlsplit(url)
        except ValueError:
            return None
        if "#" in url or not parsed.query:
            return None
        if f"{parsed.scheme}://{parsed.netloc}{parsed.path}" != self._endpoint:
            return None

        try:
            pairs = parse_qsl(
                parsed.query,
                keep_blank_values=True,
                strict_parsing=True,
                max_num_fields=len(_QUERY_NAMES),
                encoding="ascii",
                errors="strict",
            )
        except (UnicodeError, ValueError):
            return None
        if len(pairs) != len(_QUERY_NAMES):
            return None
        fields = dict(pairs)
        if len(fields) != len(pairs) or tuple(name for name, _ in pairs) != _QUERY_NAMES:
            return None
        if fields["v"] != "1" or fields["aud"] != self._audience:
            return None

        try:
            resource_id = _validate_token_text(
                fields["rid"],
                name="resource_id",
                maximum=_MAX_RESOURCE_LENGTH,
            )
            key_id = _validate_token_text(
                fields["kid"],
                name="key_id",
                maximum=_MAX_KEY_ID_LENGTH,
            )
        except (TypeError, ValueError):
            return None
        for timestamp_name in ("iat", "exp"):
            text = fields[timestamp_name]
            if not 1 <= len(text) <= 10 or not text.isascii() or not text.isdecimal():
                return None
            if str(int(text)) != text:
                return None
        issued_at = int(fields["iat"])
        expires_at = int(fields["exp"])
        if issued_at > _MAX_TIMESTAMP or expires_at > _MAX_TIMESTAMP:
            return None
        if _HEX_TAG.fullmatch(fields["sig"]) is None:
            return None

        grant = DownloadGrant(
            resource_id,
            self._audience,
            issued_at,
            expires_at,
            key_id,
        )
        if parsed.query != _canonical_query(grant, fields["sig"]):
            return None
        lifetime = expires_at - issued_at
        if not 1 <= lifetime <= self._max_lifetime:
            return None
        now = self._now()
        if issued_at > now + self._future_skew:
            return None
        if now > expires_at + self._expiry_skew:
            return None

        key = self._keys.get(key_id)
        if key is None:
            return None
        expected_tag = hmac.digest(
            key,
            _framed_claims(
                self._endpoint,
                key_id,
                resource_id,
                self._audience,
                issued_at,
                expires_at,
            ),
            "sha256",
        ).hex()
        if not hmac.compare_digest(expected_tag, fields["sig"]):
            return None
        return grant
```

## Example

```python
current_time = 1_800_000_000
# Test-only bytes; use managed key material or secrets.token_bytes(32) in production.
test_keys = {"key-1": bytes(range(32))}
signer = DownloadURLSigner(
    "https://downloads.example.com/fetch",
    "documentation-reader",
    test_keys,
    active_signing_key_id="key-1",
    clock=lambda: current_time,
    max_lifetime_seconds=300,
    future_skew_seconds=2,
    expiry_skew_seconds=2,
)

signed_url = signer.issue(
    "manual-v2.pdf",
    lifetime_seconds=60,
)
grant = signer.verify(signed_url, method="GET")
tampered_url = signed_url.replace("manual-v2.pdf", "manual-v3.pdf")

try:
    DownloadURLSigner(
        "https://0x7f000001/fetch",
        "documentation-reader",
        test_keys,
        active_signing_key_id="key-1",
        clock=lambda: current_time,
    )
except ValueError:
    legacy_numeric_endpoint_rejected = True
else:
    legacy_numeric_endpoint_rejected = False

assert (
    grant,
    signer.verify(tampered_url, method="GET"),
    signer.verify(signed_url, method="POST"),
    signer.verify(signed_url, method="GET"),
    legacy_numeric_endpoint_rejected,
) == (
    DownloadGrant(
        "manual-v2.pdf",
        "documentation-reader",
        1_800_000_000,
        1_800_000_060,
        "key-1",
    ),
    None,
    None,
    grant,
    True,
)
```

## Trade-offs and Limitations

The URL is deliberately replayable until expiry plus the configured skew; two
successful verifications do not make it one-time use. Query strings can appear
in browser history, access logs, analytics, copied messages, and referrer data.
HMAC provides integrity and authenticity, not confidentiality. HTTPS protects
the capability in transit but does not prevent leakage at either endpoint.

The verifier accepts only one canonical parameter order and spelling, one
configured endpoint, the request's exact `GET` method, and bounded
ASCII resource and audience IDs. It performs one local key lookup and one HMAC,
with no fallback scan or remote retrieval. Removing a key revokes every URL
under that key; otherwise there is no per-URL revocation. Issuer and verifier
wall clocks must remain within the chosen skew, and rotation must retain old
keys long enough for already-issued URLs while preventing new issuance with
retired IDs.

## Related Snippets

<!-- catalog:related:start -->
- [Authenticate Bounded Payloads with Versioned HMAC Keys](authenticate-bounded-payloads-with-versioned-hmac-keys.md)
- [Build a Canonical HTTP Origin Key](../networking-protocols/build-a-canonical-http-origin-key.md)
- [Resume a Bounded HTTP Byte Stream with Validated Range Responses](../networking-protocols/resume-a-bounded-http-byte-stream-with-validated-range-responses.md)
<!-- catalog:related:end -->

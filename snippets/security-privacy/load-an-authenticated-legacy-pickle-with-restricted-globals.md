---
title: "Load an Authenticated Legacy Pickle with Restricted Globals"
snippet_type: recipe
use_cases:
  - interoperability
  - security
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - authenticate-bounded-payloads-with-versioned-hmac-keys.md
  - ../configuration-serialization/evaluate-a-bounded-boolean-tag-expression-with-an-ast-allowlist.md
  - ../configuration-serialization/parse-a-bounded-nested-bracket-tree.md
---

# Load an Authenticated Legacy Pickle with Restricted Globals

## Idea and Problem

Never unpickle untrusted or attacker-modifiable data; for an unavoidable authenticated legacy channel, verify a bounded envelope before restricting every global lookup to an immutable exact allowlist.

The loader authenticates a domain-separated context and payload before it
constructs an `Unpickler`. It rejects extension opcodes, persistent references,
out-of-band buffers, and trailing bytes, then resolves globals only from
preloaded `(module, name)` entries without importing anything.

## When to Use

Use this defense-in-depth recipe only while migrating a legacy format whose
producer and symmetric key are trusted, whose messages are protected against
tampering, and whose allowed object graph has been audited. Give each schema
or purpose a different context and key, keep the allowlist minimal, and
validate the loaded value against its expected type and schema afterward.

Prefer JSON, MessagePack with a fixed schema, Protocol Buffers, or another
non-executable format for new interfaces. Authentication proves that a sender
possesses the key; it does not prove that the sender, key, or an allowed class
is harmless.

## Implementation

```python
import hmac
import io
import pickle
import pickletools
from collections.abc import Mapping
from types import MappingProxyType
from typing import BinaryIO


_TAG_BYTES = 32
_MIN_KEY_BYTES = 32
_MAX_KEY_BYTES = 64
_MAX_CONTEXT_BYTES = 128
_MAX_PAYLOAD_BYTES = 1024 * 1024
_MAX_ALLOWED_GLOBALS = 64
_MAC_DOMAIN = b"authenticated-legacy-pickle:v1\x00"
_FORBIDDEN_OPCODES = frozenset(
    {
        "BINPERSID",
        "EXT1",
        "EXT2",
        "EXT4",
        "NEXT_BUFFER",
        "PERSID",
        "READONLY_BUFFER",
    }
)


class LegacyPickleError(RuntimeError):
    pass


class LegacyPickleAuthenticationError(LegacyPickleError):
    pass


class LegacyPickleFormatError(LegacyPickleError):
    pass


class LegacyPickleLimitError(LegacyPickleError):
    pass


def _exact_bytes(value: bytes, *, name: str, lower: int, upper: int) -> bytes:
    if type(value) is not bytes:
        raise TypeError(f"{name} must be immutable bytes")
    if not lower <= len(value) <= upper:
        raise ValueError(f"{name} length is outside the supported range")
    return value


def _mac_input(context: bytes, payload: bytes) -> bytes:
    return b"".join(
        (
            _MAC_DOMAIN,
            len(context).to_bytes(2, "big"),
            context,
            len(payload).to_bytes(8, "big"),
            payload,
        )
    )


def _validate_pickle_stream(payload: bytes) -> None:
    stop_position: int | None = None
    try:
        for opcode, _argument, position in pickletools.genops(payload):
            if opcode.name in _FORBIDDEN_OPCODES:
                raise LegacyPickleFormatError(
                    f"pickle opcode {opcode.name} is forbidden"
                )
            if opcode.name == "STOP":
                stop_position = position
    except LegacyPickleFormatError:
        raise
    except ValueError as error:
        raise LegacyPickleFormatError("pickle framing is malformed") from error
    if stop_position is None or stop_position + 1 != len(payload):
        raise LegacyPickleFormatError("exactly one pickle object is required")


class _RestrictedUnpickler(pickle.Unpickler):
    def __init__(
        self,
        stream: BinaryIO,
        allowed_globals: Mapping[tuple[str, str], object],
    ) -> None:
        super().__init__(
            stream,
            fix_imports=False,
            encoding="ASCII",
            errors="strict",
            buffers=(),
        )
        self._allowed_globals = allowed_globals

    def find_class(self, module: str, name: str) -> object:
        try:
            return self._allowed_globals[(module, name)]
        except KeyError:
            raise pickle.UnpicklingError(
                f"global {module!r}.{name!r} is forbidden"
            ) from None

    def persistent_load(self, persistent_id: object) -> object:
        raise pickle.UnpicklingError("persistent IDs are forbidden")


class AuthenticatedLegacyPickleLoader:
    def __init__(
        self,
        *,
        key: bytes,
        context: bytes,
        allowed_globals: Mapping[tuple[str, str], object],
        max_payload_bytes: int,
    ) -> None:
        self._key = _exact_bytes(
            key,
            name="key",
            lower=_MIN_KEY_BYTES,
            upper=_MAX_KEY_BYTES,
        )
        self._context = _exact_bytes(
            context,
            name="context",
            lower=1,
            upper=_MAX_CONTEXT_BYTES,
        )
        if (
            isinstance(max_payload_bytes, bool)
            or not isinstance(max_payload_bytes, int)
        ):
            raise TypeError("max_payload_bytes must be an integer")
        if not 1 <= max_payload_bytes <= _MAX_PAYLOAD_BYTES:
            raise ValueError("max_payload_bytes is outside the supported range")
        if not isinstance(allowed_globals, Mapping):
            raise TypeError("allowed_globals must be a mapping")
        if len(allowed_globals) > _MAX_ALLOWED_GLOBALS:
            raise ValueError("too many allowed globals")

        copied_globals: dict[tuple[str, str], object] = {}
        for global_name, resolved_object in allowed_globals.items():
            if (
                type(global_name) is not tuple
                or len(global_name) != 2
                or any(type(part) is not str for part in global_name)
                or any(
                    not part
                    or len(part) > 200
                    or not part.isascii()
                    or not part.isprintable()
                    for part in global_name
                )
            ):
                raise ValueError(
                    "allowlist keys must be bounded exact module-name pairs"
                )
            copied_globals[global_name] = resolved_object

        self._allowed_globals = MappingProxyType(copied_globals)
        self._max_payload_bytes = max_payload_bytes

    def load(self, envelope: bytes) -> object:
        if type(envelope) is not bytes:
            raise TypeError("envelope must be immutable bytes")
        if len(envelope) < _TAG_BYTES:
            raise LegacyPickleAuthenticationError("the envelope tag is missing")
        payload = envelope[_TAG_BYTES:]
        if len(payload) > self._max_payload_bytes:
            raise LegacyPickleLimitError("the pickle payload is too large")

        supplied_tag = envelope[:_TAG_BYTES]
        expected_tag = hmac.digest(
            self._key,
            _mac_input(self._context, payload),
            "sha256",
        )
        if not hmac.compare_digest(supplied_tag, expected_tag):
            raise LegacyPickleAuthenticationError("the envelope is not authentic")

        _validate_pickle_stream(payload)
        try:
            return _RestrictedUnpickler(
                io.BytesIO(payload),
                self._allowed_globals,
            ).load()
        except (MemoryError, RecursionError):
            raise
        except Exception as error:
            raise LegacyPickleFormatError("the pickle violates the load policy") from error
```

## Example

```python
from decimal import Decimal


# Deterministic test key; never use repeated bytes in production.
key = b"k" * 32
context = b"decimal-record:v1"
mutable_allowlist = {("decimal", "Decimal"): Decimal}
loader = AuthenticatedLegacyPickleLoader(
    key=key,
    context=context,
    allowed_globals=mutable_allowlist,
    max_payload_bytes=4096,
)
mutable_allowlist.clear()


def producer_envelope(payload: bytes) -> bytes:
    tag = hmac.digest(key, _mac_input(context, payload), "sha256")
    return tag + payload


safe_payload = pickle.dumps(Decimal("12.50"), protocol=5)
loaded = loader.load(producer_envelope(safe_payload))

tampered = bytearray(producer_envelope(safe_payload))
tampered[-1] ^= 1
try:
    loader.load(bytes(tampered))
except LegacyPickleAuthenticationError:
    tampering_rejected = True
else:
    tampering_rejected = False

dangerous_payload = pickle.dumps(eval, protocol=5)
try:
    loader.load(producer_envelope(dangerous_payload))
except LegacyPickleFormatError:
    unlisted_global_rejected = True
else:
    unlisted_global_rejected = False

try:
    loader.load(producer_envelope(safe_payload + b"."))
except LegacyPickleFormatError:
    trailing_data_rejected = True
else:
    trailing_data_rejected = False

malformed_stack = bytes.fromhex("4e4e612e")
try:
    loader.load(producer_envelope(malformed_stack))
except LegacyPickleFormatError:
    malformed_stack_rejected = True
else:
    malformed_stack_rejected = False

assert (
    loaded,
    type(loaded) is Decimal,
    tampering_rejected,
    unlisted_global_rejected,
    trailing_data_rejected,
    malformed_stack_rejected,
) == (
    Decimal("12.50"),
    True,
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

This is not a sandbox and does not make hostile pickle safe. A compromised
trusted producer or key can select dangerous behavior through any allowed
callable or class. Constructors, `__new__`, `__setstate__`, reduction methods,
attribute hooks, and permitted object graphs can execute code or allocate far
more CPU and memory than the bounded input size suggests. Post-load validation
happens too late to prevent those effects.

Ordinary exceptions raised during reconstruction are normalized to
`LegacyPickleFormatError`, so callers cannot distinguish malformed stack
operations from failures inside allowed constructors by exception type alone.
`MemoryError`, `RecursionError`, and `BaseException` subclasses propagate
instead of being mislabeled as ordinary format failures.

The payload cap bounds authentication and parsing input, not expansion during
unpickling. The fixed allowlist must be reviewed whenever its objects change,
and production keys need independent high-entropy generation, secure storage,
rotation, and purpose separation. Authentication alone accepts replay or
rollback to any older valid envelope under the same key and context; bind and
check an external record identity, version, or monotonic sequence when
freshness matters. Rejecting extension and out-of-band opcodes avoids alternate
global caches and external buffers but reduces compatibility. The example's
envelope helper belongs only on the trusted producer side; consumers must never
sign arbitrary bytes merely to make them loadable.

## Related Snippets

<!-- catalog:related:start -->
- [Authenticate Bounded Payloads with Versioned HMAC Keys](authenticate-bounded-payloads-with-versioned-hmac-keys.md)
- [Evaluate a Bounded Boolean Tag Expression with an AST Allowlist](../configuration-serialization/evaluate-a-bounded-boolean-tag-expression-with-an-ast-allowlist.md)
- [Parse a Bounded Nested Bracket Tree](../configuration-serialization/parse-a-bounded-nested-bracket-tree.md)
<!-- catalog:related:end -->

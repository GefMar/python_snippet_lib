---
title: "Adapt Bounded JSON Output to a Protobuf Message"
snippet_type: integration
use_cases:
  - interoperability
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies:
  - name: protobuf
    version: "7.35.1"
related:
  - load-a-bounded-protobuf-descriptor-set-in-dependency-order.md
  - migrate-one-bounded-json-record-to-a-current-version.md
---

# Adapt Bounded JSON Output to a Protobuf Message

## Idea and Problem

Validate one already-captured JSON byte string before adapting its sole object into a fresh Protobuf message.

The boundary caps bytes, decodes strict UTF-8, detects duplicate keys and
non-finite numbers, and bounds the complete JSON tree before the official
Protobuf JSON adapter sees it. A zero-argument factory supplies a fresh message,
so a rejected conversion cannot partially mutate a caller-owned destination.

## When to Use

Use this integration when another component has already captured a small JSON
result and downstream code needs a generated Protobuf message. The message
schema and factory must be trusted and local, while the captured bytes may be
malformed or schema-incompatible.

Keep process execution, environment lookup, credential handling, transport,
and output capture outside this function. Use binary Protobuf parsing when the
producer can emit the binary wire format, and use a streaming parser when a
64-KiB in-memory boundary is not appropriate.

## Implementation

```python
import json
import math
import re
from collections.abc import Callable
from typing import TypeVar

from google.protobuf import json_format
from google.protobuf.descriptor import FieldDescriptor
from google.protobuf.message import Message

MessageT = TypeVar("MessageT", bound=Message)

_MAX_INPUT_BYTES = 64 * 1024
_MAX_DEPTH = 16
_MAX_NODES = 1024
_JSON_WHITESPACE = re.compile(r"[ \t\r\n]*", re.ASCII)
_ERROR_CODES = frozenset(
    {
        "depth_limit",
        "duplicate_key",
        "factory_failed",
        "factory_result",
        "input_size",
        "input_type",
        "invalid_json",
        "invalid_utf8",
        "node_limit",
        "non_finite_number",
        "protobuf_rejected",
        "root_not_object",
        "trailing_data",
    }
)


class ProtobufJSONError(ValueError):
    __slots__ = ("code",)

    def __init__(self, code: str) -> None:
        if code not in _ERROR_CODES:
            raise ValueError("invalid_error_code")
        self.code = code
        super().__init__(code)


class _JSONFailure(Exception):
    __slots__ = ("code",)

    def __init__(self, code: str) -> None:
        self.code = code
        super().__init__(code)


def _unique_object(pairs: list[tuple[str, object]]) -> dict[str, object]:
    result: dict[str, object] = {}
    for key, value in pairs:
        if key in result:
            raise _JSONFailure("duplicate_key")
        result[key] = value
    return result


def _reject_constant(_text: str) -> object:
    raise _JSONFailure("non_finite_number")


def _parse_float(text: str) -> float:
    value = float(text)
    if not math.isfinite(value):
        raise _JSONFailure("non_finite_number")
    return value


def _decode_utf8(payload: bytes) -> tuple[str | None, str | None]:
    try:
        return payload.decode("utf-8", errors="strict"), None
    except UnicodeDecodeError:
        return None, "invalid_utf8"


def _decode_one_json_value(text: str) -> tuple[object | None, str | None]:
    decoder = json.JSONDecoder(
        object_pairs_hook=_unique_object,
        parse_constant=_reject_constant,
        parse_float=_parse_float,
        strict=True,
    )
    start = _JSON_WHITESPACE.match(text).end()
    try:
        document, end = decoder.raw_decode(text, idx=start)
    except _JSONFailure as error:
        return None, error.code
    except (json.JSONDecodeError, RecursionError, ValueError, OverflowError):
        return None, "invalid_json"
    if _JSON_WHITESPACE.fullmatch(text, pos=end) is None:
        return None, "trailing_data"
    return document, None


def _tree_limit_error(root: object) -> str | None:
    nodes = 0
    stack = [(root, 0)]
    while stack:
        value, depth = stack.pop()
        nodes += 1
        if nodes > _MAX_NODES:
            return "node_limit"
        if depth > _MAX_DEPTH:
            return "depth_limit"
        if type(value) is dict:
            stack.extend((child, depth + 1) for child in value.values())
        elif type(value) is list:
            stack.extend((child, depth + 1) for child in value)
    return None


def _fresh_populated_message(
    document: dict[str, object],
    factory: Callable[[], MessageT],
) -> tuple[MessageT | None, str | None]:
    try:
        message = factory()
    except Exception:
        return None, "factory_failed"
    if not isinstance(message, Message) or message.ListFields():
        return None, "factory_result"
    try:
        json_format.ParseDict(
            document,
            message,
            ignore_unknown_fields=False,
        )
    except Exception:
        return None, "protobuf_rejected"
    if _contains_non_finite_float(message):
        return None, "non_finite_number"
    return message, None


def _contains_non_finite_float(root: Message) -> bool:
    stack = [root]
    while stack:
        message = stack.pop()
        for field, supplied in message.ListFields():
            if field.is_repeated:
                if field.message_type is not None and field.message_type.GetOptions().map_entry:
                    value_field = field.message_type.fields_by_name["value"]
                    values = supplied.values()
                else:
                    value_field = field
                    values = supplied
            else:
                value_field = field
                values = (supplied,)

            if value_field.type in (
                FieldDescriptor.TYPE_DOUBLE,
                FieldDescriptor.TYPE_FLOAT,
            ):
                if any(not math.isfinite(value) for value in values):
                    return True
            elif value_field.cpp_type == FieldDescriptor.CPPTYPE_MESSAGE:
                stack.extend(values)
    return False


def adapt_bounded_json_to_protobuf(
    payload: bytes,
    factory: Callable[[], MessageT],
) -> MessageT:
    if type(payload) is not bytes:
        raise ProtobufJSONError("input_type")
    if not 1 <= len(payload) <= _MAX_INPUT_BYTES:
        raise ProtobufJSONError("input_size")
    text, text_error = _decode_utf8(payload)
    if text_error is not None:
        raise ProtobufJSONError(text_error)
    if text is None:
        raise ProtobufJSONError("invalid_utf8")

    document, decode_error = _decode_one_json_value(text)
    if decode_error is not None:
        raise ProtobufJSONError(decode_error)
    if type(document) is not dict:
        raise ProtobufJSONError("root_not_object")
    if (limit_error := _tree_limit_error(document)) is not None:
        raise ProtobufJSONError(limit_error)

    message, population_error = _fresh_populated_message(document, factory)
    if population_error is not None:
        raise ProtobufJSONError(population_error)
    if message is None:
        raise ProtobufJSONError("factory_result")
    return message
```

## Example

```python
def new_file_descriptor() -> Message:
    from google.protobuf import descriptor_pb2

    return descriptor_pb2.FileDescriptorProto()


payload = b'{"name":"orchid/event.proto","package":"orchid","syntax":"proto3"}'
message = adapt_bounded_json_to_protobuf(
    payload,
    new_file_descriptor,
)


def rejection_code(
    candidate: bytes,
    factory: Callable[[], Message] = new_file_descriptor,
) -> str:
    try:
        adapt_bounded_json_to_protobuf(candidate, factory)
    except ProtobufJSONError as error:
        return error.code
    raise AssertionError("candidate was accepted")


duplicate = rejection_code(b'{"name":"first.proto","name":"second.proto"}')
unknown = rejection_code(b'{"name":"orchid/event.proto","owner":"violet"}')
trailing = rejection_code(b'{"name":"orchid/event.proto"} null')


def new_uninterpreted_option() -> Message:
    from google.protobuf import descriptor_pb2

    return descriptor_pb2.UninterpretedOption()


non_finite = rejection_code(
    b'{"doubleValue":"NaN"}',
    new_uninterpreted_option,
)

assert (message.name, message.package, duplicate, unknown, trailing, non_finite) == (
    "orchid/event.proto",
    "orchid",
    "duplicate_key",
    "protobuf_rejected",
    "trailing_data",
    "non_finite_number",
)
```

## Trade-offs and Limitations

The root object has depth zero, every JSON value including the root counts as
one of 1,024 nodes, and object keys do not add nodes. Parsing still materializes
the complete document and then lets the Protobuf adapter traverse it again,
but the 65,536-byte input ceiling bounds that work. Numeric range, enum, map,
well-known-type, and field-name semantics remain those of the pinned Protobuf
JSON adapter. A post-conversion reflection pass also rejects non-finite float
or double fields, including the Protobuf JSON string spellings for those
values.

All schema and conversion failures intentionally collapse to short stable
codes, which makes the public error safe but unsuitable for detailed debugging.
The zero-argument factory must return a new, unaliased and initially empty
message on every call. The function rejects a pre-populated result, but Python
cannot prove that an empty result is unaliased. When the factory honors the
contract, only the new candidate can be partially mutated before a rejection,
and that candidate is discarded. No subprocess, environment, token, network,
descriptor download, or caller-supplied destination is accessed.

## Related Snippets

<!-- catalog:related:start -->
- [Load a Bounded Protobuf Descriptor Set in Dependency Order](load-a-bounded-protobuf-descriptor-set-in-dependency-order.md)
- [Migrate One Bounded JSON Record to a Current Version](migrate-one-bounded-json-record-to-a-current-version.md)
<!-- catalog:related:end -->

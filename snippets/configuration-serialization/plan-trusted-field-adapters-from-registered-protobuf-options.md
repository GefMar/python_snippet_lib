---
title: "Plan Trusted Field Adapters from Registered Protobuf Options"
snippet_type: integration
use_cases:
  - configuration
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
  - resolve-bounded-configuration-through-dependent-adapters.md
  - ../python-language/dispatch-named-strategies-with-an-explicit-function-mapping.md
---

# Plan Trusted Field Adapters from Registered Protobuf Options

## Idea and Problem

Read a registered custom field option into a bounded immutable adapter plan without turning descriptor metadata into executable plugin loading.

A field descriptor exposes its options through the generated default-pool
`FieldOptions` class, so a private-pool extension is otherwise left as an
unknown wire field. Serializing those actual options and reparsing them with
the private pool's dynamic `FieldOptions` class makes `HasExtension` reliable:
an absent option is skipped, while a present option containing an empty string
is detected and rejected.

## When to Use

Use this integration after a trusted descriptor graph, including
`google/protobuf/descriptor.proto` and one proto2 custom option file, has
already been registered in an isolated `DescriptorPool`. The caller should
own an immutable allowlist of adapter keys and resolve those keys against a
separate closed adapter registry only after planning succeeds.

Do not use descriptor options as Python import paths or as authority to run
code. Descriptor acquisition, signature verification, pool construction,
adapter loading, and adapter execution belong outside this function.

## Implementation

```python
import re
from dataclasses import dataclass

from google.protobuf import descriptor, descriptor_pool, message_factory
from google.protobuf.message import DecodeError, EncodeError

_MAX_FIELDS = 128
_MAX_FULL_NAME_BYTES = 256
_MAX_FIELD_NAME_BYTES = 64
_MAX_OPTION_BYTES_PER_FIELD = 1_024
_MAX_TOTAL_OPTION_BYTES = 16_384
_MAX_ALLOWED_KEYS = 64
_MAX_ADAPTER_KEY_BYTES = 64
_MAX_PLAN_ENTRIES = 64
_IDENTIFIER_TEXT = r"[A-Za-z_][A-Za-z0-9_]{0,63}"
_IDENTIFIER = re.compile(_IDENTIFIER_TEXT, re.ASCII)
_QUALIFIED_NAME = re.compile(
    rf"{_IDENTIFIER_TEXT}(?:\.{_IDENTIFIER_TEXT})*",
    re.ASCII,
)
_ADAPTER_KEY = re.compile(r"[a-z][a-z0-9_-]{0,63}", re.ASCII)
_FIELD_OPTIONS_NAME = "google.protobuf.FieldOptions"


class AdapterPlanError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class PlannedFieldAdapter:
    field_name: str
    field_number: int
    adapter_key: str


def _checked_full_name(value: object, label: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{label} must be an exact string")
    if len(value) > _MAX_FULL_NAME_BYTES or _QUALIFIED_NAME.fullmatch(value) is None:
        raise AdapterPlanError(f"{label} is invalid or too large")
    return value


def _checked_field_name(value: object) -> str:
    if type(value) is not str:
        raise TypeError("field names must be exact strings")
    if len(value) > _MAX_FIELD_NAME_BYTES or _IDENTIFIER.fullmatch(value) is None:
        raise AdapterPlanError("a field name is invalid or too large")
    return value


def _checked_adapter_key(value: object) -> str:
    if type(value) is not str:
        raise TypeError("adapter keys must be exact strings")
    if not value:
        raise AdapterPlanError("adapter keys must not be empty")
    if len(value) > _MAX_ADAPTER_KEY_BYTES or _ADAPTER_KEY.fullmatch(value) is None:
        raise AdapterPlanError("an adapter key is invalid or too large")
    return value


def _checked_allowlist(value: object) -> frozenset[str]:
    if type(value) is not frozenset:
        raise TypeError("allowed adapter keys must be an exact frozenset")
    if not 1 <= len(value) <= _MAX_ALLOWED_KEYS:
        raise AdapterPlanError("allowed adapter key count is outside 1..64")
    if any(type(key) is not str for key in value):
        raise TypeError("allowed adapter keys must be exact strings")
    for key in sorted(value):
        _checked_adapter_key(key)
    return value


def plan_field_adapters(
    pool: descriptor_pool.DescriptorPool,
    message_descriptor: descriptor.Descriptor,
    option_full_name: str,
    allowed_adapter_keys: frozenset[str],
) -> tuple[PlannedFieldAdapter, ...]:
    if not isinstance(message_descriptor, descriptor.Descriptor):
        raise TypeError("message_descriptor must be a Protobuf Descriptor")

    message_name = _checked_full_name(message_descriptor.full_name, "message name")
    option_name = _checked_full_name(option_full_name, "option name")
    allowed_keys = _checked_allowlist(allowed_adapter_keys)
    if message_descriptor.file.pool is not pool:
        raise AdapterPlanError("message descriptor and pool do not match")
    try:
        registered_message = pool.FindMessageTypeByName(message_name)
    except (KeyError, TypeError, ValueError) as error:
        raise AdapterPlanError("message descriptor is not registered in the pool") from error
    if registered_message is not message_descriptor:
        raise AdapterPlanError("message descriptor identity does not match the pool")

    fields = message_descriptor.fields
    if len(fields) > _MAX_FIELDS:
        raise AdapterPlanError("message field count exceeds 128")

    try:
        field_options_descriptor = pool.FindMessageTypeByName(_FIELD_OPTIONS_NAME)
        option_extension = pool.FindExtensionByName(option_name)
    except (KeyError, TypeError, ValueError) as error:
        raise AdapterPlanError("required option descriptors are not registered") from error
    if field_options_descriptor.file.pool is not pool or option_extension.file.pool is not pool:
        raise AdapterPlanError("option descriptors do not belong to the supplied pool")
    if (
        not option_extension.is_extension
        or option_extension.containing_type is not field_options_descriptor
    ):
        raise AdapterPlanError("option must extend google.protobuf.FieldOptions")
    if option_extension.type != descriptor.FieldDescriptor.TYPE_STRING:
        raise AdapterPlanError("option must have string type")
    if option_extension.is_repeated or option_extension.is_required:
        raise AdapterPlanError("option must be optional and singular")

    try:
        field_options_type = message_factory.GetMessageClass(field_options_descriptor)
    except (TypeError, ValueError) as error:
        raise AdapterPlanError("matching dynamic FieldOptions class is unavailable") from error
    if field_options_type.DESCRIPTOR is not field_options_descriptor:
        raise AdapterPlanError("dynamic FieldOptions class does not match the pool")

    planned: list[PlannedFieldAdapter] = []
    total_option_bytes = 0
    for field in fields:
        if not isinstance(field, descriptor.FieldDescriptor):
            raise TypeError("message fields must be Protobuf FieldDescriptors")
        if field.containing_type is not message_descriptor or field.file.pool is not pool:
            raise AdapterPlanError("a field descriptor does not match the message pool")
        field_name = _checked_field_name(field.name)
        try:
            raw_options = field.GetOptions().SerializeToString(deterministic=True)
        except (EncodeError, TypeError, ValueError) as error:
            raise AdapterPlanError("field options could not be serialized") from error
        if len(raw_options) > _MAX_OPTION_BYTES_PER_FIELD:
            raise AdapterPlanError("one field option payload exceeds 1024 bytes")
        total_option_bytes += len(raw_options)
        if total_option_bytes > _MAX_TOTAL_OPTION_BYTES:
            raise AdapterPlanError("total field option bytes exceed 16384")

        try:
            parsed_options = field_options_type.FromString(raw_options)
        except DecodeError as error:
            raise AdapterPlanError("field option bytes are malformed") from error
        try:
            present = parsed_options.HasExtension(option_extension)
        except (KeyError, TypeError, ValueError) as error:
            raise AdapterPlanError(
                "field option does not match its registered extension"
            ) from error
        if not present:
            continue

        key = _checked_adapter_key(parsed_options.Extensions[option_extension])
        if key not in allowed_keys:
            raise AdapterPlanError("field option names an unknown adapter key")
        if len(planned) == _MAX_PLAN_ENTRIES:
            raise AdapterPlanError("adapter plan exceeds 64 entries")
        planned.append(PlannedFieldAdapter(field_name, field.number, key))

    return tuple(planned)
```

## Example

```python
from google.protobuf import descriptor_pb2


pool = descriptor_pool.DescriptorPool()
pool.AddSerializedFile(descriptor_pb2.DESCRIPTOR.serialized_pb)

option_file = descriptor_pb2.FileDescriptorProto(
    name="public/adapter_options.proto",
    package="public_options",
    dependency=("google/protobuf/descriptor.proto",),
    syntax="proto2",
)
for name, number, field_type, label, extendee in (
    (
        "adapter_key",
        50_001,
        descriptor_pb2.FieldDescriptorProto.TYPE_STRING,
        descriptor_pb2.FieldDescriptorProto.LABEL_OPTIONAL,
        ".google.protobuf.FieldOptions",
    ),
    (
        "numeric_key",
        50_002,
        descriptor_pb2.FieldDescriptorProto.TYPE_INT32,
        descriptor_pb2.FieldDescriptorProto.LABEL_OPTIONAL,
        ".google.protobuf.FieldOptions",
    ),
    (
        "repeated_key",
        50_003,
        descriptor_pb2.FieldDescriptorProto.TYPE_STRING,
        descriptor_pb2.FieldDescriptorProto.LABEL_REPEATED,
        ".google.protobuf.FieldOptions",
    ),
    (
        "message_key",
        50_004,
        descriptor_pb2.FieldDescriptorProto.TYPE_STRING,
        descriptor_pb2.FieldDescriptorProto.LABEL_OPTIONAL,
        ".google.protobuf.MessageOptions",
    ),
):
    option_file.extension.add(
        name=name,
        number=number,
        type=field_type,
        label=label,
        extendee=extendee,
    )
pool.AddSerializedFile(option_file.SerializeToString(deterministic=True))

field_options_descriptor = pool.FindMessageTypeByName("google.protobuf.FieldOptions")
dynamic_field_options = message_factory.GetMessageClass(field_options_descriptor)
adapter_key = pool.FindExtensionByName("public_options.adapter_key")


def option_bytes(key: str) -> bytes:
    options = dynamic_field_options()
    options.Extensions[adapter_key] = key
    return options.SerializeToString(deterministic=True)


model_file = descriptor_pb2.FileDescriptorProto(
    name="public/model.proto",
    package="public_model",
    dependency=(option_file.name,),
    syntax="proto2",
)

valid_message = model_file.message_type.add(name="ValidRecord")
valid_message.field.add(
    name="source",
    number=1,
    label=descriptor_pb2.FieldDescriptorProto.LABEL_OPTIONAL,
    type=descriptor_pb2.FieldDescriptorProto.TYPE_STRING,
)
for name, number, key in (("email", 2, "trim"), ("alias", 3, "lower")):
    field = valid_message.field.add(
        name=name,
        number=number,
        label=descriptor_pb2.FieldDescriptorProto.LABEL_OPTIONAL,
        type=descriptor_pb2.FieldDescriptorProto.TYPE_STRING,
    )
    field.options.MergeFromString(option_bytes(key))

for message_name, key in (("EmptyRecord", ""), ("UnknownRecord", "redact")):
    message = model_file.message_type.add(name=message_name)
    field = message.field.add(
        name="value",
        number=1,
        label=descriptor_pb2.FieldDescriptorProto.LABEL_OPTIONAL,
        type=descriptor_pb2.FieldDescriptorProto.TYPE_STRING,
    )
    field.options.MergeFromString(option_bytes(key))

pool.AddSerializedFile(model_file.SerializeToString(deterministic=True))
allowed = frozenset({"lower", "trim"})
valid_descriptor = pool.FindMessageTypeByName("public_model.ValidRecord")
plan = plan_field_adapters(
    pool,
    valid_descriptor,
    "public_options.adapter_key",
    allowed,
)


def rejected(message_name: str, option_name: str = "public_options.adapter_key") -> bool:
    try:
        plan_field_adapters(
            pool,
            pool.FindMessageTypeByName(message_name),
            option_name,
            allowed,
        )
    except AdapterPlanError:
        return True
    return False


wrong_pool = descriptor_pool.DescriptorPool()
try:
    plan_field_adapters(
        wrong_pool,
        valid_descriptor,
        "public_options.adapter_key",
        allowed,
    )
except AdapterPlanError:
    mismatch_rejected = True
else:
    mismatch_rejected = False

assert (
    plan,
    rejected("public_model.EmptyRecord"),
    rejected("public_model.UnknownRecord"),
    mismatch_rejected,
    rejected("public_model.ValidRecord", "public_options.numeric_key"),
    rejected("public_model.ValidRecord", "public_options.repeated_key"),
    rejected("public_model.ValidRecord", "public_options.message_key"),
) == (
    (
        PlannedFieldAdapter("email", 2, "trim"),
        PlannedFieldAdapter("alias", 3, "lower"),
    ),
    True,
    True,
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

The function performs `O(F + B)` work for fields `F` and serialized option
bytes `B`, within ceilings of 128 fields, 1,024 option bytes per field, 16,384
option bytes total, 64 allowlisted keys, and 64 planned entries. Message,
field, option, and adapter names also have explicit ASCII and byte bounds.

The returned tuple and frozen records contain only names, numbers, and keys;
they neither retain mutable descriptors nor resolve or invoke adapters. An
owner can therefore validate a complete plan before a separate execution
phase maps keys through its trusted registry. A failure after earlier fields
have been examined discards the local list, so no partial plan escapes.

This function checks pool identity but cannot prove that the caller has kept
the supplied pool private or stopped registering more files after the call.
It reads but does not mutate descriptors, never consults the default pool, and
does not load schema files. Standard or unrelated custom `FieldOptions` are
preserved by the wire round trip but otherwise ignored.

## Related Snippets

<!-- catalog:related:start -->
- [Load a Bounded Protobuf Descriptor Set in Dependency Order](load-a-bounded-protobuf-descriptor-set-in-dependency-order.md)
- [Resolve Bounded Configuration Through Dependent Adapters](resolve-bounded-configuration-through-dependent-adapters.md)
- [Dispatch Named Strategies with an Explicit Function Mapping](../python-language/dispatch-named-strategies-with-an-explicit-function-mapping.md)
<!-- catalog:related:end -->

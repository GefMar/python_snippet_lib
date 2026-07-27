---
title: "Load a Bounded Protobuf Descriptor Set in Dependency Order"
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
  - ../algorithms-data-structures/resolve-stable-ordering-constraints-with-topological-sort.md
  - build-a-deterministic-size-capped-ustar-archive-from-bytes.md
---

# Load a Bounded Protobuf Descriptor Set in Dependency Order

## Idea and Problem

Validate one closed, bounded Protobuf descriptor set before loading its files into a fresh private pool in stable dependency order.

Descriptor files can appear before their imports in the serialized set, but a
pool needs each imported file first. Complete graph validation followed by a
stable Kahn ordering makes registration deterministic, rejects every cycle,
and prevents any partially loaded pool from escaping after a failure.

## When to Use

Use this integration for trusted descriptor-set bytes whose complete import
closure is included in the same payload. The caller should need descriptor
metadata and isolated symbol lookup without modifying a process-wide pool.
File names and imports must follow the deliberately narrow ASCII grammar used
here.

Keep trust decisions and signature checks outside this function. Fetching
remote schemas, importing application-generated schema modules, constructing
message classes, parsing message payloads, and converting messages through
binary alternatives such as JSON or text are separate concerns.

## Implementation

```python
import heapq
import re
from dataclasses import dataclass

from google.protobuf import descriptor_pb2, descriptor_pool
from google.protobuf.message import DecodeError


_MAX_DESCRIPTOR_SET_BYTES = 65_536
_MAX_FILES = 32
_MAX_IMPORTS_PER_FILE = 16
_MAX_IMPORT_EDGES = 256
_FILE_COMPONENT = re.compile(
    r"[a-z][a-z0-9_-]{0,31}(?:\.[a-z0-9][a-z0-9_-]{0,15})?",
    re.ASCII,
)


class DescriptorSetError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class LoadedDescriptorSet:
    pool: descriptor_pool.DescriptorPool
    file_order: tuple[str, ...]


def _validated_file_name(value: object) -> str:
    if type(value) is not str:
        raise TypeError("descriptor file names must be exact strings")
    if not 1 <= len(value) <= 128:
        raise DescriptorSetError("descriptor file name length is invalid")
    if any(
        _FILE_COMPONENT.fullmatch(component) is None
        for component in value.split("/")
    ):
        raise DescriptorSetError("descriptor file name is invalid")
    return value


def _validated_file_order(
    descriptor_set: descriptor_pb2.FileDescriptorSet,
) -> tuple[
    tuple[str, ...],
    dict[str, descriptor_pb2.FileDescriptorProto],
]:
    if not 1 <= len(descriptor_set.file) <= _MAX_FILES:
        raise DescriptorSetError(
            "descriptor set must contain between 1 and 32 files"
        )

    ordered_names: list[str] = []
    files_by_name: dict[str, descriptor_pb2.FileDescriptorProto] = {}
    dependencies_by_name: dict[str, tuple[str, ...]] = {}
    edge_count = 0

    for file_proto in descriptor_set.file:
        name = _validated_file_name(file_proto.name)
        if name in files_by_name:
            raise DescriptorSetError("descriptor file names must be unique")
        if len(file_proto.dependency) > _MAX_IMPORTS_PER_FILE:
            raise DescriptorSetError(
                "a descriptor file has more than 16 imports"
            )

        dependencies: list[str] = []
        seen_dependencies: set[str] = set()
        for raw_dependency in file_proto.dependency:
            dependency = _validated_file_name(raw_dependency)
            if dependency == name:
                raise DescriptorSetError(
                    "a descriptor file cannot import itself"
                )
            if dependency in seen_dependencies:
                raise DescriptorSetError(
                    "imports within a descriptor file must be unique"
                )
            seen_dependencies.add(dependency)
            dependencies.append(dependency)

        edge_count += len(dependencies)
        if edge_count > _MAX_IMPORT_EDGES:
            raise DescriptorSetError(
                "descriptor set has more than 256 import edges"
            )
        ordered_names.append(name)
        files_by_name[name] = file_proto
        dependencies_by_name[name] = tuple(dependencies)

    declared_names = frozenset(ordered_names)
    for dependencies in dependencies_by_name.values():
        if any(dependency not in declared_names for dependency in dependencies):
            raise DescriptorSetError(
                "descriptor set contains a missing import"
            )

    positions = {name: index for index, name in enumerate(ordered_names)}
    dependents: dict[str, list[str]] = {
        name: [] for name in ordered_names
    }
    remaining = {
        name: len(dependencies_by_name[name])
        for name in ordered_names
    }
    for name in ordered_names:
        for dependency in dependencies_by_name[name]:
            dependents[dependency].append(name)

    available = [
        positions[name]
        for name in ordered_names
        if remaining[name] == 0
    ]
    heapq.heapify(available)
    file_order: list[str] = []
    while available:
        name = ordered_names[heapq.heappop(available)]
        file_order.append(name)
        for dependent in dependents[name]:
            remaining[dependent] -= 1
            if remaining[dependent] == 0:
                heapq.heappush(available, positions[dependent])

    if len(file_order) != len(ordered_names):
        raise DescriptorSetError(
            "descriptor imports must not contain a cycle"
        )
    return tuple(file_order), files_by_name


def _build_private_pool(
    file_order: tuple[str, ...],
    files_by_name: dict[str, descriptor_pb2.FileDescriptorProto],
) -> descriptor_pool.DescriptorPool | None:
    pool = descriptor_pool.DescriptorPool()
    try:
        for name in file_order:
            file_proto = files_by_name[name]
            pool.AddSerializedFile(
                file_proto.SerializeToString(deterministic=True)
            )
    except (TypeError, ValueError):
        return None
    return pool


def load_bounded_descriptor_set(payload: bytes) -> LoadedDescriptorSet:
    if type(payload) is not bytes:
        raise TypeError("payload must be exact bytes")
    if not 1 <= len(payload) <= _MAX_DESCRIPTOR_SET_BYTES:
        raise DescriptorSetError("descriptor set byte length is invalid")

    try:
        descriptor_set = descriptor_pb2.FileDescriptorSet.FromString(payload)
    except DecodeError as error:
        raise DescriptorSetError(
            "descriptor set bytes are malformed"
        ) from error

    file_order, files_by_name = _validated_file_order(descriptor_set)
    pool = _build_private_pool(file_order, files_by_name)
    if pool is None:
        raise DescriptorSetError(
            "validated descriptor files could not be loaded"
        )
    return LoadedDescriptorSet(pool=pool, file_order=file_order)
```

## Example

```python
def serialized_files(
    *files: descriptor_pb2.FileDescriptorProto,
) -> bytes:
    descriptor_set = descriptor_pb2.FileDescriptorSet()
    descriptor_set.file.extend(files)
    return descriptor_set.SerializeToString(deterministic=True)


base = descriptor_pb2.FileDescriptorProto(
    name="neutral/base.proto",
    package="neutral",
    syntax="proto3",
)
base.message_type.add(name="BaseRecord")

feature = descriptor_pb2.FileDescriptorProto(
    name="neutral/feature.proto",
    package="neutral",
    dependency=(base.name,),
    syntax="proto3",
)
feature_message = feature.message_type.add(name="FeatureRecord")
feature_message.field.add(
    name="base",
    number=1,
    label=descriptor_pb2.FieldDescriptorProto.LABEL_OPTIONAL,
    type=descriptor_pb2.FieldDescriptorProto.TYPE_MESSAGE,
    type_name=".neutral.BaseRecord",
)

reversed_payload = serialized_files(feature, base)
payload_before = bytes(reversed_payload)
loaded = load_bounded_descriptor_set(reversed_payload)
import_cycle = serialized_files(
    descriptor_pb2.FileDescriptorProto(
        name="cycle/left.proto",
        dependency=("cycle/right.proto",),
    ),
    descriptor_pb2.FileDescriptorProto(
        name="cycle/right.proto",
        dependency=("cycle/left.proto",),
    ),
)
try:
    load_bounded_descriptor_set(import_cycle)
except DescriptorSetError:
    cycle_rejected = True
else:
    cycle_rejected = False

assert (
    loaded.file_order,
    loaded.pool.FindMessageTypeByName(
        "neutral.FeatureRecord"
    ).fields_by_name["base"].message_type.full_name,
    reversed_payload == payload_before,
    cycle_rejected,
) == (
    ("neutral/base.proto", "neutral/feature.proto"),
    "neutral.BaseRecord",
    True,
    True,
)
```

## Trade-offs and Limitations

Parsing, validation, and loading use `O(B + (V + E) log V)` time for payload
bytes `B`, files `V`, and imports `E`, within the 65,536-byte, 32-file,
16-import-per-file, and 256-edge ceilings. The declaration-index heap makes
ordering stable whenever more than one file is ready. A reversed importing
file still follows its prerequisite.

The frozen `LoadedDescriptorSet` prevents rebinding its attributes, but the
contained `DescriptorPool` is mutable and can accept more files. Treat the
pool as read-only after handoff, or keep it behind an owner that controls
further registration. Each successful call creates a distinct private pool;
no process-global or default pool is read or modified.

Graph validation cannot prove all descriptor semantics. The pinned Protobuf
runtime still checks symbols, fields, and type references while files are
added. If that phase fails, the partial private pool is discarded and the
caller receives one short value-free error. The integration does not fetch or
persist schemas, access generated application schemas, construct message
classes, parse payloads, or perform `Any`, JSON, or text conversion.

## Related Snippets

<!-- catalog:related:start -->
- [Resolve Stable Ordering Constraints with Topological Sort](../algorithms-data-structures/resolve-stable-ordering-constraints-with-topological-sort.md)
- [Build a Deterministic Size-Capped USTAR Archive from Bytes](build-a-deterministic-size-capped-ustar-archive-from-bytes.md)
<!-- catalog:related:end -->

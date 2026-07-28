---
title: "Project a Bounded Document Snapshot into Collision-Audited Secondary Index Rows"
snippet_type: recipe
use_cases:
  - data-transformation
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - project-bounded-records-into-multiple-closed-output-schemas.md
  - ../configuration-serialization/normalize-a-bounded-json-copy-before-standard-schema-validation.md
  - ../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md
---

# Project a Bounded Document Snapshot into Collision-Audited Secondary Index Rows

## Idea and Problem

Validate and freeze an ordered document snapshot while projecting each alias into a separate immutable secondary-index row.

Primary identifiers are unique, aliases are unique both within one document and
across the complete snapshot, and input order determines both output orders.
Validation covers the complete batch before either tuple is returned, so a
late collision or invalid content produces no partial projection.

## When to Use

Use this recipe at an in-memory boundary where each bounded document has one
conservative ASCII identifier, an ordered alias tuple, and an already decoded
JSON-compatible value tree. The accepted tree uses only exact built-in JSON
scalar and container types. Signed 64-bit integers, finite floats, valid UTF-8
text, bounded nesting, and bounded value occurrences make the accepted subset
explicit.

The function is a snapshot projector, not a JSON decoder or storage adapter.
Keep ORM models, SQL, tables, insert or delete operations, transactions, and
persistence outside this boundary. A returned snapshot says nothing about
whether any external system accepted or stored it.

## Implementation

```python
import math
import re
from collections.abc import Mapping
from dataclasses import dataclass
from types import MappingProxyType

type JSONScalar = None | bool | int | float | str
type FrozenJSON = JSONScalar | tuple[FrozenJSON, ...] | Mapping[str, FrozenJSON]

_IDENTIFIER = re.compile(
    r"[a-z][a-z0-9]*(?:-[a-z0-9]+){0,7}\Z",
    re.ASCII,
).fullmatch
_MAX_DOCUMENTS = 256
_MAX_ALIASES_PER_DOCUMENT = 64
_MAX_TOTAL_ALIASES = 4_096
_MAX_IDENTIFIER_BYTES = 64
_MAX_TOTAL_IDENTIFIER_BYTES = 128 * 1_024
_MAX_CONTAINER_ITEMS = 256
_MAX_JSON_DEPTH = 16
_MAX_JSON_NODES = 16_384
_MAX_TEXT_VALUE_BYTES = 64 * 1_024
_MAX_JSON_TEXT_BYTES = 1 * 1_024 * 1_024
_MIN_INTEGER = -(2**63)
_MAX_INTEGER = 2**63 - 1


@dataclass(frozen=True, slots=True)
class DocumentInput:
    document_id: str
    aliases: tuple[str, ...]
    content: object


@dataclass(frozen=True, slots=True)
class FrozenDocument:
    document_id: str
    content: FrozenJSON


@dataclass(frozen=True, slots=True)
class SecondaryIndexRow:
    alias: str
    document_id: str


@dataclass(frozen=True, slots=True)
class DocumentIndexSnapshot:
    primary_records: tuple[FrozenDocument, ...]
    secondary_rows: tuple[SecondaryIndexRow, ...]


@dataclass(slots=True)
class _JSONBudget:
    nodes: int = 0
    text_bytes: int = 0


def _validated_identifier(value: object, *, role: str) -> str:
    if (
        type(value) is not str
        or not 1 <= len(value) <= _MAX_IDENTIFIER_BYTES
        or _IDENTIFIER(value) is None
    ):
        raise ValueError(f"{role} must be a conservative ASCII identifier")
    return value


def _validated_text(value: str, *, budget: _JSONBudget, role: str) -> str:
    if len(value) > _MAX_TEXT_VALUE_BYTES:
        raise ValueError(f"{role} exceeds the per-value UTF-8 byte limit")
    try:
        size = len(value.encode("utf-8"))
    except UnicodeEncodeError as error:
        raise ValueError(f"{role} must be valid UTF-8 text") from error
    if size > _MAX_TEXT_VALUE_BYTES:
        raise ValueError(f"{role} exceeds the per-value UTF-8 byte limit")
    budget.text_bytes += size
    if budget.text_bytes > _MAX_JSON_TEXT_BYTES:
        raise ValueError("document content exceeds the aggregate UTF-8 byte limit")
    return value


def _freeze_json(
    value: object,
    *,
    depth: int,
    budget: _JSONBudget,
    seen_containers: set[int],
    active_containers: set[int],
) -> FrozenJSON:
    budget.nodes += 1
    if budget.nodes > _MAX_JSON_NODES:
        raise ValueError("document content exceeds the value-occurrence limit")
    if depth > _MAX_JSON_DEPTH:
        raise ValueError("document content exceeds the nesting-depth limit")

    if value is None or type(value) is bool:
        return value
    if type(value) is int:
        if not _MIN_INTEGER <= value <= _MAX_INTEGER:
            raise ValueError("JSON integers must fit the signed 64-bit range")
        return value
    if type(value) is float:
        if not math.isfinite(value):
            raise ValueError("JSON floats must be finite")
        return value
    if type(value) is str:
        return _validated_text(value, budget=budget, role="a JSON string")
    if type(value) not in (list, dict):
        raise TypeError("content must contain only exact JSON-compatible built-in types")
    if len(value) > _MAX_CONTAINER_ITEMS:
        raise ValueError("a JSON container exceeds the entry limit")

    identity = id(value)
    if identity in active_containers:
        raise ValueError("cyclic JSON containers are not supported")
    if identity in seen_containers:
        raise ValueError("a mutable JSON container is shared by multiple paths")
    seen_containers.add(identity)
    active_containers.add(identity)
    try:
        if type(value) is list:
            return tuple(
                _freeze_json(
                    child,
                    depth=depth + 1,
                    budget=budget,
                    seen_containers=seen_containers,
                    active_containers=active_containers,
                )
                for child in value
            )

        frozen: dict[str, FrozenJSON] = {}
        for key, child in value.items():
            if type(key) is not str:
                raise TypeError("JSON object keys must be exact strings")
            checked_key = _validated_text(key, budget=budget, role="a JSON object key")
            frozen[checked_key] = _freeze_json(
                child,
                depth=depth + 1,
                budget=budget,
                seen_containers=seen_containers,
                active_containers=active_containers,
            )
        return MappingProxyType(frozen)
    finally:
        active_containers.remove(identity)


def project_document_snapshot(documents: tuple[DocumentInput, ...]) -> DocumentIndexSnapshot:
    if type(documents) is not tuple:
        raise TypeError("documents must be an exact tuple")
    if len(documents) > _MAX_DOCUMENTS:
        raise ValueError("document count exceeds the supported limit")

    document_ids: set[str] = set()
    global_aliases: set[str] = set()
    validated_documents: list[DocumentInput] = []
    alias_count = 0
    identifier_bytes = 0
    for document in documents:
        if type(document) is not DocumentInput:
            raise TypeError("documents must contain exact DocumentInput values")
        document_id = _validated_identifier(document.document_id, role="a document ID")
        if document_id in document_ids:
            raise ValueError("document IDs must be unique")
        document_ids.add(document_id)
        identifier_bytes += len(document_id)

        if type(document.aliases) is not tuple:
            raise TypeError("document aliases must be an exact tuple")
        if len(document.aliases) > _MAX_ALIASES_PER_DOCUMENT:
            raise ValueError("a document exceeds the alias-count limit")
        alias_count += len(document.aliases)
        if alias_count > _MAX_TOTAL_ALIASES:
            raise ValueError("the snapshot exceeds the total alias-count limit")

        local_aliases: set[str] = set()
        for raw_alias in document.aliases:
            alias = _validated_identifier(raw_alias, role="an alias")
            if alias in local_aliases:
                raise ValueError("aliases within a document must be unique")
            if alias in global_aliases:
                raise ValueError("aliases across documents must be unique")
            local_aliases.add(alias)
            global_aliases.add(alias)
            identifier_bytes += len(alias)
        validated_documents.append(document)

    if identifier_bytes > _MAX_TOTAL_IDENTIFIER_BYTES:
        raise ValueError("document IDs and aliases exceed the aggregate byte limit")

    budget = _JSONBudget()
    seen_containers: set[int] = set()
    frozen_contents: list[FrozenJSON] = []
    for document in validated_documents:
        frozen_contents.append(
            _freeze_json(
                document.content,
                depth=0,
                budget=budget,
                seen_containers=seen_containers,
                active_containers=set(),
            )
        )

    primary_records = tuple(
        FrozenDocument(document.document_id, frozen_content)
        for document, frozen_content in zip(
            validated_documents,
            frozen_contents,
            strict=True,
        )
    )
    secondary_rows = tuple(
        SecondaryIndexRow(alias, document.document_id)
        for document in validated_documents
        for alias in document.aliases
    )

    return DocumentIndexSnapshot(primary_records, secondary_rows)
```

## Example

```python
content = {
    "title": "Cedar",
    "measurements": [2, 3.5],
    "enabled": True,
}
documents = (
    DocumentInput("document-alpha", ("cedar", "first"), content),
    DocumentInput("document-beta", ("blue",), {"title": "Blue", "note": None}),
)

snapshot = project_document_snapshot(documents)
content["measurements"].append(99)

try:
    snapshot.primary_records[0].content["title"] = "changed"
except TypeError:
    assignment_rejected = True
else:
    assignment_rejected = False

shared: list[object] = []
try:
    project_document_snapshot(
        (DocumentInput("document-shared", (), {"left": shared, "right": shared}),)
    )
except ValueError:
    shared_container_rejected = True
else:
    shared_container_rejected = False

empty = project_document_snapshot(())
first_content = snapshot.primary_records[0].content
assert (
    tuple((row.alias, row.document_id) for row in snapshot.secondary_rows)
    == (
        ("cedar", "document-alpha"),
        ("first", "document-alpha"),
        ("blue", "document-beta"),
    )
    and first_content["measurements"] == (2, 3.5)
    and assignment_rejected
    and shared_container_rejected
    and empty == DocumentIndexSnapshot((), ())
)
```

## Trade-offs and Limitations

For `d` documents, `a` aliases, `n` JSON value occurrences, and `b` validated
text bytes, validation and projection take `O(d + a + n + b)` time. The result,
collision sets, container-identity set, and recursive traversal use `O(d + a +
n)` memory, with at most `_MAX_JSON_DEPTH + 1` active calls. Every identifier
has a 64-byte ASCII cap, and their byte budget is cumulative across document
IDs and aliases. Node and JSON text-byte budgets are also cumulative across all
document contents; object keys count toward the JSON byte budget but not the
value-occurrence budget.

Exact `dict` and `list` inputs become private read-only mappings and tuples.
Repeated references to the same mutable container are rejected even across
documents, while repeated immutable scalars are safe. Mapping insertion order
is preserved. The function does not normalize identifiers or text, validate a
domain schema, decode JSON, or protect against concurrent input mutation. It
also performs no ORM, SQL, table, insert, delete, transaction, or persistence
work.

## Related Snippets

<!-- catalog:related:start -->
- [Project Bounded Records into Multiple Closed Output Schemas](project-bounded-records-into-multiple-closed-output-schemas.md)
- [Normalize a Bounded JSON Copy Before Standard Schema Validation](../configuration-serialization/normalize-a-bounded-json-copy-before-standard-schema-validation.md)
- [Make a Defensive Copy at a Mutable Input Boundary](../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md)
<!-- catalog:related:end -->

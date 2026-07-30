---
title: "Apply a Bounded RFC 6902 JSON Patch Atomically to a Detached JSON Tree"
snippet_type: pattern
use_cases:
  - configuration
  - data-transformation
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - apply-a-bounded-rfc-7396-json-merge-patch-without-mutating-inputs.md
  - resolve-a-bounded-plain-json-pointer.md
  - parse-bounded-utf-8-json-without-duplicate-object-names-or-non-finite-numbers.md
---

# Apply a Bounded RFC 6902 JSON Patch Atomically to a Detached JSON Tree

## Idea and Problem

Apply a bounded RFC 6902 JSON Patch to an already parsed JSON tree while returning a fully detached result or leaving both caller-owned inputs untouched.

Under [RFC 6902](https://www.rfc-editor.org/rfc/rfc6902.html), operations run sequentially: each one observes the result of its predecessors, and the first error aborts the patch. The implementation supports `add`, `remove`, `replace`, `move`, `copy`, and `test`; ignores operation-object members that the selected operation does not define; and evaluates paths as [RFC 6901 JSON Pointers](https://www.rfc-editor.org/rfc/rfc6901.html).

This is a deliberately bounded profile for Python trees, not a permissive adapter. The document, patch, and intermediate results must contain only exact JSON-shaped built-ins. Container cycles and repeated container identities within either individual input tree are rejected, integer and floating-point values are bounded, and every operation is followed by a fresh result-budget check. Values inserted from the patch are cloned, copied values are cloned, and moves transfer a subtree only after detaching it from the private working tree.

## When to Use

Use this pattern when a trust boundary has already decoded JSON and an application needs deterministic, in-memory patching without exposing either input to mutation. It is especially useful when a request can contain arbitrary pointer paths or repeated `copy` operations, because pointer, operation, depth, node, and text limits make the accepted workload explicit.

Choose the limits for the surrounding protocol. Authentication, authorization for individual paths, JSON decoding, schema validation, persistence, concurrent updates, and HTTP conditional requests remain separate concerns. If clients expect URI-fragment pointers or an unbounded/general Python object walker, this profile is intentionally too narrow.

## Implementation

```python
import math

type JsonValue = None | bool | int | float | str | list[JsonValue] | dict[str, JsonValue]

_MIN_INTEGER = -(2**63)
_MAX_INTEGER = 2**63 - 1
_MAX_DEPTH = 32
_MAX_NODES = 5_000
_MAX_TEXT_BYTES = 1 << 20
_MAX_OPERATIONS = 128
_MAX_POINTER_CHARACTERS = 4_096
_MAX_POINTER_BYTES = 16_384
_MAX_POINTER_TOKENS = 256


class JsonPatchError(ValueError):
    """The input is outside the profile or the patch cannot be applied."""


def _utf8_size(text: str, label: str) -> int:
    try:
        return len(text.encode("utf-8"))
    except UnicodeEncodeError as error:
        raise JsonPatchError(f"{label} contains a Unicode surrogate") from error


def _validate_json_tree(label: str, root: JsonValue) -> None:
    seen_containers: set[int] = set()
    stack: list[tuple[JsonValue, int]] = [(root, 0)]
    nodes = 0
    text_bytes = 0

    while stack:
        value, depth = stack.pop()
        nodes += 1
        if nodes > _MAX_NODES:
            raise JsonPatchError(f"{label} exceeds the node limit")
        if depth > _MAX_DEPTH:
            raise JsonPatchError(f"{label} exceeds the depth limit")

        value_type = type(value)
        if value_type is type(None) or value_type is bool:
            continue
        if value_type is int:
            if not _MIN_INTEGER <= value <= _MAX_INTEGER:
                raise JsonPatchError(f"{label} contains an out-of-range integer")
            continue
        if value_type is float:
            if not math.isfinite(value):
                raise JsonPatchError(f"{label} contains a non-finite number")
            continue
        if value_type is str:
            text_bytes += _utf8_size(value, label)
        elif value_type is list:
            identity = id(value)
            if identity in seen_containers:
                raise JsonPatchError(f"{label} contains a cycle or container alias")
            seen_containers.add(identity)
            stack.extend((item, depth + 1) for item in reversed(value))
        elif value_type is dict:
            identity = id(value)
            if identity in seen_containers:
                raise JsonPatchError(f"{label} contains a cycle or container alias")
            seen_containers.add(identity)
            for key, item in value.items():
                key_type = type(key)
                if key_type is not str:
                    raise JsonPatchError(f"{label} contains a non-string object name")
                text_bytes += _utf8_size(key, label)
                stack.append((item, depth + 1))
        else:
            raise JsonPatchError(f"{label} contains a non-JSON value")

        if text_bytes > _MAX_TEXT_BYTES:
            raise JsonPatchError(f"{label} exceeds the UTF-8 text limit")


def _clone_json(value: JsonValue) -> JsonValue:
    value_type = type(value)
    if value_type is list:
        return [_clone_json(item) for item in value]
    if value_type is dict:
        return {key: _clone_json(item) for key, item in value.items()}
    return value


def _decode_pointer(pointer: JsonValue, operation: int, member: str) -> tuple[str, ...]:
    pointer_type = type(pointer)
    if pointer_type is not str:
        raise JsonPatchError(f"operation {operation} {member} must be a string")
    if len(pointer) > _MAX_POINTER_CHARACTERS:
        raise JsonPatchError(f"operation {operation} {member} is too long")
    if _utf8_size(pointer, f"operation {operation} {member}") > _MAX_POINTER_BYTES:
        raise JsonPatchError(f"operation {operation} {member} is too large")
    if pointer == "":
        return ()
    if not pointer.startswith("/"):
        raise JsonPatchError(f"operation {operation} {member} is not a JSON Pointer")

    raw_tokens = pointer.split("/")[1:]
    if len(raw_tokens) > _MAX_POINTER_TOKENS:
        raise JsonPatchError(f"operation {operation} {member} has too many tokens")

    tokens: list[str] = []
    for raw_token in raw_tokens:
        decoded: list[str] = []
        position = 0
        while position < len(raw_token):
            character = raw_token[position]
            if character != "~":
                decoded.append(character)
                position += 1
                continue
            if position + 1 == len(raw_token):
                raise JsonPatchError(f"operation {operation} {member} has an invalid escape")
            escape = raw_token[position + 1]
            if escape == "0":
                decoded.append("~")
            elif escape == "1":
                decoded.append("/")
            else:
                raise JsonPatchError(f"operation {operation} {member} has an invalid escape")
            position += 2
        tokens.append("".join(decoded))
    return tuple(tokens)


def _array_index(
    token: str,
    length: int,
    *,
    allow_append: bool,
    operation: int,
    member: str,
) -> int:
    if token == "-":
        if allow_append:
            return length
        raise JsonPatchError(f"operation {operation} {member} uses '-' outside add's final token")
    if (
        not token
        or (len(token) > 1 and token.startswith("0"))
        or any(character < "0" or character > "9" for character in token)
    ):
        raise JsonPatchError(f"operation {operation} {member} has a non-canonical array index")

    upper_bound = length if allow_append else length - 1
    if upper_bound < 0:
        raise JsonPatchError(f"operation {operation} {member} index is out of bounds")
    bound_text = str(upper_bound)
    if len(token) > len(bound_text) or (len(token) == len(bound_text) and token > bound_text):
        raise JsonPatchError(f"operation {operation} {member} index is out of bounds")
    return int(token)


def _resolve(
    root: JsonValue,
    tokens: tuple[str, ...],
    operation: int,
    member: str,
) -> JsonValue:
    current = root
    for token in tokens:
        current_type = type(current)
        if current_type is dict:
            if token not in current:
                raise JsonPatchError(
                    f"operation {operation} {member} names a missing object member"
                )
            current = current[token]
        elif current_type is list:
            index = _array_index(
                token,
                len(current),
                allow_append=False,
                operation=operation,
                member=member,
            )
            current = current[index]
        else:
            raise JsonPatchError(f"operation {operation} {member} traverses through a scalar")
    return current


def _add(
    root: JsonValue,
    tokens: tuple[str, ...],
    value: JsonValue,
    operation: int,
) -> JsonValue:
    if not tokens:
        return value
    parent = _resolve(root, tokens[:-1], operation, "path")
    parent_type = type(parent)
    token = tokens[-1]
    if parent_type is dict:
        parent[token] = value
    elif parent_type is list:
        index = _array_index(
            token,
            len(parent),
            allow_append=True,
            operation=operation,
            member="path",
        )
        parent.insert(index, value)
    else:
        raise JsonPatchError(f"operation {operation} path parent is a scalar")
    return root


def _remove(
    root: JsonValue,
    tokens: tuple[str, ...],
    operation: int,
) -> tuple[JsonValue, JsonValue]:
    if not tokens:
        raise JsonPatchError(f"operation {operation} cannot remove the document root")
    parent = _resolve(root, tokens[:-1], operation, "path")
    parent_type = type(parent)
    token = tokens[-1]
    if parent_type is dict:
        if token not in parent:
            raise JsonPatchError(f"operation {operation} path does not exist")
        removed = parent.pop(token)
    elif parent_type is list:
        index = _array_index(
            token,
            len(parent),
            allow_append=False,
            operation=operation,
            member="path",
        )
        removed = parent.pop(index)
    else:
        raise JsonPatchError(f"operation {operation} path parent is a scalar")
    return root, removed


def _replace(
    root: JsonValue,
    tokens: tuple[str, ...],
    value: JsonValue,
    operation: int,
) -> JsonValue:
    if not tokens:
        return value
    parent = _resolve(root, tokens[:-1], operation, "path")
    parent_type = type(parent)
    token = tokens[-1]
    if parent_type is dict:
        if token not in parent:
            raise JsonPatchError(f"operation {operation} path does not exist")
        parent[token] = value
    elif parent_type is list:
        index = _array_index(
            token,
            len(parent),
            allow_append=False,
            operation=operation,
            member="path",
        )
        parent[index] = value
    else:
        raise JsonPatchError(f"operation {operation} path parent is a scalar")
    return root


def _json_equal(left: JsonValue, right: JsonValue) -> bool:
    left_type = type(left)
    right_type = type(right)
    number_types = (int, float)
    if left_type in number_types and right_type in number_types:
        return left == right
    if left_type is not right_type:
        return False
    if left_type is list:
        return len(left) == len(right) and all(
            _json_equal(left_item, right_item)
            for left_item, right_item in zip(left, right, strict=True)
        )
    if left_type is dict:
        return left.keys() == right.keys() and all(
            _json_equal(left[key], right[key]) for key in left
        )
    return left == right


def _required(
    operation_object: dict[str, JsonValue],
    member: str,
    operation: int,
) -> JsonValue:
    if member not in operation_object:
        raise JsonPatchError(f"operation {operation} is missing {member}")
    return operation_object[member]


def apply_json_patch(document: JsonValue, patch: JsonValue) -> JsonValue:
    """Return a detached result, or raise without mutating either input."""
    _validate_json_tree("document", document)
    _validate_json_tree("patch", patch)
    patch_type = type(patch)
    if patch_type is not list:
        raise JsonPatchError("patch must be an array")
    if len(patch) > _MAX_OPERATIONS:
        raise JsonPatchError("patch exceeds the operation limit")

    working = _clone_json(document)
    for operation_index, operation_object in enumerate(patch):
        operation_object_type = type(operation_object)
        if operation_object_type is not dict:
            raise JsonPatchError(f"operation {operation_index} must be an object")

        operation_name = _required(operation_object, "op", operation_index)
        operation_name_type = type(operation_name)
        if operation_name_type is not str:
            raise JsonPatchError(f"operation {operation_index} op must be a string")
        path = _decode_pointer(
            _required(operation_object, "path", operation_index),
            operation_index,
            "path",
        )

        if operation_name == "add":
            value = _clone_json(_required(operation_object, "value", operation_index))
            working = _add(working, path, value, operation_index)
        elif operation_name == "remove":
            working, _ = _remove(working, path, operation_index)
        elif operation_name == "replace":
            value = _clone_json(_required(operation_object, "value", operation_index))
            working = _replace(working, path, value, operation_index)
        elif operation_name == "copy":
            source = _decode_pointer(
                _required(operation_object, "from", operation_index),
                operation_index,
                "from",
            )
            value = _clone_json(_resolve(working, source, operation_index, "from"))
            working = _add(working, path, value, operation_index)
        elif operation_name == "move":
            source = _decode_pointer(
                _required(operation_object, "from", operation_index),
                operation_index,
                "from",
            )
            if source == path:
                _resolve(working, source, operation_index, "from")
            elif len(source) < len(path) and path[: len(source)] == source:
                raise JsonPatchError(
                    f"operation {operation_index} cannot move a value below itself"
                )
            else:
                working, value = _remove(working, source, operation_index)
                working = _add(working, path, value, operation_index)
        elif operation_name == "test":
            expected = _required(operation_object, "value", operation_index)
            actual = _resolve(working, path, operation_index, "path")
            if not _json_equal(actual, expected):
                raise JsonPatchError(f"operation {operation_index} test failed")
        else:
            raise JsonPatchError(f"operation {operation_index} has an unsupported op")

        _validate_json_tree(f"result after operation {operation_index}", working)
    return working
```

## Example

```python


document = {
    "title": "draft",
    "tags": ["python"],
    "meta": {"views": 1},
    "obsolete": True,
}
patch = [
    {"op": "test", "path": "/title", "value": "draft", "note": "ignored"},
    {"op": "add", "path": "/tags/-", "value": {"name": "json"}},
    {"op": "replace", "path": "/title", "value": "published"},
    {"op": "copy", "from": "/tags/1", "path": "/featured"},
    {"op": "move", "from": "/meta/views", "path": "/views"},
    {"op": "remove", "path": "/obsolete"},
]

result = apply_json_patch(document, patch)
assert result == {
    "title": "published",
    "tags": ["python", {"name": "json"}],
    "meta": {},
    "featured": {"name": "json"},
    "views": 1,
}
assert result is not document
assert result["tags"] is not document["tags"]
assert result["tags"][1] is not patch[1]["value"]
assert result["featured"] is not result["tags"][1]

result["tags"][1]["name"] = "changed"
assert patch[1]["value"] == {"name": "json"}
assert result["featured"] == {"name": "json"}

unchanged = {"value": 1}
try:
    apply_json_patch(
        unchanged,
        [
            {"op": "replace", "path": "/value", "value": 2},
            {"op": "test", "path": "/value", "value": 3},
        ],
    )
except JsonPatchError:
    pass
else:
    raise AssertionError("the failing patch unexpectedly succeeded")

assert unchanged == {"value": 1}
assert apply_json_patch({"x": 1}, [{"op": "move", "from": "", "path": ""}]) == {"x": 1}
```

## Trade-offs and Limitations

The profile uses signed 64-bit integers, finite floats, at most 128 operations, 256 pointer tokens, depth 32, 5,000 nodes, and 1 MiB of UTF-8 string and object-name data. Within each document or patch tree, a container identity may occur at only one path, so even an acyclic DAG inside one tree is rejected because JSON has no aliasing semantics. Cross-input or caller-external aliases are neither detected nor forbidden; they cannot propagate into the result because the document and inserted values are cloned. Array tokens are canonical unsigned decimal indexes; `-` is accepted only as the final `add` destination. Pointer escapes are decoded as `~0` and `~1`, while URI-fragment pointer syntax is not accepted.

JSON-aware `test` keeps booleans distinct from numbers, treats integers and floats as one numeric type, compares arrays in order, and compares object members without regard to order. This avoids Python's surprising `True == 1` result. Unknown operation members are structurally validated as part of the bounded JSON patch, but otherwise ignored.

The root policy is intentionally frozen: root `add`, `replace`, `copy`, and `test` work; moving the root to itself is a validated no-op; root `remove` is rejected; and moving the root below itself is rejected by the token-prefix rule. A non-root source may still be moved or copied to the root. This function provides in-memory atomicity through isolation, not transactionality for a database or HTTP resource.

Cloning is linear in the input size. Pointer traversal is linear in pointer depth, list insertion or removal may shift elements, and validating the complete result after every operation makes the conservative worst case proportional to the operation count times the bounded result size. The repeated validation is what prevents a chain of copies from escaping the node, depth, text, or alias budget.

## Related Snippets

<!-- catalog:related:start -->
- [Apply a Bounded RFC 7396 JSON Merge Patch without Mutating Inputs](apply-a-bounded-rfc-7396-json-merge-patch-without-mutating-inputs.md)
- [Resolve a Bounded Plain JSON Pointer](resolve-a-bounded-plain-json-pointer.md)
- [Parse Bounded UTF-8 JSON Without Duplicate Object Names or Non-Finite Numbers](parse-bounded-utf-8-json-without-duplicate-object-names-or-non-finite-numbers.md)
<!-- catalog:related:end -->

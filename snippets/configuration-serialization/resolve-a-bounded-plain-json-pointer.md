---
title: "Resolve a Bounded Plain JSON Pointer"
snippet_type: recipe
use_cases:
  - configuration
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - get-nested-values-with-a-validated-dot-path.md
  - ../data-processing/project-nested-records-with-explicit-field-paths.md
  - ../security-privacy/redact-explicit-paths-in-bounded-json-like-data.md
---

# Resolve a Bounded Plain JSON Pointer

## Idea and Problem

Decode one bounded plain JSON Pointer and follow its tokens through already-validated JSON-like containers without copying the selected value.

An empty pointer selects the root. Every non-empty pointer starts with `/`, and
each reference token decodes `~0` to `~` and `~1` to `/`. Object tokens remain
strings, while a token applied to a list must use the canonical unsigned array
index grammar. Separate errors keep malformed syntax distinct from lookup and
container-shape failures.

## When to Use

Use this resolver when data has already crossed a JSON-validation boundary and
a configuration, policy, or projection needs one exact RFC 6901-style path.
It supports keys containing `/`, `~`, empty text, or decimal-looking text that
a dot-path grammar cannot address unambiguously.

Keep the document stable for the duration of the call. Use a JSON parser before
this helper when the complete value still needs validation, JSON Patch when `-`
must append to an array, or a query language when wildcards, filters, defaults,
or several results are required.

## Implementation

```python
class MalformedJsonPointerError(ValueError):
    pass


class MissingJsonPointerKeyError(KeyError):
    pass


class BadJsonPointerIndexError(IndexError):
    pass


class JsonPointerScalarContinuationError(TypeError):
    pass


class OversizedJsonPointerContainerError(ValueError):
    pass


_MAX_POINTER_CHARACTERS = 4_096
_MAX_POINTER_BYTES = 16_384
_MAX_POINTER_TOKENS = 256
_MAX_VISITED_CONTAINER_ENTRIES = 65_536
_MAX_ARRAY_INDEX = _MAX_VISITED_CONTAINER_ENTRIES - 1


def _decode_json_pointer(pointer: object) -> tuple[str, ...]:
    if type(pointer) is not str:
        raise TypeError("pointer must be an exact string")
    if len(pointer) > _MAX_POINTER_CHARACTERS:
        raise MalformedJsonPointerError("pointer exceeds the character limit")
    try:
        encoded = pointer.encode("utf-8")
    except UnicodeEncodeError:
        raise MalformedJsonPointerError("pointer must be valid UTF-8 text") from None
    if len(encoded) > _MAX_POINTER_BYTES:
        raise MalformedJsonPointerError("pointer exceeds the UTF-8 byte limit")

    if pointer == "":
        return ()
    if pointer[0] != "/":
        raise MalformedJsonPointerError("a non-empty pointer must start with '/'")
    if pointer.count("/") > _MAX_POINTER_TOKENS:
        raise MalformedJsonPointerError("pointer exceeds the token limit")

    decoded_tokens: list[str] = []
    for raw_token in pointer[1:].split("/"):
        decoded_characters: list[str] = []
        position = 0
        while position < len(raw_token):
            character = raw_token[position]
            if character != "~":
                decoded_characters.append(character)
                position += 1
                continue
            if position + 1 == len(raw_token) or raw_token[position + 1] not in "01":
                raise MalformedJsonPointerError("pointer contains a malformed escape")
            decoded_characters.append("~" if raw_token[position + 1] == "0" else "/")
            position += 2
        decoded_tokens.append("".join(decoded_characters))
    return tuple(decoded_tokens)


def _bounded_array_index(token: str, *, token_position: int) -> int:
    if token == "0":
        return 0
    if (
        not token
        or token[0] not in "123456789"
        or any(character not in "0123456789" for character in token[1:])
    ):
        raise BadJsonPointerIndexError(
            f"token {token_position} is not a canonical array index"
        )
    if len(token) > 5 or (len(token) == 5 and token > str(_MAX_ARRAY_INDEX)):
        raise BadJsonPointerIndexError(
            f"token {token_position} exceeds the bounded array index range"
        )
    return int(token)


def resolve_json_pointer(document: object, pointer: str) -> object:
    """Return the object selected by one bounded plain JSON Pointer."""
    tokens = _decode_json_pointer(pointer)
    current = document

    for token_position, token in enumerate(tokens):
        if type(current) is dict:
            if len(current) > _MAX_VISITED_CONTAINER_ENTRIES:
                raise OversizedJsonPointerContainerError(
                    f"object at token {token_position} exceeds the entry limit"
                )
            if token not in current:
                raise MissingJsonPointerKeyError(
                    f"object has no key for token {token_position}"
                )
            current = current[token]
            continue

        if type(current) is list:
            if len(current) > _MAX_VISITED_CONTAINER_ENTRIES:
                raise OversizedJsonPointerContainerError(
                    f"array at token {token_position} exceeds the entry limit"
                )
            index = _bounded_array_index(token, token_position=token_position)
            if index >= len(current):
                raise BadJsonPointerIndexError(
                    f"token {token_position} is outside the visited array"
                )
            current = current[index]
            continue

        raise JsonPointerScalarContinuationError(
            f"token {token_position} continues through a scalar value"
        )

    return current
```

## Example

```python
document = {
    "": "empty key",
    "a/b": {"~1": {"answer": 42}},
    "items": [{"00": "numeric object key"}],
    "scalar": 7,
}

root = resolve_json_pointer(document, "")
empty_key = resolve_json_pointer(document, "/")
escaped = resolve_json_pointer(document, "/a~1b/~01")
numeric_object_key = resolve_json_pointer(document, "/items/0/00")

errors = []
for pointer, error_type in (
    ("/a~2b", MalformedJsonPointerError),
    ("/missing", MissingJsonPointerKeyError),
    ("/items/00", BadJsonPointerIndexError),
    ("/scalar/next", JsonPointerScalarContinuationError),
):
    try:
        resolve_json_pointer(document, pointer)
    except error_type:
        errors.append(error_type)

assert (
    root is document,
    empty_key,
    escaped is document["a/b"]["~1"],
    escaped["answer"],
    numeric_object_key,
    tuple(errors),
) == (
    True,
    "empty key",
    True,
    42,
    "numeric object key",
    (
        MalformedJsonPointerError,
        MissingJsonPointerKeyError,
        BadJsonPointerIndexError,
        JsonPointerScalarContinuationError,
    ),
)
```

## Trade-offs and Limitations

For `P` pointer characters and `T` decoded tokens, validation, decoding, and
traversal take `O(P + T)` time and `O(P + T)` temporary memory. Dictionary
hashing and the size of the selected value are outside that bound. The
five-digit array-index guard avoids converting an integer larger than any
index permitted by the visited-container cap.

The document must already be valid JSON-like data; only exact `dict` and `list`
objects encountered while tokens remain are shape- and size-checked. Selecting
the root or a final child does not validate that complete value. The result is
the original object reference, so mutable results can change later and callers
must prevent concurrent document mutation.

This helper excludes URI-fragment and Relative JSON Pointer forms, JSONPath,
JSON Patch operations, `-` array append semantics, wildcards, defaults,
coercion, Unicode normalization, whole-document validation, and deep copying.

## Related Snippets

<!-- catalog:related:start -->
- [Get Nested Values with a Validated Dot Path](get-nested-values-with-a-validated-dot-path.md)
- [Project Nested Records with Explicit Field Paths](../data-processing/project-nested-records-with-explicit-field-paths.md)
- [Redact Explicit Paths in Bounded JSON-Like Data](../security-privacy/redact-explicit-paths-in-bounded-json-like-data.md)
<!-- catalog:related:end -->

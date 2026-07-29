---
title: "Decode Bounded JSON Lines Across Arbitrary Text Chunks"
snippet_type: pattern
use_cases:
  - data-transformation
  - parsing
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - limit-text-lines-across-arbitrary-chunks.md
  - ../configuration-serialization/parse-bounded-utf-8-json-without-duplicate-object-names-or-non-finite-numbers.md
  - ../testing-tooling/enumerate-every-contiguous-chunking-of-bounded-bytes-for-stream-tests.md
---

# Decode Bounded JSON Lines Across Arbitrary Text Chunks

## Idea and Problem

Decode a complete bounded JSON Lines document independently of where trusted text decoding divided its chunks.

The state machine recognizes LF and CRLF records even when `\r` and `\n`
arrive separately. It rejects ambiguous JSON features, freezes each parsed
tree, and returns records only after `finish()` succeeds. A content failure
poisons the decoder, so changing chunk boundaries cannot expose a different
accepted prefix or allow parsing to resume after malformed input.

## When to Use

Use this pattern after an incremental decoder has already produced exact text
chunks and one bounded in-memory result is acceptable. It fits small imports,
protocol fixtures and control files that require one JSON value per non-empty
line, with optional final unterminated data.

Use a byte-level incremental decoder first when UTF-8 sequences can cross
chunks. Use an event-based JSON parser when a single record can exceed the
pre-parse line cap, or a streaming consumer when all returned trees cannot be
retained until the document is complete.

## Implementation

```python
import json
import math
from collections.abc import Mapping
from dataclasses import dataclass
from enum import StrEnum
from types import MappingProxyType

_MAX_INPUT_BYTES = 1 * 1_024 * 1_024
_MAX_RECORD_BYTES = 64 * 1_024
_MAX_RECORDS = 4_096
_MAX_CONTAINER_DEPTH = 32
_MAX_NODES = 2_048
_MAX_CONTAINER_ITEMS = 1_024
_MAX_TEXT_ITEM_BYTES = 32 * 1_024
_MAX_TEXT_BYTES = 64 * 1_024
_MAX_NUMBER_TOKEN_CHARACTERS = 128

type FrozenJson = (
    None | bool | int | float | str | tuple[FrozenJson, ...] | Mapping[str, FrozenJson]
)


class JsonLinesProblem(StrEnum):
    INPUT_TOO_LARGE = "input_too_large"
    INVALID_UNICODE = "invalid_unicode"
    RECORD_TOO_LARGE = "record_too_large"
    TOO_MANY_RECORDS = "too_many_records"
    EMPTY_RECORD = "empty_record"
    BARE_CARRIAGE_RETURN = "bare_carriage_return"
    INVALID_JSON = "invalid_json"
    DUPLICATE_NAME = "duplicate_name"
    NON_FINITE_NUMBER = "non_finite_number"
    NUMBER_TOKEN_TOO_LONG = "number_token_too_long"
    SURROGATE_TEXT = "surrogate_text"
    CONTAINER_TOO_DEEP = "container_too_deep"
    TOO_MANY_NODES = "too_many_nodes"
    TOO_MANY_CONTAINER_ITEMS = "too_many_container_items"
    TEXT_ITEM_TOO_LARGE = "text_item_too_large"
    TEXT_BUDGET_EXCEEDED = "text_budget_exceeded"
    PARSER_RECURSION = "parser_recursion"


class JsonLinesError(ValueError):
    def __init__(self, problem: JsonLinesProblem, *, record_number: int) -> None:
        self.problem = problem
        self.record_number = record_number
        super().__init__(f"{problem.value} at record {record_number}")


class _HookProblem(Exception):
    def __init__(self, problem: JsonLinesProblem) -> None:
        self.problem = problem


@dataclass(slots=True)
class _TreeBudget:
    nodes: int = 0
    text_bytes: int = 0


def _bounded_integer(token: str) -> int:
    if len(token) > _MAX_NUMBER_TOKEN_CHARACTERS:
        raise _HookProblem(JsonLinesProblem.NUMBER_TOKEN_TOO_LONG)
    return int(token)


def _bounded_float(token: str) -> float:
    if len(token) > _MAX_NUMBER_TOKEN_CHARACTERS:
        raise _HookProblem(JsonLinesProblem.NUMBER_TOKEN_TOO_LONG)
    value = float(token)
    if not math.isfinite(value):
        raise _HookProblem(JsonLinesProblem.NON_FINITE_NUMBER)
    return value


def _reject_constant(token: str) -> object:
    del token
    raise _HookProblem(JsonLinesProblem.NON_FINITE_NUMBER)


def _unique_object(pairs: list[tuple[str, object]]) -> dict[str, object]:
    if len(pairs) > _MAX_CONTAINER_ITEMS:
        raise _HookProblem(JsonLinesProblem.TOO_MANY_CONTAINER_ITEMS)
    result: dict[str, object] = {}
    for name, value in pairs:
        if name in result:
            raise _HookProblem(JsonLinesProblem.DUPLICATE_NAME)
        result[name] = value
    return result


def _count_text(value: str, budget: _TreeBudget) -> None:
    try:
        size = len(value.encode("utf-8", errors="strict"))
    except UnicodeEncodeError:
        raise _HookProblem(JsonLinesProblem.SURROGATE_TEXT) from None
    if size > _MAX_TEXT_ITEM_BYTES:
        raise _HookProblem(JsonLinesProblem.TEXT_ITEM_TOO_LARGE)
    budget.text_bytes += size
    if budget.text_bytes > _MAX_TEXT_BYTES:
        raise _HookProblem(JsonLinesProblem.TEXT_BUDGET_EXCEEDED)


def _freeze(
    value: object,
    budget: _TreeBudget,
    *,
    container_depth: int,
) -> FrozenJson:
    budget.nodes += 1
    if budget.nodes > _MAX_NODES:
        raise _HookProblem(JsonLinesProblem.TOO_MANY_NODES)

    if value is None or type(value) in (bool, int):
        return value
    if type(value) is float:
        if not math.isfinite(value):
            raise _HookProblem(JsonLinesProblem.NON_FINITE_NUMBER)
        return value
    if type(value) is str:
        _count_text(value, budget)
        return value

    depth = container_depth + 1
    if depth > _MAX_CONTAINER_DEPTH:
        raise _HookProblem(JsonLinesProblem.CONTAINER_TOO_DEEP)
    if type(value) is list:
        if len(value) > _MAX_CONTAINER_ITEMS:
            raise _HookProblem(JsonLinesProblem.TOO_MANY_CONTAINER_ITEMS)
        return tuple(
            _freeze(item, budget, container_depth=depth)
            for item in value
        )
    if type(value) is dict:
        if len(value) > _MAX_CONTAINER_ITEMS:
            raise _HookProblem(JsonLinesProblem.TOO_MANY_CONTAINER_ITEMS)
        frozen: dict[str, FrozenJson] = {}
        for name, item in value.items():
            _count_text(name, budget)
            frozen[name] = _freeze(item, budget, container_depth=depth)
        return MappingProxyType(frozen)
    raise AssertionError("the JSON decoder returned an unsupported value")


def _decode_record(text: str, *, record_number: int) -> FrozenJson:
    try:
        parsed = json.loads(
            text,
            object_pairs_hook=_unique_object,
            parse_int=_bounded_integer,
            parse_float=_bounded_float,
            parse_constant=_reject_constant,
        )
        return _freeze(parsed, _TreeBudget(), container_depth=0)
    except _HookProblem as error:
        problem = error.problem
        del error
        raise JsonLinesError(problem, record_number=record_number) from None
    except json.JSONDecodeError:
        raise JsonLinesError(
            JsonLinesProblem.INVALID_JSON,
            record_number=record_number,
        ) from None
    except RecursionError:
        raise JsonLinesError(
            JsonLinesProblem.PARSER_RECURSION,
            record_number=record_number,
        ) from None


class JsonLinesDecoder:
    def __init__(self) -> None:
        self._state = "active"
        self._input_bytes = 0
        self._record_bytes = 0
        self._record_parts: list[str] = []
        self._pending_carriage_return = False
        self._records: list[FrozenJson] = []

    def _require_active(self) -> None:
        if self._state != "active":
            raise RuntimeError("decoder is no longer active")

    def _fail(self, problem: JsonLinesProblem) -> None:
        self._state = "failed"
        raise JsonLinesError(problem, record_number=len(self._records) + 1)

    def _admit_character(self, character: str) -> int:
        try:
            size = len(character.encode("utf-8", errors="strict"))
        except UnicodeEncodeError:
            self._fail(JsonLinesProblem.INVALID_UNICODE)
        self._input_bytes += size
        if self._input_bytes > _MAX_INPUT_BYTES:
            self._fail(JsonLinesProblem.INPUT_TOO_LARGE)
        return size

    def _accept_record(self) -> None:
        if len(self._records) >= _MAX_RECORDS:
            self._fail(JsonLinesProblem.TOO_MANY_RECORDS)
        raw_record = "".join(self._record_parts)
        if not raw_record.strip(" \t"):
            self._fail(JsonLinesProblem.EMPTY_RECORD)

        try:
            record = _decode_record(
                raw_record,
                record_number=len(self._records) + 1,
            )
        except JsonLinesError:
            self._state = "failed"
            raise
        self._records.append(record)
        self._record_parts.clear()
        self._record_bytes = 0

    def feed(self, chunk: str) -> None:
        self._require_active()
        if type(chunk) is not str:
            self._state = "failed"
            raise TypeError("chunk must be an exact string")
        for character in chunk:
            if self._pending_carriage_return:
                if character != "\n":
                    self._fail(JsonLinesProblem.BARE_CARRIAGE_RETURN)
                self._admit_character(character)
                self._pending_carriage_return = False
                self._accept_record()
                continue

            if len(self._records) >= _MAX_RECORDS and not self._record_parts:
                self._fail(JsonLinesProblem.TOO_MANY_RECORDS)
            size = self._admit_character(character)
            if character == "\r":
                self._pending_carriage_return = True
            elif character == "\n":
                self._accept_record()
            else:
                self._record_bytes += size
                if self._record_bytes > _MAX_RECORD_BYTES:
                    self._fail(JsonLinesProblem.RECORD_TOO_LARGE)
                self._record_parts.append(character)

    def finish(self) -> tuple[FrozenJson, ...]:
        self._require_active()
        if self._pending_carriage_return:
            self._fail(JsonLinesProblem.BARE_CARRIAGE_RETURN)
        if self._record_parts:
            self._accept_record()
        self._state = "finished"
        return tuple(self._records)
```

## Example

```python
def decode(chunks: tuple[str, ...]) -> tuple[FrozenJson, ...]:
    decoder = JsonLinesDecoder()
    for chunk in chunks:
        decoder.feed(chunk)
    return decoder.finish()


chunked = decode(("{\"id\": 1}\r", "\n[true,", " null]"))
whole = decode(("{\"id\": 1}\r\n[true, null]",))
empty = decode(())

failed = JsonLinesDecoder()
try:
    failed.feed('{"name": 1, "\\u006eame": 2}\n')
except JsonLinesError as error:
    failure = (error.problem, error.record_number)
else:
    failure = None

try:
    failed.feed("null\n")
except RuntimeError:
    terminal = True
else:
    terminal = False

assert chunked == whole
assert chunked[0]["id"] == 1
assert chunked[1] == (True, None)
assert empty == ()
assert failure == (JsonLinesProblem.DUPLICATE_NAME, 1)
assert terminal
```

## Trade-offs and Limitations

Aggregate admitted input is one MiB, every raw record is checked against 64
KiB before parsing, and returned trees have additional structural budgets.
The standard JSON parser still materializes one admitted raw tree before those
post-parse limits are checked. The decoder also retains all frozen records
until successful `finish()`, so its memory is bounded but not constant.

Framing accepts LF and CRLF, including `\r` and `\n` split across calls, and
accepts one final non-empty unterminated record. It rejects empty or
whitespace-only records, bare carriage returns, duplicate decoded object
names, non-finite numbers, decoded surrogate code points and malformed JSON.
An entirely empty stream returns `()`. Input is already text: the class does
not decode bytes, strip a BOM, normalize Unicode, recover partial records or
resume after a content or API-type failure. Instances are stateful and not
safe for concurrent calls.

## Related Snippets

<!-- catalog:related:start -->
- [Limit Text Lines Across Arbitrary Chunks](limit-text-lines-across-arbitrary-chunks.md)
- [Parse Bounded UTF-8 JSON Without Duplicate Object Names or Non-Finite Numbers](../configuration-serialization/parse-bounded-utf-8-json-without-duplicate-object-names-or-non-finite-numbers.md)
- [Enumerate Every Contiguous Chunking of Bounded Bytes for Stream Tests](../testing-tooling/enumerate-every-contiguous-chunking-of-bounded-bytes-for-stream-tests.md)
<!-- catalog:related:end -->

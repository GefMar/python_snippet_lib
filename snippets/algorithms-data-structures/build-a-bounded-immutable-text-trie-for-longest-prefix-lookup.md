---
title: "Build a Bounded Immutable Text Trie for Longest-Prefix Lookup"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../data-processing/route-items-by-ordered-text-prefixes.md
  - ../networking-protocols/classify-a-pre-resolved-ip-against-a-bounded-cidr-zone-snapshot.md
  - ../python-language/build-a-read-only-mapping-with-canonical-text-keys.md
---

# Build a Bounded Immutable Text Trie for Longest-Prefix Lookup

## Idea and Problem

Index a bounded set of exact text prefixes once and return the binding attached to the longest prefix of each query.

Validate every prefix and value before building a flattened trie. Each frozen
node stores an optional binding index and a sorted tuple of character-to-child
transitions. A query follows those transitions with binary search and remembers
the deepest binding reached, so overlapping prefixes use specificity rather
than declaration priority.

## When to Use

Use this structure for repeated lookups against one fixed in-memory set of text
prefixes when the longest registered match is the complete decision rule. It
fits bounded classification tables where exact, case-sensitive Unicode text is
already the intended key representation and caller order must not affect the
answer.

Use an exact mapping when only complete keys can match. Use an ordered linear
scan when caller priority should override prefix length, or a specialized
library when entries change, compressed storage is required, or queries need
autocomplete, wildcard, or approximate matching.

## Implementation

```python
from bisect import bisect_left
from dataclasses import dataclass

_MAX_BINDINGS = 2_048
_MAX_BINDING_CHARACTERS = 128
_MAX_BINDING_BYTES = 512
_MAX_TOTAL_PREFIX_CHARACTERS = 65_536
_MAX_TOTAL_BINDING_BYTES = 1 << 20
_MAX_QUERY_CHARACTERS = 4_096
_MAX_QUERY_BYTES = 16_384


@dataclass(frozen=True, slots=True)
class PrefixBinding:
    prefix: str
    value: str


@dataclass(frozen=True, slots=True)
class _TrieNode:
    binding_index: int | None
    transitions: tuple[tuple[str, int], ...]


def _validated_text(
    value: object,
    *,
    field: str,
    allow_empty: bool,
    max_characters: int,
    max_bytes: int,
) -> int:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not allow_empty and not value:
        raise ValueError(f"{field} must not be empty")
    if len(value) > max_characters:
        raise ValueError(f"{field} exceeds the character limit")
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError:
        raise ValueError(f"{field} must be valid UTF-8 text") from None
    if len(encoded) > max_bytes:
        raise ValueError(f"{field} exceeds the encoded byte limit")
    return len(encoded)


def _validated_bindings(
    bindings: object,
) -> tuple[PrefixBinding, ...]:
    if type(bindings) is not tuple:
        raise TypeError("bindings must be an exact tuple")
    if not 1 <= len(bindings) <= _MAX_BINDINGS:
        raise ValueError("binding count is outside the supported range")

    seen_prefixes: set[str] = set()
    total_prefix_characters = 0
    total_binding_bytes = 0
    for index, binding in enumerate(bindings):
        if type(binding) is not PrefixBinding:
            raise TypeError(f"bindings[{index}] must be an exact PrefixBinding")

        prefix_bytes = _validated_text(
            binding.prefix,
            field=f"bindings[{index}].prefix",
            allow_empty=False,
            max_characters=_MAX_BINDING_CHARACTERS,
            max_bytes=_MAX_BINDING_BYTES,
        )
        value_bytes = _validated_text(
            binding.value,
            field=f"bindings[{index}].value",
            allow_empty=False,
            max_characters=_MAX_BINDING_CHARACTERS,
            max_bytes=_MAX_BINDING_BYTES,
        )
        if binding.prefix in seen_prefixes:
            raise ValueError(f"bindings[{index}] duplicates a prefix")
        seen_prefixes.add(binding.prefix)

        total_prefix_characters += len(binding.prefix)
        if total_prefix_characters > _MAX_TOTAL_PREFIX_CHARACTERS:
            raise ValueError("prefixes exceed the aggregate character limit")
        total_binding_bytes += prefix_bytes + value_bytes
        if total_binding_bytes > _MAX_TOTAL_BINDING_BYTES:
            raise ValueError("bindings exceed the aggregate encoded byte limit")

    return bindings


@dataclass(frozen=True, slots=True, init=False)
class ImmutableTextTrie:
    _bindings: tuple[PrefixBinding, ...]
    _nodes: tuple[_TrieNode, ...]

    def __init__(self, bindings: tuple[PrefixBinding, ...]) -> None:
        checked_bindings = _validated_bindings(bindings)

        children: list[dict[str, int]] = [{}]
        binding_indexes: list[int | None] = [None]
        for binding_index, binding in enumerate(checked_bindings):
            node_index = 0
            for character in binding.prefix:
                child_index = children[node_index].get(character)
                if child_index is None:
                    child_index = len(children)
                    children[node_index][character] = child_index
                    children.append({})
                    binding_indexes.append(None)
                node_index = child_index
            binding_indexes[node_index] = binding_index

        nodes = tuple(
            _TrieNode(
                binding_index=binding_indexes[node_index],
                transitions=tuple(sorted(node_children.items())),
            )
            for node_index, node_children in enumerate(children)
        )
        object.__setattr__(self, "_bindings", checked_bindings)
        object.__setattr__(self, "_nodes", nodes)

    def longest_prefix(self, text: str) -> PrefixBinding | None:
        """Return the binding for the deepest exact prefix of text, if any."""
        _validated_text(
            text,
            field="text",
            allow_empty=True,
            max_characters=_MAX_QUERY_CHARACTERS,
            max_bytes=_MAX_QUERY_BYTES,
        )

        node_index = 0
        best_binding_index: int | None = None
        for character in text:
            transitions = self._nodes[node_index].transitions
            transition_index = bisect_left(
                transitions,
                character,
                key=lambda transition: transition[0],
            )
            if (
                transition_index == len(transitions)
                or transitions[transition_index][0] != character
            ):
                break

            node_index = transitions[transition_index][1]
            binding_index = self._nodes[node_index].binding_index
            if binding_index is not None:
                best_binding_index = binding_index

        if best_binding_index is None:
            return None
        return self._bindings[best_binding_index]
```

## Example

```python
bindings = (
    PrefixBinding("a", "letter"),
    PrefixBinding("app", "application"),
    PrefixBinding("apple", "fruit"),
    PrefixBinding("é", "composed"),
)
trie = ImmutableTextTrie(bindings)
reordered = ImmutableTextTrie(tuple(reversed(bindings)))

queries = ("applejack", "application", "axis", "éclair", "e\u0301clair", "")
matches = tuple(trie.longest_prefix(query) for query in queries)
reordered_matches = tuple(reordered.longest_prefix(query) for query in queries)

try:
    ImmutableTextTrie(
        (
            PrefixBinding("app", "first"),
            PrefixBinding("app", "second"),
        )
    )
except ValueError:
    duplicate_rejected = True
else:
    duplicate_rejected = False

assert (matches, reordered_matches, duplicate_rejected) == (
    (
        PrefixBinding("apple", "fruit"),
        PrefixBinding("app", "application"),
        PrefixBinding("a", "letter"),
        PrefixBinding("é", "composed"),
        None,
        None,
    ),
    matches,
    True,
)
```

## Trade-offs and Limitations

For `P` total prefix characters, construction takes `O(P log P)` worst-case
time because each node's transitions are sorted, and the flattened nodes use
`O(P)` memory. Validating a query of `Q` characters takes `O(Q)` time. Traversal
then takes `O(Q log D)` time for maximum node degree `D`, with one binary search
per consumed character.

The trie, every binding, and every node are frozen slotted dataclasses. All
retained containers and transitions are tuples, and their leaf values are
immutable strings, integers, or `None`; temporary dictionaries are discarded
before construction completes. The structure has no public mutation operation,
but Python object immutability is not a security boundary. A lookup returns the
original frozen binding, and an empty query returns `None` because empty
prefixes are forbidden.

Matching compares exact, case-sensitive Unicode code points without
normalization. Canonically equivalent spellings may therefore differ. The trie
does not implement caller-priority routing, updates, autocomplete, all-prefix
enumeration, wildcards, fuzzy or radix matching, byte or path semantics,
persistence, or thread-safety guarantees.

## Related Snippets

<!-- catalog:related:start -->
- [Route Items by Ordered Text Prefixes](../data-processing/route-items-by-ordered-text-prefixes.md)
- [Classify a Pre-Resolved IP Against a Bounded CIDR-Zone Snapshot](../networking-protocols/classify-a-pre-resolved-ip-against-a-bounded-cidr-zone-snapshot.md)
- [Build a Read-Only Mapping with Canonical Text Keys](../python-language/build-a-read-only-mapping-with-canonical-text-keys.md)
<!-- catalog:related:end -->

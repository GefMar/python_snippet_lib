---
title: "Decide Unique Decodability of a Bounded Byte Code with Sardinas–Patterson"
snippet_type: algorithm
use_cases:
  - testing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/build-a-canonical-binary-prefix-codebook-from-declared-byte-code-lengths.md
  - compile-and-match-a-bounded-postfix-regular-expression-with-a-thompson-epsilon-nfa.md
  - find-all-exact-multi-pattern-text-matches-with-aho-corasick.md
---

# Decide Unique Decodability of a Bounded Byte Code with Sardinas–Patterson

## Idea and Problem

Decide whether every finite concatenation of codewords has at most one codeword sequence that produces it.

A code need not be prefix-free to be uniquely decodable. Sardinas–Patterson
tracks the non-empty byte suffix left after cancelling one partial decoding
against another. If a residual ever equals a complete codeword, the two paths
can finish at the same byte string and the code is ambiguous. If every
reachable residual has been processed without that equality, no ambiguity
exists.

Every residual is a proper suffix of an input codeword, so the bounded search
terminates. Sorting the validated codewords makes traversal independent of the
input tuple's declaration order; the public result is only the decision.

## When to Use

Use this check before accepting a small, fixed byte code whose codeword
boundaries will not be stored separately. It is useful when a non-prefix code
is intentional and rejecting every prefix overlap would be unnecessarily
strict.

Use a direct prefix-free check when instantaneous left-to-right decoding is a
requirement. Use a format-specific parser when framing, separators, lengths,
end markers, or out-of-band boundaries make concatenations unambiguous by a
different rule.

## Implementation

```python
from collections import deque

_MAX_CODEWORD_COUNT = 32
_MAX_CODEWORD_BYTES = 32
_MAX_TOTAL_CODEWORD_BYTES = 512


def is_uniquely_decodable(codewords: tuple[bytes, ...]) -> bool:
    """Return whether every finite concatenation has at most one decoding."""
    if type(codewords) is not tuple:
        raise TypeError("codewords must be an exact tuple")
    if not 1 <= len(codewords) <= _MAX_CODEWORD_COUNT:
        raise ValueError("codeword count is outside 1..32")

    seen_codewords: set[bytes] = set()
    total_bytes = 0
    for index, codeword in enumerate(codewords):
        if type(codeword) is not bytes:
            raise TypeError(f"codewords[{index}] must be exact bytes")
        if not codeword:
            raise ValueError(f"codewords[{index}] must not be empty")
        if len(codeword) > _MAX_CODEWORD_BYTES:
            raise ValueError(f"codewords[{index}] exceeds 32 bytes")
        if codeword in seen_codewords:
            raise ValueError(f"codewords[{index}] is duplicated")
        seen_codewords.add(codeword)
        total_bytes += len(codeword)

    if total_bytes > _MAX_TOTAL_CODEWORD_BYTES:
        raise ValueError("codewords exceed 512 aggregate bytes")

    words = tuple(sorted(codewords))
    pending: deque[bytes] = deque()
    seen_residuals: set[bytes] = set()

    def enqueue(residual: bytes) -> None:
        if residual and residual not in seen_residuals:
            seen_residuals.add(residual)
            pending.append(residual)

    for shorter in words:
        for longer in words:
            if shorter != longer and longer.startswith(shorter):
                enqueue(longer[len(shorter) :])

    while pending:
        residual = pending.popleft()
        for word in words:
            if residual == word:
                return False
            if residual.startswith(word):
                enqueue(residual[len(word) :])
            elif word.startswith(residual):
                enqueue(word[len(residual) :])

    return True
```

## Example

```python
from itertools import combinations, product


def count_decodings_up_to_two(
    encoded: bytes,
    codewords: tuple[bytes, ...],
) -> int:
    counts = [0] * (len(encoded) + 1)
    counts[0] = 1
    for offset, count in enumerate(counts):
        if count == 0:
            continue
        for codeword in codewords:
            if encoded.startswith(codeword, offset):
                end = offset + len(codeword)
                counts[end] = min(2, counts[end] + count)
    return counts[-1]


def tiny_unique_decodability_oracle(
    codewords: tuple[bytes, ...],
) -> bool:
    if len(codewords) > 3 or sum(map(len, codewords)) > 6:
        raise ValueError("the exhaustive oracle accepts only tiny codebooks")

    proper_suffixes = {
        codeword[offset:] for codeword in codewords for offset in range(1, len(codeword))
    }
    maximum_parse_words = len(proper_suffixes) + 1
    for word_count in range(1, maximum_parse_words + 1):
        for parse in product(codewords, repeat=word_count):
            encoded = b"".join(parse)
            if count_decodings_up_to_two(encoded, codewords) == 2:
                return False
    return True


prefix_free = (b"0", b"10", b"11")
unique_but_not_prefix_free = (b"0", b"01")
immediate_ambiguity = (b"0", b"00")
delayed_ambiguity = (b"0", b"01", b"10")
repeated_residual = (b"a", b"ab", b"bb")
maximum_count = tuple(bytes((value,)) for value in range(_MAX_CODEWORD_COUNT))
maximum_total_bytes = tuple(bytes((value,)) * _MAX_CODEWORD_BYTES for value in range(16))

tiny_words = (b"0", b"1", b"00", b"01", b"10", b"11")
checked = 0
for codeword_count in range(1, 4):
    for codebook in combinations(tiny_words, codeword_count):
        expected = tiny_unique_decodability_oracle(codebook)
        assert is_uniquely_decodable(codebook) == expected
        assert is_uniquely_decodable(tuple(reversed(codebook))) == expected
        checked += 1

invalid_inputs = (
    [],
    (),
    (b"",),
    (bytearray(b"0"),),
    (b"0", b"0"),
    (b"x" * 33,),
    tuple(bytes((value,)) for value in range(33)),
    tuple(bytes((value,)) * 32 for value in range(17)),
)
rejected = 0
for invalid in invalid_inputs:
    try:
        is_uniquely_decodable(invalid)  # type: ignore[arg-type]
    except (TypeError, ValueError):
        rejected += 1

assert (
    checked == 41
    and rejected == len(invalid_inputs)
    and is_uniquely_decodable(prefix_free)
    and is_uniquely_decodable(unique_but_not_prefix_free)
    and not is_uniquely_decodable(immediate_ambiguity)
    and not is_uniquely_decodable(delayed_ambiguity)
    and is_uniquely_decodable(repeated_residual)
    and is_uniquely_decodable(tuple(reversed(repeated_residual)))
    and is_uniquely_decodable(maximum_count)
    and is_uniquely_decodable(maximum_total_bytes)
)
```

## Trade-offs and Limitations

For `C` codewords of at most `L` bytes, initial residual discovery takes
`O(C**2 * L)` time. Processing `S` distinct residuals takes
`O(S * C * L)` time, including bounded prefix comparisons and suffix copies.
The queue and set retain `O(S * L)` bytes in the worst case. Here every
residual belongs to the set of non-empty proper codeword suffixes, so
`S <= sum(len(word) - 1 for word in codewords) <= 496` under the declared
caps.

Prefix freedom is sufficient but not necessary: `(b"0", b"01")` is uniquely
decodable even though one word prefixes another. Repeated residuals are normal;
for `(b"a", b"ab", b"bb")`, residual `b"b"` leads back to itself, and the
visited set makes that cycle terminate without changing the correct result.

The function treats the tuple as an unordered set and rejects duplicates,
empty words, oversized words, and oversized aggregate input. It returns no
ambiguity witness and does not generate codes, decode payloads, choose framing,
recover from errors, or provide error-correcting guarantees.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Canonical Binary Prefix Codebook from Declared Byte Code Lengths](../configuration-serialization/build-a-canonical-binary-prefix-codebook-from-declared-byte-code-lengths.md)
- [Compile and Match a Bounded Postfix Regular Expression with a Thompson Epsilon-NFA](compile-and-match-a-bounded-postfix-regular-expression-with-a-thompson-epsilon-nfa.md)
- [Find All Exact Multi-Pattern Text Matches with Aho-Corasick](find-all-exact-multi-pattern-text-matches-with-aho-corasick.md)
<!-- catalog:related:end -->

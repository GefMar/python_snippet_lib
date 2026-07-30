---
title: "Generate a Bounded One-Edit Byte Mutation Corpus"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - generate-integer-boundary-probes-around-closed-cut-points.md
  - shrink-a-bounded-failing-sequence-to-a-one-deletion-minimal-subsequence.md
  - enumerate-every-contiguous-chunking-of-bounded-bytes-for-stream-tests.md
---

# Generate a Bounded One-Edit Byte Mutation Corpus

## Idea and Problem

Generate a deterministic, globally deduplicated corpus containing every admitted one-byte deletion, substitution, and insertion around a short seed.

Operations are attempted in that order. Within an operation, seed positions
come first and the strictly increasing alphabet comes second. When different
edits produce the same body, the first operation in this canonical order owns
the sole retained record, keeping repetitive seeds predictable.

## When to Use

Use this corpus for focused parser, decoder, framing, and validation tests when
a reviewed seed is at most 128 bytes and a small closed alphabet represents the
byte classes worth perturbing. The frozen records let a failing test report the
edit kind, original position, participating byte value, and exact mutated body.

Choose the alphabet deliberately: it may contain delimiters, boundary byte
values, or a few format-specific symbols. A maximum of 16 alphabet values keeps
the corpus reviewable. Set `max_cases` from the test budget; the function either
returns the complete unique corpus admitted by the inputs or raises an error.

## Implementation

```python
from dataclasses import dataclass
from itertools import pairwise
from typing import Literal

MutationKind = Literal["delete", "substitute", "insert"]
_MAX_SEED_BYTES = 128
_MAX_ALPHABET_BYTES = 16
_MAX_CASES = 4_096


@dataclass(frozen=True, slots=True)
class ByteMutation:
    kind: MutationKind
    index: int
    byte_value: int
    body: bytes


def generate_one_edit_byte_mutations(
    seed: bytes,
    alphabet: bytes,
    *,
    max_cases: int,
) -> tuple[ByteMutation, ...]:
    """Return unique one-edit bodies in canonical first-operation order."""
    if type(seed) is not bytes:
        raise TypeError("seed must be exact bytes")
    if len(seed) > _MAX_SEED_BYTES:
        raise ValueError("seed exceeds the supported byte limit")
    if type(alphabet) is not bytes:
        raise TypeError("alphabet must be exact bytes")
    if not 1 <= len(alphabet) <= _MAX_ALPHABET_BYTES:
        raise ValueError("alphabet length is outside the supported range")
    if any(left >= right for left, right in pairwise(alphabet)):
        raise ValueError("alphabet bytes must be strictly increasing")
    if type(max_cases) is not int:
        raise TypeError("max_cases must be an exact non-boolean integer")
    if not 1 <= max_cases <= _MAX_CASES:
        raise ValueError("max_cases is outside the supported range")

    mutations: list[ByteMutation] = []
    seen_bodies: set[bytes] = set()

    def retain(
        kind: MutationKind,
        index: int,
        byte_value: int,
        body: bytes,
    ) -> None:
        if body in seen_bodies:
            return
        if len(mutations) == max_cases:
            raise ValueError("unique mutation count exceeds max_cases")
        seen_bodies.add(body)
        mutations.append(ByteMutation(kind, index, byte_value, body))

    for index, removed in enumerate(seed):
        retain("delete", index, removed, seed[:index] + seed[index + 1 :])

    for index, original in enumerate(seed):
        for replacement in alphabet:
            if replacement == original:
                continue
            retain(
                "substitute",
                index,
                replacement,
                seed[:index] + bytes((replacement,)) + seed[index + 1 :],
            )

    for index in range(len(seed) + 1):
        for inserted in alphabet:
            retain(
                "insert",
                index,
                inserted,
                seed[:index] + bytes((inserted,)) + seed[index:],
            )

    return tuple(mutations)


```

## Example

```python
mutations = generate_one_edit_byte_mutations(
    b"aa",
    b"ab",
    max_cases=7,
)

assert mutations == (
    ByteMutation("delete", 0, ord("a"), b"a"),
    ByteMutation("substitute", 0, ord("b"), b"ba"),
    ByteMutation("substitute", 1, ord("b"), b"ab"),
    ByteMutation("insert", 0, ord("a"), b"aaa"),
    ByteMutation("insert", 0, ord("b"), b"baa"),
    ByteMutation("insert", 1, ord("b"), b"aba"),
    ByteMutation("insert", 2, ord("b"), b"aab"),
)
assert len({mutation.body for mutation in mutations}) == len(mutations)

assert generate_one_edit_byte_mutations(
    b"",
    b"\x00\xff",
    max_cases=2,
) == (
    ByteMutation("insert", 0, 0, b"\x00"),
    ByteMutation("insert", 0, 255, b"\xff"),
)

try:
    generate_one_edit_byte_mutations(b"aa", b"ab", max_cases=6)
except ValueError:
    undersized_cap_rejected = True
else:
    undersized_cap_rejected = False

try:
    generate_one_edit_byte_mutations(b"seed", b"ba", max_cases=100)
except ValueError:
    unsorted_alphabet_rejected = True
else:
    unsorted_alphabet_rejected = False

assert undersized_cap_rejected and unsorted_alphabet_rejected
```

## Trade-offs and Limitations

For seed length `N` and alphabet length `A`, at most
`M = N + N*A + (N + 1)*A` edit attempts are considered before no-op
substitutions and duplicate bodies are removed. Because immutable byte slicing
and hashing can inspect `O(N)` bytes per attempt, runtime is `O(M*N)`. If `U`
unique bodies survive, the returned records and deduplication set use `O(U*N)`
memory. The implementation retains at most `max_cases` bodies and fails on the
next unique body, exposing no partial result.

For deletion, `byte_value` is the removed byte; for substitution and insertion,
it is the new byte. Deletion and substitution indices address the original
seed, while an insertion index addresses one of its `N + 1` boundaries. Global
deduplication means later edit records are deliberately absent when their body
was already produced by an earlier operation or position.

This is not an exhaustive arbitrary-byte fuzzer because the alphabet admits at
most 16 of 256 values. It does not create multi-edit cases, understand a file
format, preserve mutable-buffer identity, minimize failures, execute the system
under test, assign expected outcomes, or provide cryptographically random or
security-adversarial input generation.

## Related Snippets

<!-- catalog:related:start -->
- [Generate Integer Boundary Probes Around Closed Cut Points](generate-integer-boundary-probes-around-closed-cut-points.md)
- [Shrink a Bounded Failing Sequence to a One-Deletion-Minimal Subsequence](shrink-a-bounded-failing-sequence-to-a-one-deletion-minimal-subsequence.md)
- [Enumerate Every Contiguous Chunking of Bounded Bytes for Stream Tests](enumerate-every-contiguous-chunking-of-bounded-bytes-for-stream-tests.md)
<!-- catalog:related:end -->

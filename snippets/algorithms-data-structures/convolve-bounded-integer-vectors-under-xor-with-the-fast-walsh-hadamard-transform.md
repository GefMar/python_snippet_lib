---
title: "Convolve Bounded Integer Vectors under XOR with the Fast Walsh-Hadamard Transform"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - convolve-bounded-integer-polynomials-modulo-998244353-with-an-iterative-ntt.md
  - build-a-canonical-reduced-xor-basis-for-bounded-unsigned-integers.md
  - find-the-earliest-maximum-xor-index-pair-in-bounded-unsigned-64-bit-integers-with-a-binary-trie.md
---

# Convolve Bounded Integer Vectors under XOR with the Fast Walsh-Hadamard Transform

## Idea and Problem

Compute the exact convolution whose output index combines two input indexes by bitwise XOR without evaluating every index pair.

For two vectors of equal power-of-two length `N`, XOR convolution is
`result[k] = sum(left[i] * right[i xor k] for i in range(N))`. The
Walsh-Hadamard transform replaces this quadratic sum with layers of integer
sum-and-difference butterflies. After pointwise multiplication, applying the
same transform again produces `N` times the desired result.

Python integers preserve the complete intermediate values, and exact division
by `N` makes the inverse step free of modular inverses or rounding.

## When to Use

Use XOR convolution for bounded bitmask-indexed counts, scores, or dynamic
programming states when pair composition is XOR and an exact integer answer is
required. It is especially useful when a naive `O(N²)` oracle is available for
small tests but production vectors are larger.

Use ordinary polynomial convolution when indexes add, cyclic convolution when
indexes add modulo a length, and subset transforms when the operation is
bitwise AND or OR. Use a modular profile when results intentionally belong to
a finite field.

## Implementation

```python
from itertools import product
from random import Random

_MAX_XOR_CONVOLUTION_LENGTH = 4_096
_MIN_SIGNED_32 = -(1 << 31)
_MAX_SIGNED_32 = (1 << 31) - 1


def _validated_xor_vector(name: str, values: tuple[int, ...]) -> list[int]:
    if type(values) is not tuple:
        raise TypeError(f"{name} must be an exact tuple")
    if not 1 <= len(values) <= _MAX_XOR_CONVOLUTION_LENGTH:
        raise ValueError(f"{name} length is outside 1..4,096")
    if len(values) & (len(values) - 1):
        raise ValueError(f"{name} length must be a power of two")
    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"{name}[{index}] must be an exact integer")
        if not _MIN_SIGNED_32 <= value <= _MAX_SIGNED_32:
            raise ValueError(f"{name}[{index}] is outside signed 32-bit range")
    return list(values)


def _walsh_hadamard_in_place(values: list[int]) -> None:
    width = 1
    while width < len(values):
        for block_start in range(0, len(values), 2 * width):
            for offset in range(width):
                left_index = block_start + offset
                right_index = left_index + width
                left_value = values[left_index]
                right_value = values[right_index]
                values[left_index] = left_value + right_value
                values[right_index] = left_value - right_value
        width *= 2


def xor_convolve(
    left: tuple[int, ...],
    right: tuple[int, ...],
) -> tuple[int, ...]:
    """Return exact integer XOR convolution for equal power-of-two vectors."""
    transformed_left = _validated_xor_vector("left", left)
    transformed_right = _validated_xor_vector("right", right)
    if len(transformed_left) != len(transformed_right):
        raise ValueError("left and right must have equal lengths")

    _walsh_hadamard_in_place(transformed_left)
    _walsh_hadamard_in_place(transformed_right)
    transformed_product = [
        left_value * right_value
        for left_value, right_value in zip(
            transformed_left,
            transformed_right,
            strict=True,
        )
    ]
    _walsh_hadamard_in_place(transformed_product)

    length = len(transformed_product)
    result: list[int] = []
    for value in transformed_product:
        quotient, remainder = divmod(value, length)
        if remainder:
            raise AssertionError("inverse Walsh-Hadamard value is not exactly divisible")
        result.append(quotient)
    return tuple(result)
```

## Example

```python
def naive_xor_convolution(
    left: tuple[int, ...],
    right: tuple[int, ...],
) -> tuple[int, ...]:
    return tuple(
        sum(left[index] * right[index ^ output] for index in range(len(left)))
        for output in range(len(left))
    )


assert xor_convolve((1, 2, 3, 4), (5, 6, 7, 8)) == (70, 68, 62, 60)

checked = 0
for length in (1, 2, 4):
    vectors = tuple(product((-1, 0, 1), repeat=length))
    for left in vectors:
        for right in vectors:
            result = xor_convolve(left, right)
            assert result == naive_xor_convolution(left, right)
            assert result == xor_convolve(right, left)
            assert sum(result) == sum(left) * sum(right)
            checked += 1

rng = Random(0xF4_57)
for _ in range(1_000):
    left = tuple(rng.randrange(-20, 21) for _ in range(8))
    right = tuple(rng.randrange(-20, 21) for _ in range(8))
    assert xor_convolve(left, right) == naive_xor_convolution(left, right)

boundary = tuple((index % 17) - 8 for index in range(4_096))
impulse = (1,) + (0,) * (len(boundary) - 1)
assert checked == 6_651 and xor_convolve(boundary, impulse) == boundary
```

## Trade-offs and Limitations

For length `N`, three transforms and pointwise multiplication take
`O(N log N)` time and `O(N)` memory. Intermediate values can be much larger
than the signed-32-bit inputs, but Python integers retain them exactly. The
explicit length and input bounds keep that growth finite.

The transform requires equal power-of-two lengths and returns one complete
materialized vector. It does not compute ordinary, cyclic, AND, or OR
convolution; it has no floating-point normalization, finite-field modulus,
streaming mode, sparse representation, multidimensional layout, or automatic
padding policy.

## Related Snippets

<!-- catalog:related:start -->
- [Convolve Bounded Integer Polynomials Modulo 998244353 with an Iterative NTT](convolve-bounded-integer-polynomials-modulo-998244353-with-an-iterative-ntt.md)
- [Build a Canonical Reduced XOR Basis for Bounded Unsigned Integers](build-a-canonical-reduced-xor-basis-for-bounded-unsigned-integers.md)
- [Find the Earliest Maximum-XOR Index Pair in Bounded Unsigned 64-Bit Integers with a Binary Trie](find-the-earliest-maximum-xor-index-pair-in-bounded-unsigned-64-bit-integers-with-a-binary-trie.md)
<!-- catalog:related:end -->

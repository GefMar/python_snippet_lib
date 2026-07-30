---
title: "Convolve Bounded Integer Polynomials Modulo 998244353 with an Iterative NTT"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-a-distant-linear-recurrence-term-modulo-an-integer-by-polynomial-reduction.md
  - interpolate-a-global-polynomial-exactly-from-bounded-integer-points.md
  - solve-a-bounded-linear-congruence-as-a-canonical-residue-class.md
---

# Convolve Bounded Integer Polynomials Modulo 998244353 with an Iterative NTT

## Idea and Problem

Multiply two bounded integer polynomials and return their linear convolution as canonical residues modulo 998244353.

The number-theoretic transform evaluates coefficient vectors at powers of a
primitive root inside a finite field. Pointwise multiplication in that domain,
followed by the inverse transform, produces the same coefficients as ordinary
polynomial multiplication. Padding to a power of two avoids cyclic wraparound.

## When to Use

Use this algorithm when dense coefficient sequences are large enough for
quadratic multiplication to be costly and residues modulo `998_244_353` are the
required result. Signed input coefficients are accepted and normalized, which
is convenient for recurrence, generating-function, and combinatorial work over
this field.

Use direct nested loops for small or sparse polynomials. Choose a library with
CRT reconstruction when exact unbounded integer coefficients are required, or
when the modulus is not compatible with this transform. This function does not
compute cyclic, floating-point, multidimensional, or arbitrary-modulus
convolutions.

## Implementation

```python
_NTT_MODULUS = 998_244_353
_NTT_PRIMITIVE_ROOT = 3
_MIN_SIGNED_64 = -(1 << 63)
_MAX_SIGNED_64 = (1 << 63) - 1
_MAX_POLYNOMIAL_LENGTH = 65_536
_MAX_NTT_LENGTH = 131_072


def _normalized_ntt_coefficients(values: object, *, field: str) -> list[int]:
    if type(values) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    if len(values) > _MAX_POLYNOMIAL_LENGTH:
        raise ValueError(f"{field} contains too many coefficients")

    normalized: list[int] = []
    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"{field}[{index}] must be an exact integer")
        if not _MIN_SIGNED_64 <= value <= _MAX_SIGNED_64:
            raise ValueError(f"{field}[{index}] is outside the signed 64-bit range")
        normalized.append(value % _NTT_MODULUS)
    return normalized


def _transform_ntt(values: list[int], *, inverse: bool) -> None:
    size = len(values)
    destination = 0
    for source in range(1, size):
        bit = size >> 1
        while destination & bit:
            destination ^= bit
            bit >>= 1
        destination ^= bit
        if source < destination:
            values[source], values[destination] = values[destination], values[source]

    stage_length = 2
    while stage_length <= size:
        stage_root = pow(
            _NTT_PRIMITIVE_ROOT,
            (_NTT_MODULUS - 1) // stage_length,
            _NTT_MODULUS,
        )
        if inverse:
            stage_root = pow(stage_root, _NTT_MODULUS - 2, _NTT_MODULUS)

        half_length = stage_length // 2
        for start in range(0, size, stage_length):
            root_power = 1
            for offset in range(half_length):
                even = values[start + offset]
                odd = values[start + offset + half_length] * root_power % _NTT_MODULUS
                values[start + offset] = (even + odd) % _NTT_MODULUS
                values[start + offset + half_length] = (even - odd) % _NTT_MODULUS
                root_power = root_power * stage_root % _NTT_MODULUS
        stage_length *= 2

    if inverse:
        inverse_size = pow(size, _NTT_MODULUS - 2, _NTT_MODULUS)
        for index, value in enumerate(values):
            values[index] = value * inverse_size % _NTT_MODULUS


def convolve_modulo_998244353(
    left: tuple[int, ...],
    right: tuple[int, ...],
) -> tuple[int, ...]:
    """Return the bounded linear convolution as canonical field residues."""
    left_values = _normalized_ntt_coefficients(left, field="left")
    right_values = _normalized_ntt_coefficients(right, field="right")
    if not left_values or not right_values:
        return ()

    result_length = len(left_values) + len(right_values) - 1
    transform_length = 1 << (result_length - 1).bit_length()
    if transform_length > _MAX_NTT_LENGTH:
        raise ValueError("required transform length exceeds 131072")

    left_values.extend([0] * (transform_length - len(left_values)))
    right_values.extend([0] * (transform_length - len(right_values)))
    _transform_ntt(left_values, inverse=False)
    _transform_ntt(right_values, inverse=False)

    product = [
        left_value * right_value % _NTT_MODULUS
        for left_value, right_value in zip(left_values, right_values, strict=True)
    ]
    _transform_ntt(product, inverse=True)
    return tuple(product[:result_length])
```

## Example

```python
def convolve_quadratically(
    left: tuple[int, ...],
    right: tuple[int, ...],
) -> tuple[int, ...]:
    if not left or not right:
        return ()
    result = [0] * (len(left) + len(right) - 1)
    for left_degree, left_value in enumerate(left):
        for right_degree, right_value in enumerate(right):
            result[left_degree + right_degree] += left_value * right_value
    return tuple(value % _NTT_MODULUS for value in result)


def exercise_small_polynomials() -> int:
    from itertools import product

    polynomials = [
        polynomial
        for length in range(4)
        for polynomial in product((-1, 0, 1), repeat=length)
    ]
    checked = 0
    for left in polynomials:
        for right in polynomials:
            assert convolve_modulo_998244353(left, right) == convolve_quadratically(
                left,
                right,
            )
            checked += 1
    return checked


def exercise_seeded_polynomials() -> int:
    from random import Random

    generator = Random(20_260_730)
    checked = 0
    for _ in range(100):
        left = tuple(generator.randint(-10**9, 10**9) for _ in range(generator.randrange(17)))
        right = tuple(generator.randint(-10**9, 10**9) for _ in range(generator.randrange(17)))
        assert convolve_modulo_998244353(left, right) == convolve_quadratically(left, right)
        checked += 1
    return checked


def evaluate_modulo(coefficients: tuple[int, ...], argument: int) -> int:
    result = 0
    for coefficient in reversed(coefficients):
        result = (result * argument + coefficient) % _NTT_MODULUS
    return result


def raises(error_type: type[Exception], operation) -> bool:
    try:
        operation()
    except error_type:
        return True
    return False


left_sample = (_MIN_SIGNED_64, -3, 0, 11, _MAX_SIGNED_64)
right_sample = (7, -5, 2)
sample_product = convolve_modulo_998244353(left_sample, right_sample)
argument = 123_456

maximum_operand = (0,) * (_MAX_POLYNOMIAL_LENGTH - 1) + (_MAX_SIGNED_64,)
maximum_product = convolve_modulo_998244353(maximum_operand, (0, 1))

assert (
    exercise_small_polynomials() == 1_600
    and exercise_seeded_polynomials() == 100
    and convolve_modulo_998244353((), (1, 2)) == ()
    and convolve_modulo_998244353((0, 0), (4, -9)) == (0, 0, 0)
    and evaluate_modulo(sample_product, argument)
    == evaluate_modulo(left_sample, argument) * evaluate_modulo(right_sample, argument)
    % _NTT_MODULUS
    and len(maximum_product) == _MAX_POLYNOMIAL_LENGTH + 1
    and not any(maximum_product[:-1])
    and maximum_product[-1] == _MAX_SIGNED_64 % _NTT_MODULUS
    and raises(TypeError, lambda: convolve_modulo_998244353([1], (1,)))
    and raises(ValueError, lambda: convolve_modulo_998244353((1,) * 65_537, (1,)))
)
```

## Trade-offs and Limitations

For padded transform length `L`, the algorithm performs `O(L log L)` modular
operations and retains `O(L)` residues. Python integer multiplication,
remainder, and exponentiation costs still depend on operand bit lengths, even
though every transform residue is bounded by the fixed modulus.

The implementation accepts at most 65,536 coefficients per exact tuple and
rejects a transform longer than 131,072. It always returns the linear result in
`0..998_244_352`; it does not recover signed or unbounded integer coefficients.
The fixed modulus and primitive root are part of the contract, not parameters.

## Related Snippets

<!-- catalog:related:start -->
- [Compute a Distant Linear-Recurrence Term Modulo an Integer by Polynomial Reduction](compute-a-distant-linear-recurrence-term-modulo-an-integer-by-polynomial-reduction.md)
- [Interpolate a Global Polynomial Exactly from Bounded Integer Points](interpolate-a-global-polynomial-exactly-from-bounded-integer-points.md)
- [Solve a Bounded Linear Congruence as a Canonical Residue Class](solve-a-bounded-linear-congruence-as-a-canonical-residue-class.md)
<!-- catalog:related:end -->

---
title: "Evaluate Salted Percentage Rollouts with Integer Buckets"
snippet_type: algorithm
use_cases:
  - configuration
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../reliability-resilience/assign-stable-schedule-slots-with-a-digest.md
  - derive-a-versioned-record-key-from-explicit-identity-fields.md
  - ../algorithms-data-structures/map-keys-with-an-immutable-consistent-hash-ring.md
---

# Evaluate Salted Percentage Rollouts with Integer Buckets

## Idea and Problem

Make a repeatable percentage-rollout decision by hashing a bounded subject and public namespace into an integer bucket, then testing one half-open bucket range.

The namespace and subject are length-framed under a fixed versioned BLAKE2b
configuration, so concatenation boundaries cannot collide. Integer range
comparison avoids floating-point edge behavior. Expanding `size` while every
other input remains unchanged preserves all previously included subjects.

## When to Use

Use this algorithm for a local, deterministic cohort decision when the caller
already has a stable textual subject key and a public namespace that identifies
one rollout. Store the returned bucket when an audit needs to explain the
decision independently of a changing percentage.

Use a stateful assignment service when membership must survive algorithm or
bucket-count changes. Use a keyed, reviewed system when allocation must resist
prediction or manipulation. This unkeyed helper is never an authorization or
abuse-prevention boundary.

## Implementation

```python
from dataclasses import dataclass
from hashlib import blake2b


_PERSONALIZATION = b"rollout-v1"
_DOMAIN = b"percentage-rollout-v1"
_MAX_SUBJECT_BYTES = 512
_MAX_NAMESPACE_BYTES = 128
_MAX_BUCKET_COUNT = 1_000_000


@dataclass(frozen=True, slots=True)
class RolloutDecision:
    bucket: int
    included: bool


def _bounded_utf8(
    value: object,
    *,
    name: str,
    maximum_bytes: int,
) -> bytes:
    if type(value) is not str:
        raise TypeError(f"{name} must be exact text")
    if not value or len(value) > maximum_bytes:
        raise ValueError(f"{name} length is outside the supported range")
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError as error:
        raise ValueError(f"{name} must be valid UTF-8 text") from error
    if not 1 <= len(encoded) <= maximum_bytes:
        raise ValueError(f"{name} byte length is outside the supported range")
    return encoded


def _exact_integer(value: object, *, name: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    return value


def evaluate_rollout(
    subject_key: str,
    *,
    namespace: str,
    start: int,
    size: int,
    bucket_count: int = 10_000,
) -> RolloutDecision:
    subject_bytes = _bounded_utf8(
        subject_key,
        name="subject_key",
        maximum_bytes=_MAX_SUBJECT_BYTES,
    )
    namespace_bytes = _bounded_utf8(
        namespace,
        name="namespace",
        maximum_bytes=_MAX_NAMESPACE_BYTES,
    )
    bucket_total = _exact_integer(bucket_count, name="bucket_count")
    range_start = _exact_integer(start, name="start")
    range_size = _exact_integer(size, name="size")
    if not 1 <= bucket_total <= _MAX_BUCKET_COUNT:
        raise ValueError("bucket_count is outside the supported range")
    if not 0 <= range_start <= bucket_total:
        raise ValueError("start is outside the bucket range")
    if not 0 <= range_size <= bucket_total - range_start:
        raise ValueError("size does not fit after start")

    digest = blake2b(digest_size=8, person=_PERSONALIZATION)
    digest.update(_DOMAIN)
    for field in (namespace_bytes, subject_bytes):
        digest.update(len(field).to_bytes(2, byteorder="big"))
        digest.update(field)
    bucket = int.from_bytes(digest.digest(), byteorder="big") % bucket_total
    return RolloutDecision(
        bucket=bucket,
        included=range_start <= bucket < range_start + range_size,
    )
```

## Example

```python
decision = evaluate_rollout(
    "reader-42",
    namespace="homepage-v1",
    start=0,
    size=2_500,
)
repeated = evaluate_rollout(
    "reader-42",
    namespace="homepage-v1",
    start=0,
    size=2_500,
)
another_namespace = evaluate_rollout(
    "reader-42",
    namespace="search-v1",
    start=0,
    size=2_500,
)
single_bucket = evaluate_rollout(
    "reader-42",
    namespace="homepage-v1",
    start=decision.bucket,
    size=1,
)
zero = evaluate_rollout(
    "reader-42",
    namespace="homepage-v1",
    start=0,
    size=0,
)
full = evaluate_rollout(
    "reader-42",
    namespace="homepage-v1",
    start=0,
    size=10_000,
)
framed_buckets = (
    evaluate_rollout("c", namespace="ab", start=0, size=0).bucket,
    evaluate_rollout("bc", namespace="a", start=0, size=0).bucket,
)
unicode_buckets = (
    evaluate_rollout("\u00e9", namespace="unicode", start=0, size=0).bucket,
    evaluate_rollout("e\u0301", namespace="unicode", start=0, size=0).bucket,
)

assert (
    decision,
    repeated,
    another_namespace.bucket,
    framed_buckets,
    unicode_buckets,
    single_bucket.included,
    zero.included,
    full.included,
) == (
    RolloutDecision(bucket=5_314, included=False),
    RolloutDecision(bucket=5_314, included=False),
    2_237,
    (3_146, 123),
    (6_395, 4_887),
    True,
    False,
    True,
)
```

## Trade-offs and Limitations

Runtime is linear in the bounded UTF-8 input size and result storage is
constant. The fixed digest produces a stable 64-bit value, but modulo reduction
has a tiny bias whenever `bucket_count` does not divide `2**64`. Different
namespaces provide approximate cohort independence, not a mathematical
guarantee.

Exact encoded bytes define identity, so composed and decomposed Unicode forms
remain different unless the caller deliberately normalizes them. Changing the
encoding, framing, digest settings, namespace, subject, or bucket count remaps
subjects. Changing `start` moves the cohort; only increasing `size` with the
same start is monotonic. A public digest does not anonymize identifiers and is
predictable by anyone with the input.

## Related Snippets

<!-- catalog:related:start -->
- [Assign Stable Schedule Slots with a Digest](../reliability-resilience/assign-stable-schedule-slots-with-a-digest.md)
- [Derive a Versioned Record Key from Explicit Identity Fields](derive-a-versioned-record-key-from-explicit-identity-fields.md)
- [Map Keys with an Immutable Consistent Hash Ring](../algorithms-data-structures/map-keys-with-an-immutable-consistent-hash-ring.md)
<!-- catalog:related:end -->

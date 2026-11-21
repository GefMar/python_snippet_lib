---
title: "Run Trusted Doctest Examples with an Explicit Namespace"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - extract-bounded-doctest-examples-without-executing-them.md
  - compare-a-bounded-text-capture-against-a-golden-fixture.md
---

# Run Trusted Doctest Examples with an Explicit Namespace

## Idea and Problem

Verify a trusted in-memory Python transcript against named dependencies and capture any mismatch for a test report.

`DocTestParser` recognizes interactive prompts and expected output. Give the
parsed test an explicit namespace, then run it with `DocTestRunner`. The
examples share that test's namespace with each other, and an output callback
collects the runner's diagnostics.

## When to Use

Use this when you already have a short, reviewed transcript and the functions
it should demonstrate. A test can run it without searching modules or files.
Use static doctest extraction when you only need the prompt and output text.

## Implementation

```python
import doctest


def run_doc_examples(
    transcript: str, namespace: dict[str, object], *, name: str
) -> tuple[doctest.TestResults, str]:
    test = doctest.DocTestParser().get_doctest(transcript, namespace, name, None, 0)
    fragments: list[str] = []
    result = doctest.DocTestRunner(verbose=False).run(test, out=fragments.append)
    return result, "".join(fragments)
```

## Example

```python
def double(value: int) -> int:
    return value * 2


namespace: dict[str, object] = {"scale": double}
transcript = """\
>>> subtotal = scale(3)
>>> subtotal
6
>>> scale(5)
9
"""

result, diagnostics = run_doc_examples(transcript, namespace, name="scaling")

assert (result.attempted, result.failed) == (3, 1)
assert namespace == {"scale": double}
assert "scale(5)" in diagnostics and "Expected:" in diagnostics and "Got:" in diagnostics
```

## Trade-offs and Limitations

Doctest executes Python code with the test process's permissions. The
namespace is a shallow copy: a new binding such as `subtotal` stays in the
test, but code could still mutate objects shared with the caller. Use trusted
examples and keep their effects small enough for a controlled test.

The normal runner compares interactive output as text and temporarily redirects
standard output. Use it in a controlled, single-threaded test. Keep transcripts
short and expected output stable, and avoid emitting sensitive values into
the diagnostics.

## Related Snippets

<!-- catalog:related:start -->
- [Extract Bounded Doctest Examples Without Executing Them](extract-bounded-doctest-examples-without-executing-them.md)
- [Compare a Bounded Text Capture Against a Golden Fixture](compare-a-bounded-text-capture-against-a-golden-fixture.md)
<!-- catalog:related:end -->

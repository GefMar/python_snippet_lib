# Testing and Tooling

The primary consumer is a test/build/developer workflow and the taught behavior
is how that workflow is constructed or validated.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Compare a Bounded Text Capture Against a Golden Fixture](compare-a-bounded-text-capture-against-a-golden-fixture.md) | testing-technique | testing, validation | Compare an already captured exit status and text streams with an exact golden record while keeping mismatch diagnostics deterministic and bounded. |
| [Extract Bounded Native-Test Failure Highlights](extract-bounded-native-test-failure-highlights.md) | recipe | observability, parsing, testing | Extract a small, attributed set of diagnostic lines from already bounded native-test stdout and stderr without treating the result as a complete report. |
| [Parse a Bounded Space-Indented Test Outline into Leaf Paths](parse-a-bounded-space-indented-test-outline-into-leaf-paths.md) | algorithm | parsing, testing, validation | Turn one strict space-indented test listing into immutable leaf paths without silently repairing malformed hierarchy. |
| [Verify Ordered HTTP Client Expectations with Bounded Mismatch Reports](verify-ordered-http-client-expectations-with-bounded-mismatch-reports.md) | testing-technique | networking, testing, validation | Test an HTTP client through a finite in-memory transport that consumes only exact ordered expectations and reports mismatches without echoing request values. |
| [Wait for Named Queue Conditions Under One Monotonic Deadline](wait-for-named-queue-conditions-under-one-monotonic-deadline.md) | testing-technique | concurrency-control, testing, validation | Wait for several order-independent message conditions while sharing one monotonic timeout and a finite consumption budget. |
<!-- catalog:category:end -->

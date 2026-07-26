# Observability and Operations

Telemetry, logging, metrics, monitoring or operational automation is the
primary behavior.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Count Values in Fixed Upper-Bound Bins](count-values-in-fixed-upper-bound-bins.md) | algorithm | data-transformation, observability | Count finite observations in stable right-closed bins defined by a strictly increasing sequence of upper bounds. |
| [Format Log Records as JSON with Explicit Extra Fields](format-log-records-as-json-with-explicit-extra-fields.md) | recipe | observability, serialization | Serialize each log record to one strict JSON object while keeping caller-supplied extra fields separate from standard logging metadata. |
| [Scope Structured Log Fields with Context Variables](scope-structured-log-fields-with-context-variables.md) | pattern | concurrency-control, observability | Attach nested structured fields to log events without passing the same identifiers through every function call. |
<!-- catalog:category:end -->

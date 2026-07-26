# Networking and Protocols

Wire format, transport, HTTP/RPC/socket behavior or protocol-client semantics
determine correctness.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Iterate Cursor-Paginated Results Lazily](iterate-cursor-paginated-results-lazily.md) | pattern | networking, resource-management | Hide cursor pagination behind an iterator while keeping page fetches lazy, ordered, and bounded by explicit safety checks. |
| [Read and Write Size-Capped Varint Frames](read-and-write-size-capped-varint-frames.md) | recipe | networking, serialization, validation | Frame byte payloads on a blocking stream with a canonical unsigned LEB128 length while rejecting oversized declarations before reading their bodies. |
<!-- catalog:category:end -->

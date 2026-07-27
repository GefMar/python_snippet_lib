# Security and Privacy

A threat model, cryptographic, authentication, authorization, redaction,
privacy or hostile-input invariant is central.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Redact a Printable ASCII Secret with a Bounded Visible Tail](redact-a-printable-ascii-secret-with-a-bounded-visible-tail.md) | recipe | observability, security, validation | Mask every character of a short secret and reveal only a bounded suffix of longer values while keeping at least eight characters hidden. |
| [Validate a Conservative Unicode Filename Component](validate-a-conservative-unicode-filename-component.md) | recipe | interoperability, validation | Normalize and validate one user-facing Unicode filename component without treating it as an authorized storage path. |
| [Verify an RFC 7636 S256 PKCE Challenge](verify-an-rfc-7636-s256-pkce-challenge.md) | recipe | interoperability, security, validation | Validate an OAuth PKCE verifier and compare its RFC 7636 S256 transformation with an exact unpadded challenge. |
<!-- catalog:category:end -->

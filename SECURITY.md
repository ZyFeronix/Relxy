# Security Policy

## Reporting a vulnerability

Please do **not** open a public issue for a security problem. Use GitHub's private
vulnerability reporting instead:

<https://github.com/ZyFeronix/Relxy/security/advisories/new>

Include what is affected (endpoint, version or commit), a reproduction or a sketch of
one, the impact you believe it has, and whether an unauthenticated client can trigger
it. Acknowledgement is best effort — this is a small project without a paid on-call
rotation — but credible reports are taken seriously and will be credited unless you
prefer otherwise.

## Scope

In scope, for this repository:

- Bypassing manifest signature verification or the `app`, `channel` and `expires` checks.
- Leaking the GitHub credential, the hook token, or a signed asset URL through a
  response, log line, error message or redirect.
- Cache poisoning: serving bytes that do not match the signed `sha256` and `size`.
- Cross-tenant confusion: reaching one product's repository, keys or limits from
  another product's domain.
- SSRF through the hook, a redirect, or any client-supplied value.
- Amplification: making a single request cost unbounded upstream calls or disk.
- Host-header handling that lets an unmapped host reach a tenant.

Out of scope:

- Compromise of GitHub itself, or of a product repository or its signing key.
- A client that ignores signature verification. The design assumes the client
  verifies; that is a client bug.
- Denial of service that needs traffic well beyond the documented rate limits and
  capacity caps.
- Missing hardening of a deployment that deviates from the documented example
  (`docs/PLAN.md` §15).

## Known residual risk

A compromised Relxy host can serve an older, validly signed release until that
release's signed `expires` timestamp passes. This is by design and is documented in
`docs/PLAN.md` §5.6 and §14; it is not a reportable vulnerability on its own. A
deployment that needs a stronger guarantee must shorten the signing window.

## Supported versions

The service is pre-1.0 and unreleased. Only `main` is supported; there are no
maintenance branches. Signing-key rotation and token rotation are described in
`docs/PLAN.md` §5.8 and §6.1.

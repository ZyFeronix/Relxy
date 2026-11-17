# Relxy

Release proxy for ZyFeronix apps. Relxy brokers private GitHub Releases to clients
that must never hold a GitHub credential: it verifies and serves signed update
manifests, and streams release artifacts from a local content-addressed cache over
one domain per product.

> **Status: design phase.** This repository holds the design document and project
> scaffolding. There is no buildable code yet — the implementation plan and its
> phases live in [`docs/PLAN.md`](docs/PLAN.md).

## The problem

Release artifacts for ZyFeronix products live in private GitHub repositories. A
desktop updater cannot authenticate against GitHub without shipping a credential to
every installation, and a leaked update credential is a repository compromise.
GitHub's release API is also not a CDN: downloads are not resumable, and there is no
way to keep one product's traffic from affecting another's.

## What Relxy does

- Serves a **signed manifest** per channel and target. The signature is produced by
  the product's release pipeline, never by Relxy.
- Serves **artifacts by sha256**, so a URL names exact bytes: caches are immutable,
  downloads are resumable with `Range`, and a release landing mid-download cannot
  cause a mismatch.
- Keeps a **local cache** of retained releases, so a GitHub outage does not stop
  updates and GitHub's API rate limit is not spent on client traffic.
- Is **multi-tenant by `Host`**: one domain, one product, one repository, one
  credential set, one signing-key set. Unknown hosts get a 404 and no upstream call.
- Accepts an optional **release hook** from the product pipeline, so a new release is
  picked up immediately instead of on the next poll.

## API surface

| Path | Methods | Purpose |
|---|---|---|
| `/v1/{channel}/{platform}/{variant}/manifest.json` | `GET`, `HEAD` | Signed manifest envelope, with ETag and 304 |
| `/v1/{channel}/{platform}/{variant}/download/{sha256}` | `GET`, `HEAD` | Artifact bytes, `Range`-capable, immutable caching |
| `/v1/hooks/release` | `POST` | "Go look at GitHub" notification from the pipeline |
| `/healthz` | `GET` | Public minimal health probe |

Channels are `latest` and `beta`; platforms are `win32` and `linux`; variants are
`portable` and `installed`. Operational endpoints (`/version`, `/metrics`, detailed
health) are served on a separate loopback-only listener.

[`docs/PLAN.md` §3](docs/PLAN.md) is the normative contract. This table is a summary
and must not be treated as the specification.

## Security model

Every manifest is Ed25519-signed by the product pipeline and verified by both the
proxy and the client. The running service never holds a signing key, and clients
never hold a GitHub credential: signature verification and content addressing are the
security boundary, while the proxy's own verification is defence in depth. A fully
compromised Relxy can cause a denial of service, or replay an older validly signed
release until that release's signed `expires` passes; it cannot forge a new one. The
full threat model is in [`docs/PLAN.md` §14](docs/PLAN.md).

## Repository layout

Planned; directories appear as their phase lands (`docs/PLAN.md` §18).

```text
cmd/relxy/          entry point and subcommands (serve, keygen, sign, verify, hook-token, version)
internal/config/    configuration loading, strict validation, secret wrapper
internal/tenant/    Host to tenant registry
internal/httpapi/   routing, middleware, handlers, JSON errors
internal/github/    GitHub API client, credentials, budget guard, circuit breaker
internal/manifest/  schema validation, canonical JSON, signature verification
internal/cache/     content-addressed store, single-flight fills, retention
internal/refresh/   per-tenant refresher, hook handling, snapshots
internal/limit/     rate limiting and client IP resolution
internal/metrics/   Prometheus text exposition
deploy/             Dockerfile, systemd unit, Caddy example
docs/               design documents
```

## Documentation

- [`docs/PLAN.md`](docs/PLAN.md) — design and implementation plan, the source of truth.
- [`CHANGELOG.md`](CHANGELOG.md) — notable changes.

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md). Security reports go through
[`SECURITY.md`](SECURITY.md), never a public issue.

## License

No license has been chosen yet. Until one is added, all rights are reserved and this
code may not be reused, modified or redistributed.

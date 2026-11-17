# Relxy

A self-hosted release proxy for private GitHub repositories. Clients ask Relxy for an
update manifest and download the artifacts named in it, so they never talk to GitHub
and never hold a GitHub credential.

> **Status: design phase.** Nothing is built yet — there is no code, no binary and no
> configuration to run. The design and the implementation plan live in
> [`docs/PLAN.md`](docs/PLAN.md).

## The problem

Release artifacts often live in private repositories, but the clients that need them
cannot authenticate against GitHub without shipping a credential to every
installation. A leaked update credential is a repository compromise, and GitHub's
release API is not a CDN: downloads are not resumable, and one client's traffic cannot
be isolated from another's.

Relxy puts a server you control between those clients and GitHub, so the GitHub
credential stays there instead of on every machine you ship to.

## Documentation

- [`docs/PLAN.md`](docs/PLAN.md) — design, constraints and implementation plan.
- [`CHANGELOG.md`](CHANGELOG.md) — notable changes.

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md). Security reports go through
[`SECURITY.md`](SECURITY.md), never a public issue.

## License

No license has been chosen yet. Until one is added, all rights are reserved and this
code may not be reused, modified or redistributed.

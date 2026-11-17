# Contributing

Relxy is a small, single-purpose service. This page covers the conventions that keep
changes reviewable.

## Scope

In scope: the release proxy in this repository — its API, cache, refresher,
configuration, deployment files and documentation.

Out of scope, and not to be added here (`docs/PLAN.md` §20): consumer applications
and their updaters or installers, Electron and builder configuration in the product
repositories, product CI pipelines, and any web UI, database, accounts or analytics.

## Language

Code, comments, identifiers, commit messages, documentation, issues and pull requests
are all English.

## Workflow

1. Branch from `main`.
2. Keep one topic per branch.
3. Open a pull request describing what changed, why, and how it was verified.
4. Be accurate about what was and was not tested.

Direct commits to `main` are fine for trivial documentation fixes.

Commit messages follow [Conventional Commits](https://www.conventionalcommits.org/):
`feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `build:`, `chore:`. The summary is
imperative and lower-case.

## Before opening a pull request

Once there is code, run and report these:

```sh
gofmt -l .          # must print nothing
go vet ./...
go test ./...
```

`go test -race ./...` is required too, but `-race` needs CGO, which is disabled on the
Windows development machine (`CGO_ENABLED=0`). Run it in WSL or Docker and paste the
result, or state clearly that it was not run.

Tests must not reach the network: the GitHub API is faked with `httptest`.

## Secrets

- Never commit a GitHub token, a private key, a hook token, or a real domain list.
- Credentials come from the environment (`GITHUB_TOKEN_<PRODUCT>`), never from a
  committed file. `config.example.yml` holds shapes and placeholders only.
- The real `config.yml` is git-ignored.
- If a secret does reach a commit, rotate it first, then rewrite history — not the
  other way round.

## Changing the design

`docs/PLAN.md` is the source of truth. If a change touches the consumer contract
(§3), the manifest format (§5), or a non-negotiable property (§1), update the plan in
the same pull request. Do not silently diverge from it.

## Changelog

Add an entry under `## [Unreleased]` in [`CHANGELOG.md`](CHANGELOG.md) for anything a
client author or a service operator would notice.

## What

<!-- What changes, and why. Link the plan section or the issue. -->

## How it was verified

<!-- Commands actually run, and their result. Say plainly what was not tested. -->

## Checklist

- [ ] `gofmt -l .` prints nothing
- [ ] `go vet ./...` passes
- [ ] `go test ./...` passes
- [ ] `go test -race ./...` was run in WSL or Docker (or is explicitly reported as not run)
- [ ] No secrets, tokens, private keys, or real hosts and domains in the diff
- [ ] `docs/PLAN.md` updated if the consumer contract or manifest format changed
- [ ] `CHANGELOG.md` updated under `## [Unreleased]`

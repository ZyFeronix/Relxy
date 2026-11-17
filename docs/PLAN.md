# Relxy — implementation plan (handoff document)

> **Audience:** the engineer/agent who will implement this. You were not in the planning conversation, so this
> file is written to be self-contained. Where it says "decided", the owner already chose; do not re-open it.
> Where it says "default", pick that unless the owner says otherwise. Where something is genuinely unspecified,
> it is listed in section 21 — ask, do not guess.
>
> **Scope: ONLY this service.** Do not touch any consumer app, installer, updater, Electron config or app CI.
> Those come later against the contract defined here.
>
> **Process:** commit this file as `docs/PLAN.md` first, get the owner's agreement, then implement in the
> phases of section 18. Follow the org's git rules (feature branch, merge request, never push to main, no secrets
> in commits). Everything in code and docs is English.

---

## 0. Glossary (terms used precisely in this document)

| Term | Meaning |
|---|---|
| **Tenant / product** | One distributable application (e.g. Woom!Toon). One tenant = one domain list = one GitHub repo = one credential set = one signing-key set = one cache namespace. |
| **Pipeline** | The product repo's GitHub Actions release workflow. It builds artifacts, writes + signs the manifest, publishes the GitHub release, then optionally calls the hook. It is NOT part of this service. |
| **Manifest** | JSON document (schema 1) describing the release: version, notes, and per platform/variant: file, sha256, size, kind, url. Signed by the pipeline. |
| **Envelope** | What this service serves at `manifest.json`: `{"manifest": <raw>, "sig": <raw>}`. |
| **Target** | One (platform, variant) pair, e.g. `win32/portable`. |
| **Channel** | `latest` (stable releases) or `beta` (prereleases). |
| **Object** | One cached artifact file on disk, addressed by its sha256. |
| **Index** | In-memory (rebuildable) map of the retained verified releases per channel and `sha256 → artifact reference`. |
| **Snapshot** | An immutable struct holding a tenant's current index + published envelope; replaced atomically on refresh. |
| **Retained** | Among the N newest verified releases per channel (default 4 latest / 8 beta); their objects are kept on disk. |
| **Prewarm** | Downloading a new release's artifacts into the cache before clients are told about it. |
| **Fill** | One upstream download that populates one object. |
| **Hook** | `POST /v1/hooks/release`, a notification from the pipeline: "go look at GitHub". It carries hints, never content. |
| **Stale** | A previously verified manifest served while GitHub is failing (bounded to 6 h). |
| **LKG** | Last known good snapshot. |
| **Token bucket** | Rate limiter with a burst size (capacity) and a steady refill rate (10.1). |
| **Sliding window counter** | Rate limiter that estimates the rate from the previous and current window counts, avoiding the boundary burst of a fixed window (10.1). |
| **Fail open** | If the limiter itself breaks, allow the request instead of blocking everyone. |

`/healthz` ends in "z" purely by convention (Google/Kubernetes style; the suffix avoids clashing with
application routes). The name is arbitrary; keep `/healthz`.

---

## 1. Goal, design stance, constraints

**What:** a small, single-binary, multi-tenant HTTP service that distributes application releases from PRIVATE
GitHub repositories. Clients (desktop apps, CLI tools) never talk to GitHub and never hold a credential: they ask
this service for a manifest, verify its signature themselves, and download the artifact through this service.

**One-line repo description:** "A simple proxy for distribution of the releases of the ZyFeronix products and
applications."

**Design stance:** Relxy is a dumb, replaceable file server for release bytes. A full compromise of it may
cause denial of service, or a *rollback to an older validly signed release* (bounded by `expires`, section 5.6).
It must not be able to deliver a manifest or artifact that a client would accept as new and malicious. The
client's signature verification is the real security boundary; the proxy's verification is defense in depth.

**Hard constraints from the owner**
- **No outside services.** Only GitHub (as the source) and, if truly needed later, free Cloudflare features.
  No database, message broker, Redis, object storage, SaaS metrics, Kubernetes.
- **As lightweight as possible:** one static binary, stdlib-first, minimal dependencies. Target: a YAML parser and
  nothing else mandatory. Every extra dependency must be justified in the MR description.
- One small VPS. Must run either as a container or directly on the host under systemd (ship both).
- Cloudflare is optional; the service must work with Caddy directly on the internet.
- The owner is a software engineer new to Go: favor boring, idiomatic, well-commented code over cleverness; keep
  doc comments short (one-line summary, say what/why, not how).

**Non-negotiable properties**
- N1. The GitHub credential lives only server-side (env var/file), never in the repo, an error message or a `%+v` dump.
- N2. Private repositories are the first-class case.
- N3. Clients receive zero credentials; nothing in any response can be replayed against GitHub.
- N4. Multi-tenant by request `Host`. Unknown Host → 404 JSON `unknown_tenant`, no default tenant, **no upstream
  call**. The repository is never accepted from path, query, header or body. (Sole exception: public `/healthz`,
  section 3.2, which is host-agnostic and reveals nothing.)
- N5. Manifests are Ed25519-signed by the pipeline. The running service never signs and never holds a private key.
- N6. Downloads are resumable (HTTP Range); "up to date" is distinguishable from "error" without parsing prose.
- N7. Cheap to run (see constraints).

---

## 2. Architecture overview

```
 Product repo (GitHub Actions)                         Relxy host (one VPS)
 ┌───────────────────────────────┐                    ┌──────────────────────────────────────────────┐
 │ build (Electron tooling)      │                    │  Caddy :443  (TLS, HTTP/2)                    │
 │ write update-manifest.json    │                    │     │  reverse_proxy                          │
 │ relxy sign → .sig      │  gh release create │     ▼                                         │
 │ publish GitHub release+assets │───────────────┐    │  relxy  127.0.0.1:8080  (public API)   │
 │ curl POST /v1/hooks/release   │──────────────┐│    │                127.0.0.1:9090  (admin/ops)    │
 └───────────────────────────────┘              ││    │   ├─ tenant resolver (Host → tenant)          │
                                                ││    │   ├─ per-tenant Snapshot (index+envelope)     │
 Clients (desktop apps)                         ││    │   ├─ refresher (hook + 15 min poll + backoff) │
 ┌──────────────┐  GET manifest.json            ││    │   ├─ cache (disk objects, single-flight fill) │
 │ updater      │──────────────────────────────►││    │   ├─ rate limiters (4 budgets)               │
 │ (verifies    │  GET download/{sha256} (Range)││    │   └─ GitHub client (API + blob, token guard)  │
 │  signature)  │◄──────────────────────────────┘│    └───────────────┬──────────────────────────────┘
 └──────────────┘                                │                    │ HTTPS (token only on api.github.com)
                                                 ▼                    ▼
                                          GitHub (private repo)  release-assets.githubusercontent.com
```

Three activities that must never be confused:

1. **Client request path** (`manifest.json`, `download/{sha}`): answered from memory/disk. It calls GitHub only
   in two bounded cases: cold start with nothing cached, and a download for an unknown sha256 (section 8.5).
2. **Refresh path** (background): hook-triggered and 15-minute poll; fetch → validate → verify → prewarm → publish.
3. **Operator path** (admin listener, CLI subcommands): health, metrics, keygen, sign, hook-token.

---

## 3. Consumer contract (fixed here; later clients depend on it — do not "improve" it)

### 3.1 Facts about the consumers
- App versions are NOT semver. Current format has 4 segments: `YYYY.M.D.HHmm`, e.g. `2026.11.13.1319`. Treat the
  version as an **opaque string**: never parse it, never sort with it, validate only "non-empty printable
  characters, ≤ 64 chars". The client does the numeric comparison. "Newest release" is decided by GitHub
  `published_at`, never by version.
- Platforms: `win32`, `linux`. Variants: `portable`, `installed`. Anything else → 404 `unsupported_target`.
- Channels: `latest` (stable) and `beta` (prerelease). A `dev` channel must not exist (config validation rejects it).
  Channel is a path segment; an unknown channel → 404 `unsupported_target`.
- Artifact kinds: `zip`, `tar.gz`, `nsis`, and `deb` (provisional for `linux/installed`). Keep ONE table in code
  mapping (platform, variant) → allowed kinds, and (kind) → `Content-Type`, so adding a kind is a one-line change:

  | kind | Content-Type |
  |---|---|
  | zip | application/zip |
  | tar.gz | application/gzip |
  | nsis | application/vnd.microsoft.portable-executable (installer .exe) |
  | deb | application/vnd.debian.binary-package |

  Initial matrix: `win32/portable=zip`, `win32/installed=nsis`, `linux/portable=tar.gz`, `linux/installed=deb`.
- Release notes must be servable in-app (the repo is private; there is no public page): notes live in the
  manifest as markdown. Client contract: sanitize rendered markdown.
- Artifacts are 100–250 MiB. Assume flaky connections and mobile tethering.

### 3.2 Endpoints (per-tenant domain; the tenant is the Host)

```
GET  /v1/{channel}/{platform}/{variant}/manifest.json
       200 application/json   the signed envelope (section 5.7)
       ETag: "<sha256 of the envelope bytes>"; If-None-Match → 304 with NO upstream call
       Cache-Control: public, max-age=300, must-revalidate
GET  /v1/{channel}/{platform}/{variant}/download/{sha256}
       200 artifact bytes, or 206 with Range / If-Range support
       Content-Length, Content-Type (3.1 table), ETag: "<sha256>", Accept-Ranges: bytes
       Cache-Control: public, max-age=31536000, immutable
       {sha256}: 64 lowercase hex chars; MUST be the sha256 of THIS (platform, variant) target in a verified
       release the proxy retains/indexes (section 8.5); otherwise 404 no_release.
HEAD on both endpoints.
POST /v1/hooks/release      (section 7.3) 202 when authenticated
```

**`GET /healthz` is public** (decided: external monitors must reach it) and is the single exception to "unknown Host
→ 404": it is host-agnostic and never touches upstream. Response is minimal and leaks nothing: `200 {"status":"ok"}`
or `503 {"status":"degraded"}` — no version, no tenant names, no ages, no paths. Semantics: 200 when config is loaded
and the cache dir is writable (probe cached ~30 s); if the Host matches a tenant, it is also 503 when that tenant has
had no successful refresh for longer than `stale_max_age` (so a monitor on `woomtoon-updates.…/healthz` reports that
product's freshness). It is exempt from the rate-limit budgets (cheap, in-memory answer) and not counted as a tenant
request in metrics beyond a plain counter.

All other operational endpoints are NOT on the public listener. They are on `admin_listen` (default
`127.0.0.1:9090`), never reachable through tenant domains: `GET /version` (build version + commit), `GET /metrics`,
and `GET /healthz?detail=1` (the detailed per-tenant view of section 13).

Why content-addressed downloads: the artifact URL names the exact bytes, so `immutable` caching is valid, a new
release landing between a client's manifest fetch and its download cannot cause a mismatch, and resume is safe.

Notes that surprise people:
- Every `.../{platform}/{variant}/manifest.json` of a channel returns the **same envelope**: the signature covers
  the whole manifest, so the proxy cannot filter it. The platform/variant in the path only decides the 404
  `unsupported_target` for a retired target.
- `manifest.targets[p][v].url` is exactly the download path `/v1/{channel}/{p}/{v}/download/{sha256}`; the proxy
  validates this (section 5.3). The signed document carries no other URL.

### 3.3 Errors
Always JSON, `Content-Type: application/json`, `Cache-Control: no-store`, stable machine code, a short
credential-free English sentence, never a stack trace, never an upstream body:

```json
{ "code": "no_release", "message": "No release is available for this channel." }
```

| code | HTTP | when |
|---|---|---|
| unknown_tenant | 404 | Host not mapped (body reveals nothing about tenants) |
| not_found | 404 | unknown route |
| method_not_allowed | 405 | wrong method (include `Allow`) |
| unsupported_target | 404 | bad channel/platform/variant, or target absent from the manifest |
| no_release | 404 | no release for the channel (incl. `beta` with no prerelease), or sha256 not in the index |
| range_not_satisfiable | 416 | bad Range |
| rate_limited | 429 + `Retry-After` | per-IP metadata or concurrent-download budget exhausted |
| capacity | 503 + `Retry-After` | tenant-wide concurrency/egress cap, GitHub circuit open, or disk full |
| upstream_unavailable | 502 | GitHub unreachable/5xx and no usable stale data |
| manifest_invalid | 502 | any validation / signature / digest / expiry failure (document never echoed) |
| unauthorized | 401 | missing/wrong hook token |
| internal | 500 | anything else |

A compact random request id (8 random bytes, hex) is returned in `X-Request-Id` for client-side correlation. It never
encodes IPs or tokens. An incoming `X-Request-Id` is ignored. Never emit a `Server` header or version banner.
Every response gets `X-Content-Type-Options: nosniff`.

---

## 4. Tenancy and domains

- **One product = one tenant = one domain list = one repo = one credential set = one key set.** Domain → tenant is
  the only mapping.
- A product is identified by two independent things: the **domain** it talks to and the **`app` field in the signed
  manifest** (the client checks `manifest.app` equals its compiled-in product id). Use a **different Ed25519 key
  per product** so product A's manifest cannot validate for product B even if a client is pointed at the wrong
  domain.
- Why separate domains (owner's reasoning): per-domain Cloudflare rules; and limiting one product must never block
  updates of another product installed on the same machine. Therefore **every app-level limiter is keyed by
  (tenant, client IP)**, never by IP alone, and nothing is shared between tenants except disk space.
- Host normalization: lowercase; strip port and trailing dot; exact match against the tenant's `hosts`. Only the
  `Host` header as received from the reverse proxy is used; `X-Forwarded-Host` and friends are ignored. Caddy must be
  configured to serve only the mapped domains (no on-demand TLS).
- Domains are configuration only. The real domains are not known yet; use placeholders such as
  `woomtoon-updates.example.com`. **If Cloudflare proxying (orange cloud) is used on the free plan**, note that its
  free Universal certificate covers one label under the apex, so prefer `woomtoon-updates.<apex>` over
  `woomtoon.updates.<apex>`.

---

## 5. Manifest and signature

### 5.1 What the pipeline publishes
Two assets on each GitHub release (names fixed, matched against `allow_assets`):
- `update-manifest.json` — the manifest.
- `update-manifest.sig` — `{"sigs":[{"key_id":"<short id>","sig":"<base64url unpadded, 64-byte Ed25519>"}]}`.
  A single-object form `{"key_id":…,"sig":…}` is also accepted. The proxy accepts the manifest if ANY listed
  signature verifies against a key in the tenant's allow-list (key-rotation overlap).

Signature input (UTF-8 bytes): `"relxy-manifest-v1\n" + JCS(manifest)` where JCS is RFC 8785 canonical JSON.
Because the signature is over the canonical form, the uploaded file's key order/whitespace do not matter.

### 5.2 Manifest schema 1

```json
{
  "schema": 1,
  "app": "woomtoon",
  "channel": "latest",
  "version": "2026.11.13.1319",
  "published": "2026-11-13T13:19:00Z",
  "expires": "2027-02-13T13:19:00Z",
  "notes": "## What's new\n- …",
  "targets": {
    "win32": {
      "portable":  { "file": "WoomToon-2026.11.13.1319.zip",       "sha256": "<64 hex>", "size": 123456789,
                     "kind": "zip",  "url": "/v1/latest/win32/portable/download/<sha256>" },
      "installed": { "file": "WoomToon-2026.11.13.1319-Setup.exe", "sha256": "<64 hex>", "size": 123456789,
                     "kind": "nsis", "url": "/v1/latest/win32/installed/download/<sha256>" }
    },
    "linux": { "portable": { … "kind": "tar.gz" … }, "installed": { … "kind": "deb" … } }
  }
}
```

Numbers: `schema` and `size` are integers (`size` ≤ 2^53). Reject any non-integer/exponent number anywhere in the
manifest; this makes canonicalization of numbers trivial (decimal integer).

### 5.3 Validation (FAIL-CLOSED — any failure ⇒ the release is invalid; the document is never echoed)
1. JSON parse with a decoder that uses `UseNumber` and **rejects duplicate object keys** (parser differentials are a
   real attack on signed JSON). Size cap: manifest ≤ 1 MiB; `notes` ≤ 64 KiB.
2. `schema == 1`; `app == tenant.app`; `channel == requested channel`; `version` is a non-empty printable string
   ≤ 64 chars (nothing else is checked; never parse it); `published` and `expires` parse as RFC 3339 UTC.
3. The GitHub release's `prerelease` flag is consistent with `channel` (`beta` ⇒ prerelease true, `latest` ⇒ false).
4. Every target present has: allowed (platform, variant), `kind` allowed for that pair (table 3.1), `sha256` =
   64 lowercase hex, `size` > 0, `file` non-empty and matching a pattern in `allow_assets`, **and an asset with
   exactly that name exists in the SAME GitHub release** with GitHub-reported `size` equal to `size` and state
   `uploaded`. `url` equals the canonical download path for the target (`/v1/{channel}/{p}/{v}/download/{sha256}`).
5. Signature verifies (5.4). Unknown `key_id` ⇒ invalid.
6. `expires` is in the future at refresh time and at serve time (an expired current manifest is not served).
7. An absent target is not a failure: it makes that target return 404 `unsupported_target` (how a variant is
   retired). At least one target must exist.

### 5.4 Signature verification
- Parse `update-manifest.sig`; decode base64url with `base64.RawURLEncoding` (unpadded; reject padded input).
- Canonicalize a *parsed copy* of the manifest with the in-house JCS function (5.5); prepend the domain-separation
  prefix; `ed25519.Verify` against each configured key whose `key_id` matches.
- `--allow-unsigned` (flag, never config) skips only the signature check. Default deny. Used by tests only.
- Public key format in config: base64url unpadded, 32 bytes (43 chars). Signature: 64 bytes (86 chars).

### 5.5 JCS (RFC 8785) — implement in-house, ~80 lines, no dependency
Restricted to the manifest's value types (objects, arrays, strings, integers, booleans, null):
- Object keys sorted by **UTF-16 code unit order** (NOT Go's byte order; they differ for characters outside the BMP
  — use `unicode/utf16` for the comparison).
- Strings: escape only `"`, `\`, and control characters < U+0020 (use `\b \f \n \r \t` short forms, others `\u00xx`
  lowercase hex); everything else, including non-ASCII, is emitted as raw UTF-8.
- Integers as plain decimal; no whitespace anywhere.
- Test with the RFC 8785 examples/test vectors and with a Unicode-key ordering case. The same function is used by
  the verifier and by the `sign` subcommand, so signer and verifier cannot disagree.

### 5.6 Replay / rollback protection (decided)
`expires` is part of the signed manifest. The proxy never serves an expired manifest. **Client contract:** reject if
`expires` is in the past, and reject a `published` older than the last manifest it accepted. Residual risk to
document in the README: a compromised proxy can still serve an older signed release until its `expires`; the
pipeline chooses the window (suggest 30–90 days; a fresh release re-signs).

### 5.7 The envelope
Embed the verified asset bytes verbatim: `{"manifest": <update-manifest.json bytes>, "sig": <update-manifest.sig
bytes>}` using `json.RawMessage`. **Never re-serialize or reorder the manifest, never mutate numbers.** The client
re-canonicalizes to verify. ETag = sha256 of the envelope bytes (strong, quoted). The upstream (GitHub) ETag is a
different thing and is stored separately for conditional requests.

### 5.8 Key management (decided: generated by our Go program, but NOT held by the server)
The owner first asked for the proxy to auto-generate keys. A key the server generates and holds would let a
compromised server forge updates, defeating the design. Therefore the same binary offers one-shot CLI subcommands
(section 12) that a human or the pipeline runs; the **running service has no code path that reads a private key**
(a test asserts the service rejects a config containing one).
- `relxy keygen --key-id 2026a` → prints a fresh Ed25519 pair once. Private key → pipeline secret store
  (GitHub Actions secret of the product repo). Public key → tenant `signing_keys` in config AND the product's
  updater (client).
- `relxy sign` signs a manifest file (section 12). Rotation: generate `2027a`, add its public key beside
  `2026a`, start signing with the new key (optionally dual-sign), ship clients that know both, later remove the old.

### 5.9 Release selection and last-known-good
- Releases are listed newest-first by GitHub `published_at`; drafts are ignored. `latest` = non-prerelease only;
  `beta` = prerelease only. No prerelease ⇒ `beta` returns 404 `no_release` (decided; no fallback to stable).
- Walk the listing newest-first, validate (5.3) up to the first valid release for the channel; that is the
  published release. Invalid newer releases are skipped and counted in `manifest_invalid_total` — a bad
  new release must not break the fleet: keep serving the last good one (bounded by `expires` and the 6 h stale
  rule when the cause is upstream failure). If NO valid release exists for a channel and nothing is cached,
  `manifest.json` returns 502 `manifest_invalid` (or 404 `no_release` when there are simply no releases).
- Known consequence (tell the client team): since `beta` is prerelease-only, a client on `beta` never sees a newer
  stable release through the `beta` channel; the updater should consult both channels if beta users should also
  receive stable releases.

---

## 6. Upstream access (GitHub)

### 6.1 Credentials — read this section carefully (the owner is new to GitHub tokens)
GitHub has no exact equivalent of a GitLab service account/bot with project tokens. Options:

| Option | What it is | Pros | Cons |
|---|---|---|---|
| **A. Fine-grained PAT from a machine user (v1)** | Create a normal GitHub account used only as a bot; add it as a **read-only collaborator** on each private repo; create a *fine-grained personal access token* limited to the selected repo(s) with **Contents: Read** (Metadata: Read is implied). | Simple; closest to a GitLab bot + project token; least privilege per repo. | Expires (≤ ~1 year); must be rotated; may use a seat on a paid org; the org may need to approve fine-grained PATs. |
| B. PAT of a real person | Same token owned by an employee | Zero setup | Tied to a person. **Avoid.** |
| **C. GitHub App (v2, recommended long-term)** | An org-owned App with `Contents: read` installed on the repos. The service signs a short JWT (RS256, stdlib `crypto/rsa`) with the app private key and exchanges it for a 1-hour *installation token*, refreshed automatically. | No expiry chores, no seat, not tied to a person, auto-rotating, higher rate limits. | ~100 extra lines; one more secret (the private key). |

Plan: define a small `Credential` interface (`Token(ctx) (Secret, error)`, `MarkFailed(status)`), with a `pat`
implementation now and a `github_app` implementation later (interface and config shape prepared; not built in v1).
Multiple tokens per tenant are an **ordered list** (rotation without downtime: add the new token first, remove the
old later). Fail over to the next token on 401, or on 403 with exhausted rate limit. Note GitHub answers **404** (not
403) for a token that lacks access to a private repo: repeated 404 on a repo that previously worked ⇒ treat as a
credential problem (`upstream_auth_failures_total`). Optional config `expires: "<date>"` per token feeds a
`token_expires_in_seconds` gauge so operators can alert before expiry. Rate limit: 5,000 requests/hour per token;
a conditional request answered 304 is free, hence ETags are mandatory.

### 6.2 Calls (all built explicitly; no header passthrough; no client-controlled URL)
Fixed headers on API calls: `Accept: application/vnd.github+json`, `X-GitHub-Api-Version: 2022-11-28`, a fixed
`User-Agent: Relxy/<version>`, and `Authorization: Bearer <token>`.
- **List releases:** `GET /repos/{repo}/releases?per_page=N` (N = max over channels of `retain` + margin, ≤ 100) with
  `If-None-Match: <stored etag>`. 304 ⇒ nothing changed, no further calls.
- **Targeted fetch (hook hint):** `GET /repos/{repo}/releases/tags/{tag}` when the hook supplied a valid `tag`.
- **Manifest assets:** download `update-manifest.json` and `update-manifest.sig` of a release only when the release's
  asset ids changed since the last successful fetch (asset ids are immutable; a re-upload gets a new id).
- **Artifact bytes:** `GET /repos/{repo}/releases/assets/{id}` with `Accept: application/octet-stream` and the
  Authorization header. GitHub answers 302 with a signed `release-assets.githubusercontent.com` URL; follow it
  **server-side** (6.3). The asset id always comes from the fetched release JSON, located by asset *name*; it never
  comes from the manifest, the hook, or a client.
- Release JSON fields used: `id`, `tag_name`, `draft`, `prerelease`, `published_at`, `assets[].{id,name,size,state}`.

### 6.3 Redirect handling and token-leak prevention (must be proven by tests)
- Use **two separate `http.Client`s**: `apiClient` (sends the token, `CheckRedirect` returns `http.ErrUseLastResponse`
  so redirects are never followed automatically) and `blobClient` (never has a token by construction; used only for
  the signed URL, also without automatic redirects).
- Flow: `apiClient` requests the asset; on 302, read `Location`, validate it (`https` only, host matches the
  allow-list `*.githubusercontent.com`, no userinfo) and request it with `blobClient`. A test asserts the second hop
  carries no `Authorization` header and that a redirect to any other host is refused.
- Both clients: `Transport.Proxy = nil` (an `HTTP_PROXY` env var must not divert the token), explicit timeouts
  (dial, TLS handshake, response header), HTTP/2 allowed, bounded idle connections.
- The signed URL is never echoed or returned. Go's `*url.Error` embeds the URL in `Error()`: **scrub it**
  (replace with a generic error type) before it reaches any response or error message.
- Bound everything: max manifest/asset-metadata body sizes (`io.LimitReader`), idle-stream timeout on artifact
  transfers (abort if no bytes arrive for e.g. 60 s), overall fill timeout.

### 6.4 Budget guard and circuit breaker (per token)
Track `X-RateLimit-Remaining` / `X-RateLimit-Reset` from every response. When remaining drops below a low-water mark
(default 200), pause non-essential refreshes (poll, miss lookups) until reset; at 0 the circuit is open for GitHub
calls with that token until reset (`capacity` for client paths that would need it). Also open the circuit for a short
window after repeated 5xx/timeouts (e.g. 5 failures → 30 s, doubling to 5 min). Never let GitHub answer 403 for rate
limiting if this guard can prevent it.

---

## 7. State and refresh model

### 7.1 Snapshot
Each tenant holds an immutable `Snapshot` behind an `atomic.Pointer` (readers load it lock-free; the refresher builds
a new one and swaps it; no data races by construction):
```
Snapshot {
  channels: map[channel]ChannelState{
     published   *Release          // the release whose envelope is served now
     envelope    []byte; etag string
     releases    []*Release        // retained + index_depth verified releases, newest first
  }
  bySHA: map[sha256]ArtifactRef{ channel, platform, variant, releaseTag, assetName, assetID, size, kind }
  fetchedAt, lastGoodAt time.Time; releaseListETag string
}
```
The snapshot is rebuildable from GitHub at any time (cold start ⇒ refresh). Nothing but cached objects persists.

### 7.2 Refresher (one goroutine per tenant, plus a trigger channel)
Triggers: startup, hook, poll timer (`poll_interval`, default 15 m, ±10% jitter), miss-lookup (8.5), retry backoff.
Rules: **single-flight** per tenant; triggers arriving during a run set a "dirty" flag that causes exactly one more run;
**debounce** ≥ 10 s between runs (except startup). One run =
1. List/target-fetch from GitHub (conditional). 304 and no hint ⇒ done.
2. For each channel, validate releases newest-first (5.3/5.9) and pick the published one.
3. If the published release changed: **prewarm** its artifacts (8.3), then swap the snapshot. If prewarm exceeds
   `prewarm_timeout` (default 10 min) or fails, publish anyway (fill-then-serve covers the cold object).
   **At startup do not wait for prewarm**: publish what is valid immediately; prewarm continues in the background.
4. Run retention bookkeeping (8.4).

Error classification during refresh:

| Upstream outcome | Behavior |
|---|---|
| 5xx, timeout, connection error, rate limit/circuit open | keep serving LKG up to **6 h** (`stale_max_age`), count it in a metric; schedule retry with backoff (5 s, 15 s, 45 s, 2 m, then poll cadence) |
| 401/403 (non-rate-limit), repeated 404 on the repo | credential problem: `upstream_auth_failures_total`, try the next token; keep serving LKG |
| release/tag **404 (deleted/pulled release)** | **do not** serve it from stale: drop it from the snapshot (a pulled bad release must not linger); fall back to the next valid release |
| new release invalid (5.3) | skip it, keep LKG, count it in `manifest_invalid_total` |
| LKG older than 6 h and still failing | `manifest.json` returns 502 `upstream_unavailable` |

### 7.3 Release webhook (owner's idea, adopted) — `POST /v1/hooks/release`
The last step of the product's release workflow notifies the proxy so it fetches the new release immediately instead of
waiting for the poll.
- Auth: `Authorization: Bearer <hook token>`; the config stores only the token's SHA-256 (`hook_token_sha256`);
  compare hashes in constant time (`crypto/subtle`). Missing/wrong ⇒ 401 `unauthorized`, counted against a dedicated
  per-IP failed-auth bucket. Body ≤ 1 KiB. Same Host→tenant resolution as everything else. POST only.
- Optional JSON body with **untrusted hints** (unknown fields ignored, never acted on):
  ```json
  { "channel": "beta", "tag": "v2026.11.13.1319", "version": "2026.11.13.1319", "manifest_sha256": "<hex>" }
  ```
  - `tag` (strict charset/length regex) ⇒ targeted fetch `/releases/tags/{tag}`; `channel` narrows the refresh.
  - `version` and `manifest_sha256` (digest of the uploaded `update-manifest.json`) are **expectations**: after
    fetching, compare with GitHub's manifest. Mismatch ⇒ GitHub is not consistent yet (hook fired early) or something
    is wrong: **do not publish**, retry with backoff.
  - **Never accept a URL, a repo, or the manifest body from the hook.** A caller URL would be a client-controlled
    fetch target (SSRF), and caller-supplied manifest bytes add nothing because the canonical bytes come from GitHub.
    The repo always comes from tenant config.
- Response: 202 immediately; all work in the background. The pipeline must call it **after** the release is published
  (not draft) with all assets uploaded; if the proxy refreshes too early and sees nothing new, it retries with the
  backoff above before waiting for the next poll.
- A leaked hook token can at worst trigger a refresh (bounded by single-flight + debounce + GitHub budget); it can
  never inject content, because content only comes from GitHub and must pass signature verification.
- The hook is optional: with no `hook_token_sha256` configured the endpoint returns 404 and only polling runs.
- Token generation: `relxy hook-token` (CLI) prints a random token once plus its SHA-256 to paste in config.
  Chosen over "generate on first run" because it needs no state file and no way to retrieve a secret later.

---

## 8. Cache and retention

### 8.1 Layout and rules
`cache/objects/<sha256[0:2]>/<sha256>` — written once, immutable, name = sha256 of content. Downloads land in
`cache/tmp/` first. The whole cache directory is safe to delete at any time (it is a cache, not the source of truth).
Single instance is assumed: document that multiple instances require shared storage; do not implement it.

### 8.2 Fill (the only way an object is created)
1. Check free disk space ≥ `size` + `min_free_bytes`; if not, evict unpinned objects (8.4); still not enough ⇒
   503 `capacity` (and a metric).
2. Acquire the per-sha **single-flight** slot: concurrent requests for the same sha256 wait on one upstream fetch.
3. Stream from upstream (6.3) into `cache/tmp/<random>`, computing sha256 while writing and counting bytes.
4. Require: hash equals the manifest sha256 AND byte count equals the manifest size AND GitHub-reported size.
   Mismatch ⇒ delete the temp file, return 502 `manifest_invalid`, **do not poison the cache**, count a metric.
5. `fsync` the file, atomically `rename` into `objects/xx/<sha256>` (then best-effort fsync of the directory).
6. Wake all waiters; they serve from disk.
The fill runs on a context **detached from the requesting client** (own timeout `fill_timeout`, default 30 min), so a
client disconnect never cancels the transfer for other waiters. Global cap on concurrent fills (default 2 per tenant,
`max_concurrent_fills`). Resuming an interrupted *fill* is not needed in v1 (restart it); note as future work.

### 8.3 Fill-then-serve (decided) and prewarm (decided)
The proxy always completes and verifies a fill **before** sending the first byte; then it serves from disk. Reasons:
a hash cannot be verified on a partial transfer, and the acceptance criterion "wrong sha256 ⇒ zero bytes to the
client" requires it. Consequently Range requests are served from disk; **Range is never passed upstream.**
Prewarm (7.2): when a refresh finds a new release, fetch all its targets in the background (bounded concurrency)
before publishing the new manifest, so clients almost never hit a cold object.

### 8.4 Retention (owner's requirement, refined)
- Per tenant, configurable: `retain: { latest: 4, beta: 8 }` = number of most recent **verified** releases per channel
  kept on disk. A periodic sweep (every 10 min and at startup) deletes objects that are not "pinned", where pinned =
  referenced by a retained release.
- The index additionally covers `index_depth` (default retain + 4) releases so older-but-recent versions can still be
  fetched on demand.
- **Deviation from the owner's idea, on purpose:** an on-demand object outside the retention window is NOT deleted the
  instant a download finishes. "Finished" cannot be known reliably (interrupted clients, Range resume, concurrent
  clients) and deleting would defeat resume and single-flight. Instead such objects get an **idle TTL**
  (`on_demand_idle_ttl`, default 1 h since last access) and the same sweep removes them.
- Last-access tracking: keep it in memory and `os.Chtimes` the file (debounced) when served, because `atime` is
  unreliable (noatime); on startup rebuild the in-memory view by scanning `objects/` (sizes + mtime).
- Global backstop `max_cache_bytes`: when exceeded, evict unpinned objects oldest-access-first; **never evict pinned
  objects or in-flight fills**. If still over the cap, record a metric (do not delete pinned data).
- Startup: delete everything in `cache/tmp/`; ignore/remove files in `objects/` whose name is not 64 hex.
- Sizing example for the README: worst case 12 versions × 4 targets × 200 MiB ≈ 9.6 GB per tenant.
- Seam for later: an `ObjectStore` interface (disk now); an R2/S3 mirror may be added if egress hurts. Do not build it.

### 8.5 Unknown sha256 ⇒ check GitHub once, bounded (decided)
If the requested sha256 is not in the index: force **one** index refresh from GitHub looking back
`miss_lookup_depth` releases (default 30), then re-check. Found ⇒ serve (and index it); still missing ⇒ 404
`no_release`. Because the sha comes from the client and could be random, guards are mandatory: lookups share the
tenant refresher's single-flight and 10 s debounce (N concurrent misses ⇒ one upstream refresh); misses are
remembered in a bounded negative cache (default 5 min, per sha, size-capped); the lookup counts against the caller's
per-IP metadata budget (it is cache-miss work); and a per-tenant `miss_lookups_per_minute` cap (default 6) keeps GitHub
spend bounded (over the cap ⇒ answer 404 immediately with no upstream call). Legitimate misses: the hook was missed,
or the release is older than the retained window.

---

## 9. Download request algorithm (step by step)

1. Middleware: request id → Host normalization → tenant lookup (unknown ⇒ `unknown_tenant`, stop) → response
   wrapper recording status/bytes for metrics.
2. Validate path: channel ∈ tenant.channels, platform/variant ∈ allowed sets, `sha256` matches `^[0-9a-f]{64}$`.
3. Load the snapshot; find `bySHA[sha]` and verify it belongs to the same (platform, variant) (and channel) as the
   URL; not found ⇒ 8.5 lookup, else 404 `no_release`.
4. Acquire a per-IP concurrent-download slot (10.2) and a tenant capacity slot; failure ⇒ 429/503.
5. If the object is missing on disk ⇒ fill (8.2) — waiting on the single-flight; map errors to 502/503.
6. Open the file; set `Content-Type` from the kind table, `ETag: "<sha256>"`, `Accept-Ranges`, cache headers, then
   `http.ServeContent(w, r, "", modTime, file)`. It handles `Range`, suffix ranges, `If-Range`, 206 `Content-Range`,
   416, `HEAD`, and `Content-Length`. Because the ETag is set, a stale `If-Range` falls back to a full 200.
7. Release slots on return (`defer`). Touch last-access. Record bytes and duration in metrics.

Go pitfalls here: (a) do NOT set `http.Server.WriteTimeout` (it would kill a 200 MiB download); use `ReadHeaderTimeout`,
`IdleTimeout`, and extend a rolling write deadline with `http.NewResponseController(w).SetWriteDeadline` while
streaming. (b) Any `ResponseWriter` wrapper (for byte counting/metrics) must preserve `Flusher`/`ReaderFrom`
behavior; implement `Unwrap() http.ResponseWriter` so `ResponseController` works. (c) `ServeContent` needs an
`io.ReadSeeker` — an `*os.File` is perfect.

Manifest request: middleware → validate path → snapshot → channel state → target present? (else 404
`unsupported_target`) → if `If-None-Match` equals ETag ⇒ 304 (no upstream) → else 200 with the envelope. If the
snapshot has no data yet (cold start), wait up to `cold_wait` (default 10 s) for the first refresh, then 502/503.

---

## 10. Rate limiting and abuse control (four INDEPENDENT budgets, all keyed by (tenant, client IP))

1. **Metadata** per IP: token bucket, default 60/hour with a burst (e.g. 10). **304s and cache hits do NOT consume
   it** — only cache-miss work (cold start waits, unknown-sha lookups) does. A daily check from many installs behind
   one NAT/CGNAT address must never be limited. Over budget ⇒ 429 `rate_limited` + `Retry-After`.
2. **Concurrent downloads** per IP: default 2 (configurable). A 200 MiB transfer is one request and the most
   expensive thing the service does, so this is a concurrency count, not a request rate. Slot acquired immediately
   before streaming (waiting on a fill does not burn slots). Over ⇒ 429 + `Retry-After`; clients retry.
3. **Per-tenant** caps: `max_concurrent_downloads` (default 20) ⇒ 503 `capacity`; optional `max_egress_mbps` (default
   off; implement as a rate-limited writer only if cheap) plus the per-token GitHub budget guard (6.4).
4. **Hook failures** per IP: small bucket on failed hook authentications (anti brute force). Successful hooks are
   bounded by single-flight + the 10 s debounce.

Client IP: trust the configured header (`client_ip_header`) **only if the TCP peer address is inside
`trusted_proxies`**; otherwise ignore the header and use the peer address. With Caddy on the same box the peer is
loopback; with Cloudflare in front the header is `CF-Connecting-IP`. A bare `X-Forwarded-For` from an untrusted peer
is ignored (test it). Use the right-most untrusted entry when parsing a multi-value `X-Forwarded-For`. IPv6 clients are
keyed by their /64.

Counters live in memory only; idle entries swept every few minutes; the map has a hard entry cap. Raw IPs are never
persisted or exposed (metrics are labeled by tenant only).
Cloudflare per-domain rules (if used) are the outer layer; the app limits are a lenient backstop.

### 10.1 Algorithms (chosen from the "Visualizing algorithms for rate limiting" article, smudge.ai, 2024)

The article compares fixed window (cheap, but a burst of up to 2× the limit at a window boundary), sliding window log
(exact but stores one timestamp per request), sliding window counter (approximation from two counters, no boundary
burst) and token bucket (separate burst size and long-run average). Mapping to our budgets — implement exactly TWO small
primitives in `internal/limit`, both in-memory, ~30 lines each, with an injectable clock for tests:

| Budget | Algorithm | Why |
|---|---|---|
| 1. Metadata per (tenant, IP) | **Token bucket** | want a generous burst (many installs behind one NAT) but a bounded average: `capacity = meta_burst`, refill rate = `meta_per_hour / 3600` per second. Lazy refill: store `tokens` and `last` per key; on each call add `elapsed × rate` capped at capacity, then try to take 1 |
| 2. Concurrent downloads (IP and tenant) | **Counting semaphore** (not a rate) | the cost is the transfer duration, not the request count; the article's algorithms don't apply |
| 3. Per-tenant miss lookups (`miss_lookups_per_minute`) | **Token bucket** | `capacity = N`, refill `N/min`; bursts OK, long-run bounded |
| 3b. GitHub budget guard | **Mirror GitHub's own fixed window** | GitHub enforces a fixed hourly window; follow its `X-RateLimit-Remaining/Reset` headers instead of modeling it |
| 4. Hook auth failures per IP | **Sliding window counter** | an attacker would exploit a fixed window's boundary 2× burst. Keep `prev` and `curr` counts per window; `estimate = prev × (window − elapsed_in_curr)/window + curr`; deny when `estimate ≥ limit` (default 10 failures per 5 min) |

Not used: sliding window log (memory grows with traffic), plain fixed window (boundary burst), leaky bucket (not needed).
Implementation notes: a token-bucket entry whose tokens would already be full is indistinguishable from "never seen", so
the sweeper simply deletes it — memory stays proportional to *recently active* keys, plus a hard entry cap. State lives in
memory only; a restart resets it (acceptable for a single instance; stated in the README). On any internal limiter error,
**fail open** (allow, and record `rate_limit.decision=fail_open` in the event) — never take the service down because the
limiter broke (the article's advice). Keys are (tenant, IP) because clients are unauthenticated; the NAT/CGNAT weakness the
article describes is why 304s and cache hits do not consume the metadata budget and why bursts are generous. A
client-supplied installation id as an extra key was considered and NOT adopted (spoofable, adds a client contract).
Rejections return 429/503 with an exact `Retry-After` (for a token bucket: `ceil((1 − tokens) / rate)` seconds) and
`X-RateLimit-Limit/Remaining/Reset` headers on 429 responses so the updater knows how long to wait.

---

## 11. Configuration reference (YAML, strict: unknown fields rejected; secrets from env)

```yaml
listen: "127.0.0.1:8080"
admin_listen: "127.0.0.1:9090"
cache_dir: "/var/lib/relxy/cache"
max_cache_bytes: 53687091200       # 50 GiB global backstop (reference profile; minimum profile: 10 GiB, see 15.4)
min_free_bytes: 2147483648         # refuse fills that would leave less free disk
trusted_proxies: ["127.0.0.1/32"]
client_ip_header: "X-Forwarded-For"   # or "CF-Connecting-IP"
shutdown_grace: "60s"
tenants:
  - hosts: ["woomtoon-updates.example.com"]   # placeholder; real domains configured later by the infra owner
    app: "woomtoon"
    repo: "ZyFeronix/Woom-Toon"
    auth:
      type: "pat"                              # later: "github_app"
      tokens:                                  # ordered; failover on 401 / rate-limit 403
        - { env: "GITHUB_TOKEN_WOOMTOON", expires: "2027-09-01" }   # `expires` optional, feeds a gauge
    channels: ["latest", "beta"]
    retain: { latest: 4, beta: 8 }
    index_depth: 12                            # default = largest retain + 4
    on_demand_idle_ttl: "1h"
    poll_interval: "15m"
    prewarm_timeout: "10m"
    stale_max_age: "6h"
    miss_lookup_depth: 30
    miss_lookups_per_minute: 6
    hook_token_sha256: "<hex sha256 from `relxy hook-token`>"   # optional
    allow_assets: ["WoomToon-*.zip", "WoomToon-*-Setup.exe", "WoomToon-*.tar.gz", "WoomToon-*.deb",
                   "update-manifest.json", "update-manifest.sig"]
    signing_keys:
      - { key_id: "2026a", public_key: "<base64url unpadded 32-byte ed25519 public key>" }
    limits: { meta_per_hour: 60, meta_burst: 10, concurrent_downloads: 2, max_concurrent_downloads: 20 }
```

Startup validation is strict and fails fast; print a config summary with secrets redacted, then exit non-zero on any
error: duplicate host (within or across tenants), missing token env var, empty token, invalid public key (not 32
bytes / not base64url), a private-key-looking field anywhere, unreadable/unwritable cache dir, unsupported channel name
(only `latest`, `beta`), `retain` < 1, `allow_assets` empty or missing the two manifest assets, invalid duration/CIDR,
`hook_token_sha256` not 64 hex.

Secrets handling: wrap every token in `type Secret` whose `String()`, `GoString()`, `Format(fmt.State, rune)`,
and `MarshalJSON/MarshalText` all return `[REDACTED]`, with a comment saying why (`%+v` dumps would otherwise leak
it). Never expose the token, even partially. The real config is git-ignored;
`config.example.yml` is committed. Test-only flags (never config): `--allow-unsigned`; `--test-upstream <url>` and
`--test-redirect-hosts <suffixes>` to point the GitHub client and the redirect allow-list at a local fake (refuse to
start with them unless an explicit `--i-am-testing` guard is given).

---

## 12. CLI subcommands (same binary)

```
relxy serve  --config config.yml [--allow-unsigned]       # default command
relxy keygen --key-id 2026a                                # prints private+public (base64url) once
relxy sign   --key-env RELEASE_SIGNING_KEY --key-id 2026a --in update-manifest.json --out update-manifest.sig
relxy verify --pub <base64url> --manifest update-manifest.json --sig update-manifest.sig   # debug aid
relxy hook-token                                           # prints token once + its sha256 for config
relxy version
```
`sign` reads the private key from an env var (never argv), canonicalizes with the shared JCS code, and writes the
`{"sigs":[…]}` file. `keygen`/`hook-token` use `crypto/rand`. Keep these thin; they exist so signer and verifier share
one implementation and so no one hand-rolls crypto in shell.

---

## 13. Metrics, health and shutdown

**Metrics** (hand-written Prometheus text exposition on the admin listener; label by `tenant` only — never IP, version,
sha or token): `requests_total{tenant,route,status}`, `request_duration_seconds` (histogram, few buckets),
`bytes_sent_total`, `active_downloads`, `fills_total{result}`, `fill_bytes_total`, `cache_bytes`, `cache_objects`,
`cache_evictions_total{reason}`, `singleflight_waiters`, `refresh_total{trigger,result}`, `refresh_last_success_
timestamp_seconds`, `manifest_invalid_total{reason}`, `stale_served_total`, `rate_limited_total{budget}`,
`hook_total{result}`, `miss_lookups_total{result}`, `upstream_requests_total{kind,status}`,
`upstream_rate_limit_remaining`, `upstream_auth_failures_total`, `circuit_open`, `token_expires_in_seconds`.

**`/healthz` (public listener)**: minimal, see 3.2 (`{"status":"ok"|"degraded"}` only).
**`/healthz?detail=1` (admin listener only)**: small JSON with per-tenant `last_refresh_age_seconds`, published version
per channel, cache usage — no secrets.

**Graceful shutdown:** on SIGTERM/SIGINT stop accepting, `http.Server.Shutdown` with `shutdown_grace`, let in-flight
downloads finish, then cancel remaining fills and clean `cache/tmp/`. Fills detached from client contexts must observe
the shutdown signal.

---

## 14. Security notes

Threat model (what each compromise/abuse can and cannot do):

| Threat | Mitigation |
|---|---|
| Proxy host compromised | cannot forge a manifest (no private key on the box, client verifies); can DoS or replay an older signed release until `expires` |
| GitHub read token leaked from the host | read-only, one repo, fine-grained; rotate; token never in responses or error messages; ordered list for rotation |
| Hook token leaked | can only trigger a refresh (bounded); cannot inject content; rotate by changing `hook_token_sha256` |
| Client floods downloads / metadata | per-(tenant,IP) budgets, tenant cap, single-flight, negative cache, GitHub circuit breaker |
| Random-sha flood to burn GitHub quota | miss budget + negative cache + debounced shared refresh (8.5) |
| SSRF via redirect or caller URL | no caller-supplied URLs anywhere; redirect host allow-list; https only; `Proxy=nil` |
| Path traversal / cache-key injection | cache keys are validated 64-hex only; never derive paths from other input; `filepath.Join` on validated parts |
| Cache poisoning | write only after full sha256+size verification; content-addressed; atomic rename |
| Host-header spoofing / tenant confusion | exact normalized Host match; unknown ⇒ 404; Caddy serves only mapped hosts |
| Cross-product replay | per-product keys + signed `app` + signed `channel` checked by proxy and client |
| Rollback/freeze by a malicious proxy | signed `expires`; client monotonic `published` check (5.6) |
| Disk exhaustion | `max_cache_bytes`, `min_free_bytes`, eviction of unpinned objects, fill concurrency cap |
| Slowloris | `ReadHeaderTimeout`, idle timeouts, Caddy in front |
| Secret leakage via errors/dumps | `Secret` type; scrubbed upstream errors; client-controlled strings never echoed unsanitized |

Additional rules: never trust `X-Forwarded-*` except from `trusted_proxies`; reject request bodies on GET; cap header
size; `Cache-Control: no-store` on errors; set `nosniff`; constant-time compare for the hook token.

---

## 15. Deployment

### 15.1 TLS and reverse proxy (no outside services)
Run the Go binary on loopback behind **Caddy on the same host** (default, decided by the owner's preference; owner is
not 100% sure — keep both modes documented).
- Mode 1, domains DNS-only: Caddy obtains certificates via ACME HTTP-01 for each mapped domain; `trusted_proxies:
  ["127.0.0.1/32"]`, `client_ip_header: X-Forwarded-For` (Caddy sets it from the real peer).
- Mode 2, domains proxied by Cloudflare (the owner already hosts the domains there): Caddy with a Cloudflare Origin CA
  certificate or ACME DNS-01; the real client address arrives in `CF-Connecting-IP`, so configure Caddy to trust
  Cloudflare's published IP ranges and pass the client IP onward (verify the exact Caddy directive — `trusted_proxies`
  / `client_ip_headers` — against the current Caddy docs before writing the example); set
  `client_ip_header: CF-Connecting-IP`. Remember the single-label hostname rule (4). Optional later: a Cloudflare cache
  rule on `/v1/*/download/*` (immutable, content-addressed URLs are ideal) to offload egress — only if needed.
- Do not compress downloads (archives are already compressed); compressing `manifest.json` is fine.
- Caddy example (to ship in `deploy/Caddyfile.example`):
  ```
  woomtoon-updates.example.com {
      reverse_proxy 127.0.0.1:8080
  }
  ```
- The admin listener (`:9090`) is never proxied.

### 15.2 Deliverables
- Multi-stage `Dockerfile`: build stage with the Go toolchain matching `go.mod` (planning machine reported
  `go1.27.4 windows/amd64`; pin `golang:1.27` and re-check with `go version`), `CGO_ENABLED=0 go build -trimpath
  -ldflags "-s -w -X main.version=… -X main.commit=…"`, final stage distroless/scratch with a non-root UID, cache dir as
  a volume, `EXPOSE` only the public port.
- systemd unit example: `DynamicUser=yes`, `StateDirectory=relxy`, `EnvironmentFile=` for secrets (0600),
  `ProtectSystem=strict`, `NoNewPrivileges=yes`, `PrivateTmp=yes`, `Restart=on-failure`, `LimitNOFILE`, `TimeoutStopSec`
  larger than `shutdown_grace`.
- systemd unit ships with the reference-profile defaults; commented-out lines show the minimum-profile memory settings
  from 15.4 (`GOMEMLIMIT=450MiB`, `GOGC=50`, `MemoryMax=700M`).
- `config.example.yml` (section 11), `.gitignore` entry for the real config.

### 15.4 Host profiles and sizing
**Design rule (decided): the built-in defaults and `config.example.yml` target an "ideal" host/container, not the
smallest one.** The service must also run on a minimal box, so the README documents two profiles that differ only in
configuration, never in code:

| Setting | Reference profile (defaults) | Minimum profile (e.g. GCP `e2-micro`) |
|---|---|---|
| Host | ≥ 2 vCPU, 2–4 GB RAM, ≥ 80 GB disk | 1 GB RAM, ~30 GB disk, shared/burstable CPU |
| `max_cache_bytes` | 50 GiB | 10–12 GiB |
| `retain` | `latest: 4, beta: 8` | `latest: 2, beta: 4` |
| `max_concurrent_fills` (per tenant) | 2 | 1 |
| `GOMEMLIMIT` / `GOGC` / `MemoryMax` | unset (Go defaults) | `450MiB` / `50` / `700M` |
| Swap | not required | 1–2 GB swap file |

The notes below come from an external review (adopted where they hold up) and apply mainly to the minimum profile:
- **RAM/CPU:** the Go process should idle around 15–30 MB. Cached files are served from `*os.File` via `ServeContent`,
  which on Linux can use `sendfile` (kernel-to-socket) when the `ResponseWriter` exposes `ReaderFrom` and nothing
  compresses the body — so any `ResponseWriter` wrapper in section 9 MUST preserve `ReaderFrom`/`Unwrap`, and Caddy must not
  compress downloads. Fills must stream with a small buffer (`io.CopyBuffer`, 32–64 KiB) and never `io.ReadAll` an
  artifact. Expected whole-box usage (OS + Caddy + proxy) roughly 250–350 MB.
- **OOM protection:** set `GOMEMLIMIT=450MiB` and `GOGC=50` in the systemd unit, plus `MemoryMax` as a hard ceiling; add a
  1–2 GB swap file in the README (e2-micro has none by default). `max_concurrent_fills` default 1 on this class of host.
- **CPU:** e2-micro has a small sustained CPU share (burstable). Hashing a 200 MiB fill and Caddy's TLS for downloads are
  the main costs; keep fills serialized (1 at a time) and prewarm bounded.
- **Disk:** 30 GB total must hold OS, the binary and the cache. Defaults for this host: `max_cache_bytes`
  ≈ 10–12 GiB and `retain: { latest: 2, beta: 4 }` (the generic default stays 4/8 for bigger disks; with several
  products the 4/8 worst case of ~9.6 GB per product will NOT fit). Run the binary
  directly under systemd rather than Docker to save space, and keep the pinned-objects arithmetic in the README:
  retained releases × targets × artifact size must fit under `max_cache_bytes`, otherwise the sweep raises an error metric.
- **Egress depends on the GCP network tier (corrected after the owner's input):** Google gives **200 GB/month of free
  internet egress on the Standard network tier** (announced for all customers from Oct 2023), while the Always Free
  e2-micro allowance applies to the **Premium tier, which is only about 1 GB/month** from North America (excluding
  China/Australia). So the 200 GB figure is real but only if the VM uses **Standard tier networking**; verify both
  numbers and the regional restrictions on Google's current pricing pages before deployment. At 200 MiB per update,
  200 GB is ~1,000 full downloads/month. This is a hosting decision, not a code change. If traffic ever outgrows the
  free egress, Cloudflare caching of `/v1/*/download/*` (immutable content-addressed URLs; the free plan's 512 MB
  per-file cache limit fits 100–250 MiB artifacts — verify limits and acceptable-use terms for large binaries) is the
  planned relief valve. On any other provider, check the bundled bandwidth the same way.
- Short ops README: deploy, rotate a GitHub token, add a tenant/domain, generate keys and the hook token, purge the
  cache, read metrics, expected disk usage, rotate signing keys, Cloudflare-in-front variant, multi-instance caveat,
  the ObjectStore seam.

### 15.3 Example pipeline step (documentation only — NOT built or modified by this task)
In the product repo's release workflow, roughly:
```
build artifacts (Electron tooling)
write update-manifest.json   # version, published, expires, notes, targets{…sha256,size,kind,url}
relxy sign --key-env RELEASE_SIGNING_KEY --key-id 2026a --in update-manifest.json --out update-manifest.sig
gh release create <tag> [--prerelease for beta] <artifacts> update-manifest.json update-manifest.sig
curl -fsS -X POST https://<product-domain>/v1/hooks/release -H "Authorization: Bearer $RELXY_HOOK_TOKEN" \
     -H 'Content-Type: application/json' -d '{"channel":"…","tag":"<tag>","version":"…","manifest_sha256":"<sha256 of update-manifest.json>"}'
```
The hook step must run after the release is published and assets are uploaded; it should not fail the release if the
proxy is briefly down (the 15-minute poll covers it).

---

## 16. Code layout and Go notes

Suggested layout (adjust if a simpler one emerges; keep packages few):
```
cmd/relxy/        main + subcommands (serve, keygen, sign, verify, hook-token, version)
internal/config/         load, strict validate, Secret type, redacted summary
internal/tenant/         host → tenant registry
internal/httpapi/        mux, middleware (request id, tenant, limits), handlers, JSON errors
internal/github/         apiClient/blobClient, Credential (pat), release/asset types, budget guard, breaker
internal/manifest/       schema, validation, JCS, signature verify, envelope
internal/cache/          store, single-flight, fill, sweep/eviction, last-access
internal/refresh/        per-tenant refresher, hook handling, poll, prewarm, snapshot
internal/limit/          token bucket, sliding window counter, counting semaphore, client-IP resolution (10.1)
internal/metrics/        tiny Prometheus text writer
docs/PLAN.md  deploy/ (Dockerfile, systemd, Caddyfile.example)  config.example.yml  README.md  CHANGELOG.md
```
Introduce an interface only where tests need a seam (clock, upstream GitHub). Stdlib covers nearly everything:
`net/http` (Go ≥ 1.22 `ServeMux` supports `GET /v1/{channel}/{platform}/{variant}/manifest.json` patterns), `crypto/ed25519`,
`encoding/base64` (`RawURLEncoding`), `context`, `sync`, `sync/atomic`, `crypto/subtle`. Allowed small
dependencies: a YAML parser (strict mode); optionally `golang.org/x/sync/singleflight` (or a ~30-line in-house version).
Do NOT add `client_golang`, web frameworks, or ORMs. Expect roughly 2–3k lines. `go build` with
`CGO_ENABLED=0` yields one static binary. Concurrency is goroutines + `sync`/channels. Run `go vet`, `gofmt`, and
`go test -race` (on Windows `-race` needs CGO/gcc: run it in WSL or Docker). Inject the build version with `-ldflags`;
keep a `CHANGELOG.md`. Development may happen on Windows but production is Linux: use `filepath`, avoid
platform-specific syscalls, and note that `os.Rename` and directory `fsync` behave differently on Windows (best-effort).

---

## 17. Tests (no network; local `httptest` fake of GitHub with a fake token; no test may reach api.github.com)

Table-driven unit tests + `httptest` fakes for:
- **Tenancy:** unknown Host ⇒ 404 `unknown_tenant`, no upstream call, no default tenant; host normalization; two tenants
  isolated (repo, token, limits, allow-list, keys).
- **Manifest:** each validation failure of 5.3 individually; version passthrough (`2026.11.13.1319` byte-identical in the
  envelope); expired manifest; channel/app/prerelease mismatch; duplicate JSON keys; non-integer numbers; size caps;
  unsupported target ⇒ 404; allow-list enforcement.
- **Signature:** valid, tampered manifest, wrong key, missing sig, unknown `key_id`, multi-signature rotation,
  `--allow-unsigned`; padded base64 rejected; JCS RFC 8785 vectors + Unicode key ordering; `keygen → sign → service
  verifies` round trip; service refuses a config containing a private key.
- **Caching/HTTP:** ETag/304 with ZERO upstream calls; content-addressed download (unknown sha ⇒ 404; sha of another
  target ⇒ 404); Range: 206, `Content-Range`, suffix range, `If-Range` match/mismatch, 416; HEAD.
- **Single-flight:** N concurrent requests for a cold sha ⇒ exactly ONE upstream fetch, the rest served from disk;
  fill continues if the first client disconnects.
- **Integrity:** digest or size mismatch ⇒ temp file discarded, 502, zero bytes sent, cache not poisoned.
- **Retention:** keeps 4/8; deletes older; idle TTL for on-demand objects; never evicts pinned/in-flight; cap eviction
  order; startup cleans `tmp/`.
- **Refresh:** hook wrong token 401 / right token 202 + refresh + prewarm; hook hints (tag ⇒ targeted fetch; version or
  digest mismatch ⇒ not published + retry; URL/repo/manifest fields in the body ignored); debounce; early-hook retry
  backoff (fake clock); poll fallback; unknown-sha lookup (found after forced refresh; not found ⇒ 404 + negative cache;
  N concurrent misses ⇒ one upstream refresh; per-tenant miss cap; random-sha flood ⇒ bounded upstream calls); a warm
  client request never calls upstream.
- **Stale policy:** upstream 5xx/timeout ⇒ serve stale within 6 h, then 502; release 404 ⇒ NOT served stale; invalid new
  release ⇒ keeps LKG; cold + invalid ⇒ 502 `manifest_invalid` (acceptance 4).
- **Rate limits:** each of the four budgets independently; (tenant,IP) keying (limiting product A does not affect
  product B); 304s/cache hits don't consume the metadata budget; trusted-proxy handling (bare `X-Forwarded-For` from an
  untrusted peer ignored; trusted peer honored); IPv6 /64 keying.
- **Public `/healthz`:** answers on any Host (incl. unmapped and by IP) with only `{"status":…}`; 503 when the cache dir
  is unwritable or the matched tenant has no refresh within `stale_max_age`; never calls upstream; leaks no version,
  tenant name or path; `/metrics`, `/version`, `?detail=1` are NOT served on the public listener.
- **Rate-limit algorithms:** token bucket (lazy refill, burst, exact `Retry-After`, full-bucket entries swept),
  sliding window counter (no 2× boundary burst — test a burst straddling the boundary), semaphore release on every
  path incl. panic/abort, fail-open on limiter error, 429 carries `X-RateLimit-*`.
- **sendfile readiness:** the byte-counting `ResponseWriter` wrapper still exposes `io.ReaderFrom`/`Unwrap`.
- **Secrets:** the fake token appears in NO response byte, NO error body, no `%+v` dump; the
  redirect hop carries no `Authorization` header; redirect to a non-allow-listed or non-https host is refused; signed
  URL absent from all errors and responses.
- Large-file tests generate the ~200 MiB stream on the fly (no fixtures). Must pass: `go vet ./...`, `gofmt -l`
  (clean), `go test -race ./...`. **Report exactly what was run; never claim a test passed that was not executed.**
  Docker build and systemd unit are exercised or explicitly reported as untested.

---

## 18. Implementation phases (each ends with its tests green; one MR per phase is fine)

1. Skeleton: config + strict validation + `Secret` type + tenant registry + JSON error model + request id.
2. GitHub client: two clients, `pat` credential, conditional requests, redirect control, scrubbed errors, budget guard +
   circuit breaker; fake-upstream test harness.
3. Manifest: JCS, schema validation, signature verify, release selection, envelope, ETag/304; CLI `keygen`/`sign`/
   `verify`/`hook-token`.
4. Refresher + snapshot: background refresh, poll with jitter, backoff, stale policy, LKG, hook endpoint + hints.
5. Cache: content-addressed store, single-flight, detached fill, verification, prewarm, retention sweep, eviction,
   startup recovery, unknown-sha lookup + negative cache.
6. Download serving: `ServeContent`, Range/If-Range, write-deadline handling, concurrency slots.
7. Rate limits (4 budgets; token bucket + sliding window counter + semaphore, 10.1), client-IP resolution, admin listener (`/healthz`, `/version`, `/metrics`), graceful shutdown.
8. Dockerfile, systemd unit, Caddy examples, README, `config.example.yml`; report what was not exercised.

---

## 19. Acceptance criteria

1. With a private test repo, a client holding no credentials can fetch a signed envelope and download a 200 MiB
   artifact, resuming an interrupted transfer with Range.
2. The raw token appears in NO response byte, NO error body, and is never sent to the redirect host
   (proven by test).
3. Two domains map to two different repos with independent tokens, limits, keys and allow-lists; an unknown Host is a
   404 `unknown_tenant` with no upstream call; limiting one product never affects the other.
4. A tampered `update-manifest.json`, a wrong sha256, an unknown `key_id`, or an expired manifest results in
   `manifest_invalid` (cold start, no LKG) and zero artifact bytes to the client; with an LKG the previous good release
   keeps being served.
5. N concurrent downloads of a cold artifact produce exactly one upstream fetch; subsequent requests are served from disk.
6. A 4-segment version `2026.11.13.1319` round-trips byte-identically through the service.
7. Retention keeps the configured N latest / M beta versions, removes older ones, and expires on-demand objects by idle
   TTL.
8. The hook triggers an immediate refresh + prewarm; with no hook the 15-minute poll still picks up the release; an
   unknown sha256 triggers at most one bounded upstream lookup.
9. `go test -race ./...`, `go vet ./...`, `gofmt -l` are clean; Docker build and systemd unit are exercised or
   explicitly reported untested.

---

## 20. Out of scope (do not build)

Any client app, updater, installer, Electron/builder change or app CI (the pipeline example in 15.3 is documentation
only); the service signing anything at runtime; a web UI, admin panel, database, user accounts, licenses, payments,
analytics; staged-rollout percentages, A/B channels, delta/patch updates; multi-instance coordination, Redis,
Kubernetes manifests; object-storage mirroring (leave the `ObjectStore` seam only); GitHub App auth (interface prepared,
implement later); resuming interrupted fills; macOS/mobile targets.

---

## 21. Decisions log, known consequences, remaining items

**Decided by the owner**
- Content-addressed download URLs; prewarm + fill-then-serve; metadata budget counts only cache-miss work; manifest
  `expires` + client monotonic check.
- One domain per product; per-product keys; Caddy on the same server in front (Cloudflare variant documented).
- Manifest comes from GitHub release assets built/signed by the pipeline (not pushed by CI).
- Webhook + 15-minute poll fallback; hook carries hints only (tag, version, manifest digest), never a URL/manifest body;
  unknown sha256 triggers a bounded GitHub lookup.
- Retention defaults 4 latest / 8 beta, configurable; older objects removed; on-demand objects expire by idle TTL
  (refinement of "delete after download").
- `beta` with no prerelease ⇒ 404; `linux/installed` = `deb` (provisional); Go version = toolchain on the machine
  (planning machine reported `go1.27.4`).
- Signing keys are generated by the Go program's `keygen` command but the running service never holds a private key
  (5.8). **The owner asked for auto-generation by the proxy; this is the one deliberate deviation, made for security.
  If the owner still wants server-held keys, record that signatures then only protect against transport corruption.**

- `/healthz` is public (minimal body, host-agnostic) so external monitors can use it; detailed health, `/metrics` and
  `/version` stay on the loopback admin listener.
- Defaults target an ideal host/container; a small host (e.g. e2-micro) is a documented "minimum profile" that only
  changes configuration (15.4). Memory/disk hardening adopted from external review.
- Egress: 200 GB/month free applies to GCP Standard network tier; Premium/Always Free e2-micro is ~1 GB (verify).

- Rate limiting uses token bucket (metadata, miss lookups), sliding window counter (hook failures), semaphores
  (downloads) and GitHub's own window headers (10.1), per the smudge.ai article.

**Known consequences to communicate to the client team**
- Verify signature, `app`, `channel`, `expires`, and monotonic `published`; sanitize markdown notes; compare versions
  numerically themselves; retry on 429/503 with `Retry-After`; a `beta` client sees only prereleases via `beta`.

**Remaining (non-blocking)**
1. Go module path and license (match the org's existing choice). The project name is **Relxy** (decided; the
   owner checked that nothing software-related uses it). Before the first tag, confirm the repo name, module path,
   Docker image name and that no `relxy` command clashes on the target distros.
2. Real domains and the first two repos (Woom!Toon + a second dummy/public repo for an end-to-end test) — config only.
3. Exact Caddy directives for the Cloudflare variant (verify against current Caddy docs).
4. Whether `max_egress_mbps` is worth implementing in v1 (default: skip).
5. Hosting target: confirm the host and, if GCP, that the VM uses Standard network tier (15.4). Cloudflare-cached
   downloads remain an optional relief valve.

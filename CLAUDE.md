# CLAUDE.md: Staffbase Content Copy Tool (sbcopy)

Read this file at the start of every session. It overrides anything else in this repo or in a prompt unless the human owner explicitly says otherwise in the current session.

## What we are building

An internal Staffbase Professional Services tool that copies HTML-based content (Classic pages, News posts) and all of its media from one Staffbase installation to another, registers every image and file in the destination File Manager, and prepares Classic pages for Staffbase's native "Move to Content Designer" upgrade. It runs only on a consultant's laptop. Full plan: `docs/plan.md` (v3). Where this file and the plan disagree, this file wins.

Primary use case: source is an older account with Classic (HTML) pages, destination is a newer account where Content Designer is the default. The public API cannot write Content Designer structure. We copy Classic drafts through the API. A human then runs the Studio upgrade per page. The tool never performs or scrapes the upgrade.

## Hard constraints (never violate, never "temporarily" bypass)

1. **Local only.** Customer content and credentials exist only on the consultant's machine. No cloud services, hosted backends, queues, ETL platforms, analytics, telemetry, crash reporting, auto-update checks, or external AI/LLM calls anywhere in the product code, tests, or build.
2. **One egress path.** All HTTP goes through `internal/egress`. It enforces: exact-host allowlist, TLS 1.2+, no plaintext HTTP, redirects only to allowlisted hosts with `Authorization` stripped on host change, rejection of loopback, link-local, private, and metadata IP ranges, no system proxy unless explicitly configured. No other package may import `net/http` clients or open sockets. A lint rule enforces this. Do not weaken it.
3. **Source is read-only.** The source client is an operation allowlist: GET, plus the two read-only search POSTs (`POST /medialibrary/entries`, `POST /medialibrary/entries/all`). Everything else must fail in code and in tests.
4. **Copy, never move. Never delete.** No delete or replace operations are exposed in the MVP, on either side. No publishing. No `notificationChannels` on any news create or update. Never call the make-media-public endpoint.
5. **Everything created is invisible.** Pages: `published:false` and no `readerIds`. News: omit `published` (draft). Collections default to token and administrators only unless the human picks an accessor group.
6. **Credentials.** Tokens live in process memory by default, arrive over stdin or the OS keychain, never in argv, environment variables, files, test fixtures, logs, or git history. Never read `.env`, keychain exports, or any file that may hold secrets. Never print a token.
7. **No customer content in logs.** Logging uses typed fields only (IDs, status codes, counts, durations, host names). No titles, bodies, file names, URLs with query strings, or tokens. A CI test fails the build if a log line contains any of those.
8. **Local persistence.** Run data goes in the OS app-data directory, never Desktop, Documents, Downloads, or synced folders. Staged payloads are AES-GCM encrypted with a per-run key. The journal stores IDs, hashes, and statuses only. Cleanup destroys the key.
9. **No live calls from you.** You must not call any real Staffbase installation, run any command that does, or ask for tokens. Build and test against mocks and fixtures only. The human runs live tests and may paste back only redacted typed output (status codes, counts, IDs). If you ever see what looks like real customer content or a credential, stop, tell the human, and do not copy it anywhere.
10. **No undocumented endpoints, no scraping the Studio.** Use only operations in the vendored OpenAPI specs under `specs/`. If something needs an operation that is not there, stop and ask.

## Architecture decisions already made

- Engine in Go, no CGO, single static binary for macOS (arm64, amd64) and Windows. CLI first. A Tauri desktop shell comes later and is out of scope until the human says so.
- Engine speaks newline-delimited JSON events on stdout and commands on stdin. The CLI and any future UI use the same contract.
- HTML is rewritten with a tokenizer and byte-span edits, never parse and reserialize. Staffbase custom elements (for example `hashtag-block`) are copied verbatim.
- Write-ahead journal: record intent (deterministic externalId, source hash, attempt), then perform the call, then record receipt. A crash between them leaves the item `UNCERTAIN` and resume reconciles by reading the destination before any retry. Never blindly retry a POST.
- externalId scheme, valid for both pages and news: lowercase `sbmig_{projectkey}_{kind}_{sourceid}` using only a to z, 0 to 9, `_`, `-`.
- Media pipeline per distinct source file: download (authenticated), strip transform segments to get the original, `POST /media`, then `PUT /medialibrary/entries/{mediumId}` to register in the File Manager, then add to a collection in batches.
- Retries: 5 attempts, exponential backoff with full jitter, 1 second base, 60 second cap. Start at 2 concurrent requests, halve on any 429.
- Pin API versions through `Accept` and `Content-Type` headers. No header returns the oldest version.

## API facts you can rely on (Verified in specs) and facts you must not assume

Verified: `GET/POST /pages`, `GET/PUT/DELETE /pages/{pageId}`, `GET /pages/sync`. Page `contents.{locale}.{title,content}` where `content` is an HTML string. Request uses `readerIds` and `adminIds`, response returns `readers` and `admins`. News: `POST /channels/{channelID}/posts` with `contents.{locale}.{title,teaser,content,image}`, `externalID` (lowercase a to z, 0 to 9, `_`, `-`), notifications only if `notificationChannels` is sent. No create-channel operation exists. Media: `POST /media` multipart with `file` and optional `metadata` JSON, response has `id`, `ownerID`, `resourceInfo`, `transformations`. Media URLs require authentication. File Manager: `PUT /medialibrary/entries/{mediumId}`, collections create and add via `POST /medialibrary/collections` and `/entries`, search via `POST /medialibrary/entries`. The File Manager is per owner. Pagination: cursor on pages and File Manager lists, limit and offset on media list.

Unknown, never assume (tests must not encode a guess): what `POST /pages` creates in a Content Designer-default account, behavior after Feb 4, 2027, duplicate `externalId` behavior, whether `PUT` registration is idempotent, internal link URL formats, upload and body size limits, rate limit numbers, media hostnames beyond the same-domain `/api/media/secure/...` path, what GET returns for Content Designer pages. Model each as an explicit `TODO(U#)` and a swappable behavior behind an interface so Phase 0 results can be dropped in.

## Repository layout

```
cmd/sbcopy/            CLI entry
internal/egress/       the only network code
internal/staffbase/    spec-derived clients (source ro, destination rw), operation allowlists
internal/mockserver/   in-process Staffbase mock built from specs/ for tests
internal/journal/      write-ahead journal, state machine, reconciliation
internal/stage/        encrypted staging
internal/htmlrewrite/  tokenizer, URL classification, readiness lint
internal/media/        download, upload, File Manager registration, collections
internal/connectors/   news, pages
internal/plan/         plan, dry run, approval
internal/report/       redacted and full reports, upgrade worklist
internal/logx/         typed logger only
specs/                 vendored public OpenAPI YAML (no customer data)
docs/plan.md           the v3 plan
testdata/              synthetic fixtures only, invented content, no real names or branding
```

## How to work

- Start every task in plan mode. Show the plan, wait for approval, then implement.
- Tests first for anything with a decision in it. Table-driven tests. Fault injection (429, 5xx, timeouts, partial bodies, redirects to foreign hosts) lives in the mock server.
- Small commits. Each commit builds, passes `make check`, and does not weaken a constraint above.
- `make check` runs: format, vet, lint (including the egress and logging rules), unit tests, mock-server tests, SBOM-friendly dependency check. Add a dependency only with a one-line justification in the PR description. Prefer the standard library. No dependency may do networking or telemetry.
- Never write secrets, real URLs of customer installations, or real content into code, tests, fixtures, comments, or commit messages. Use invented hosts like `source.example.test` and `dest.example.test`.
- Writing style for docs and comments: plain, direct. No em dashes or en dashes anywhere.
- If a requirement here conflicts with convenience, stop and ask. Do not silently choose convenience.

## Definition of done for any milestone

Acceptance tests listed in `MILESTONES.md` pass, `make check` is green, no constraint above is weakened, new unknowns are recorded as `TODO(U#)`, and a short note lists what the human must run live and what typed output to bring back.

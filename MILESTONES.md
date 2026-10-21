# MILESTONES.md

Milestones are small, each ends with acceptance tests Claude Code can run without any live Staffbase call. Gated milestones wait for the human to confirm Phase 0 results.

Two tracks run in parallel:

- **Track A (Claude Code):** M1 to M6, built on specs, mocks, and synthetic fixtures.
- **Track B (human):** the Phase 0 experiment in `docs/plan.md` section 13, run by hand on test installations. Its results unlock M7 and M8.

## M0: Phase 0 experiment (human, not Claude Code)

Run section 13 of the plan against your test installations. Bring back **typed, redacted output only**: status codes, counts, IDs, booleans (for example "created page opened in Classic Editor: yes", "appears in Ready tab: yes", "duplicate externalId: rejected with 409"), hostnames contacted. Record results in `docs/phase0-results.md`. No page content, titles, file names, or tokens.

## M1: Foundations and security skeleton (kickoff scope)

Deliver:

1. Go module, `Makefile` with `make check`, CI config that runs it locally (no cloud CI required), lint config.
2. `internal/egress`: allowlisted HTTP client per section "Hard constraints" in `CLAUDE.md`.
3. `internal/logx`: typed logger that cannot accept free-form strings from payloads.
4. Secret handling: token intake over stdin, in-memory holder that redacts on print and marshal, no env or argv path.
5. `internal/staffbase`: minimal clients generated or hand-written from `specs/` for the operations needed by News, media, File Manager, and Pages (list, get, create, update). Source client and destination client are separate types. Operation allowlists enforced in code.
6. `internal/mockserver`: in-process Staffbase mock built from `specs/`, with fault injection (429, 5xx, timeouts, partial bodies, redirect to a foreign host).

Acceptance tests:

- A request to a non-allowlisted host fails before any connection is opened.
- A redirect to a non-allowlisted host is refused, and `Authorization` is stripped on an allowlisted host change.
- Requests resolving to loopback, link-local, private, and metadata ranges are refused.
- A plaintext `http://` URL is refused. TLS below 1.2 is refused.
- The source client rejects every operation outside its allowlist (table test over all operations in the specs).
- Logger test: attempting to log a title, body, file name, URL with query, or token pattern fails to compile or fails the test. A CI test greps logs from a full mock run for synthetic canary strings and finds none.
- A secret holder never appears in `fmt`, JSON, or error output.
- Lint fails on any direct `net/http` client use outside `internal/egress`.
- Mock server serves a paginated list of pages and media and honors cursor and offset semantics from the specs.

## M2: Journal, state machine, encrypted staging

Write-ahead journal (intent then receipt), item state machine (`PLANNED`, `FETCHED`, `UPLOADED`, `REGISTERED`, `IN_COLLECTION`, `CREATED`, `VERIFIED`, `FAILED`, `SKIPPED`, `UNCERTAIN`), per-run AES-GCM staging with key destroy on cleanup, host stamping on every row, resume refused on host mismatch.

Acceptance: randomized crash injection at every state transition over a synthetic run yields zero duplicate creates after resume against the mock. Cleanup leaves no readable staged content. Journal contains no titles, bodies, or URLs (schema test).

## M3: HTML rewrite, URL classification, readiness lint

Tokenizer span-edit rewrite. URL classifier for source media, source internal links, other Staffbase links, external URLs, dangerous schemes. Readiness lint with the rule table in plan section 5.0 (advisory only).

Acceptance: rewrite preserves every byte outside edited URL spans (property test). Custom elements are untouched. External URLs are never fetched (canary host test). Lint ratings are deterministic over the synthetic corpus. Internal link format is behind an interface marked `TODO(U2)`.

## M4: Media pipeline and File Manager

Download (authenticated), original extraction, deduplication by source medium ID, `POST /media`, `PUT /medialibrary/entries/{id}`, collection create, batch add, reconciliation for media with no external ID (filename, size, creation time).

Acceptance against the mock: one file used by 10 items uploads once. Every uploaded file is registered and in the designated collection. Interrupted runs reconcile without orphans beyond those reported. Behavior for unknown cases (idempotent `PUT`, size limits) is injectable and covered both ways.

## M5: News connector end to end against mocks

List channels, map channels by name (human confirms), list posts, copy as drafts (no `published`, no `notificationChannels`), rewrite media and `image` URLs, `externalID` rules, validation reads.

Acceptance: a test asserts no request body ever contains `notificationChannels` or `published`. Mock end to end run passes plan section 5.7 validations. Live run is a human task: bring back typed output only.

## M6: CLI workflow

`plan`, `dry-run`, `approve` (typed destination hostname and counts), `run`, `resume`, `validate`, `report` (redacted default, full by flag), `cleanup`. Event stream on stdout.

Acceptance: dry run performs zero writes (mock records zero non-read requests). `run` without prior approval fails. Report redaction test. Rerun of a completed plan creates nothing.

## M7 (gated on Phase 0): Pages connector and upgrade worklist

Unlocks only when `docs/phase0-results.md` answers U7 (what the destination creates and whether it is upgrade-eligible). Classic page listing and classification, draft creation, link rewrite pass via `PUT /pages/{id}`, upgrade worklist (CSV and local HTML), re-validate command.

Acceptance: worklist ordering low risk first, no content in redacted mode, link rewrite idempotent, tests encode Phase 0 findings explicitly and cite them.

## M8 (gated on Window Gate): evidence pipeline and release

Egress capture lab scripts, SBOM, signed build, release attestation page, runbook. The Tauri desktop shell is a separate decision and is not part of M8.

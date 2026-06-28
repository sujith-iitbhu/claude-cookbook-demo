# CYCLE 1 — Tiny URL Shortener (in-memory)

**Linear:** SUJ-5 · **Specs:** `specs/PROJECT.md`, `specs/TECH_SPEC.md`
**Goal:** Ship the full in-memory URL shortener API — create, redirect, stats,
TTL, and rate limiting — behind a JSON REST interface.
**Duration:** 1 cycle (~1 day, S). One cycle covers the whole feature.

## Story Map / Dependency Order

```
S1 (scaffold + store) ─┬─▶ S2 (create) ─┬─▶ S3 (redirect + analytics) ──▶ S4 (stats)
                       │                 └─▶ S5 (TTL/expiry)
                       └─────────────────────▶ S6 (rate limiting)
S1 ──▶ S7 (health + error envelope)  [parallel with S2+]
```

S1 is the foundation. S2 unblocks S3/S5; S3 unblocks S4. S6 and S7 depend only
on S1 and can proceed in parallel.

---

## S1 — Project scaffold and in-memory store

**Description:** Stand up the FastAPI app skeleton and the `LinkStore` (dict-backed,
lock-guarded) plus the base62 code generator. No HTTP endpoints with behavior yet
beyond wiring. Establishes module layout from TECH_SPEC §1.

**Effort:** S
**Dependencies:** none

**Acceptance Criteria**
- **Given** a fresh app, **when** the store is asked to create a record for a
  valid URL, **then** it returns a unique 7-char base62 code and stores a
  `LinkRecord` retrievable by that code.
- **Given** a code that collides with an existing one, **when** generating,
  **then** the generator retries (up to N) and never returns a duplicate; if it
  exhausts retries it raises an error surfaced as `500`.
- **Given** codes are generated, **when** inspecting the source of randomness,
  **then** `secrets` (not `random`) is used.
- **Given** concurrent create/read calls, **when** they run, **then** access is
  guarded by a lock with no lost updates (covered by a unit test).

---

## S2 — Create short link endpoint (`POST /links`)

**Description:** Implement `POST /links` with Pydantic validation per TECH_SPEC
§3.1, returning `201` with the short URL payload. Includes input validation and
the `422` error path. `expires_in` parsing only (enforcement is S5).

**Effort:** S
**Dependencies:** S1

**Acceptance Criteria**
- **Given** a valid `http(s)` URL, **when** `POST /links` is called, **then** the
  response is `201` with `code`, `short_url`, `long_url`, `created_at`, and
  `expires_at` (null when no `expires_in`).
- **Given** `short_url`, **when** it is built, **then** it uses the configured
  `BASE_URL` prefix.
- **Given** a missing, empty, non-string, non-`http(s)` (e.g. `javascript:`,
  `data:`), or over-length (> `MAX_URL_LENGTH`) URL, **when** `POST /links` is
  called, **then** the response is `422` with the `invalid_request` error envelope.
- **Given** an `expires_in` that is ≤ 0, non-integer, or > `MAX_TTL_SECONDS`,
  **when** `POST /links` is called, **then** the response is `422`.
- **Given** an unexpected extra field in the body, **when** `POST /links` is
  called, **then** it is rejected (`extra="forbid"`).

---

## S3 — Redirect endpoint with click analytics (`GET /{code}`)

**Description:** Implement `GET /{code}` redirecting to the stored URL and
incrementing analytics counters per TECH_SPEC §3.2.

**Effort:** S
**Dependencies:** S2

**Acceptance Criteria**
- **Given** an existing, non-expired code, **when** `GET /{code}` is called,
  **then** the response is `302` with `Location` set to the original URL.
- **Given** a successful redirect, **when** it completes, **then** `clicks` is
  incremented by 1 and `last_access` is updated to the current UTC time.
- **Given** an unknown code, **when** `GET /{code}` is called, **then** the
  response is `404` with the `not_found` envelope.
- **Given** concurrent redirects to the same code, **when** they run, **then**
  the final `clicks` count equals the number of successful redirects (no lost
  increments).

---

## S4 — Stats endpoint (`GET /links/{code}/stats`)

**Description:** Expose per-link analytics per TECH_SPEC §3.3.

**Effort:** S
**Dependencies:** S3

**Acceptance Criteria**
- **Given** an existing code, **when** `GET /links/{code}/stats` is called,
  **then** the response is `200` with `code`, `long_url`, `clicks`, `created_at`,
  `expires_at`, `last_access`, and `expired`.
- **Given** a link that has been redirected N times, **when** stats are queried,
  **then** `clicks` equals N.
- **Given** an expired-but-not-yet-evicted code, **when** stats are queried,
  **then** the response is `200` with `expired: true`.
- **Given** an unknown or evicted code, **when** stats are queried, **then** the
  response is `404`.

---

## S5 — Link expiration / TTL enforcement

**Description:** Enforce TTL lazily on access and add the optional background
sweep per TECH_SPEC §7. Builds on the `expires_at` set in S2.

**Effort:** S
**Dependencies:** S2 (and integrates with S3/S4)

**Acceptance Criteria**
- **Given** a link created with `expires_in`, **when** the current time is before
  `expires_at`, **then** `GET /{code}` still returns `302`.
- **Given** a link whose `expires_at` has passed, **when** `GET /{code}` is
  called, **then** the response is `410` with the `expired` envelope.
- **Given** a link with no `expires_in`, **when** any amount of time passes,
  **then** it never expires.
- **Given** expiry boundary times, **when** tested at just-before and just-after
  `expires_at`, **then** behavior flips exactly at the boundary.
- **Given** the background sweep runs, **when** expired records exist, **then**
  they are evicted and subsequently return `404`.
- **Given** all time handling, **when** timestamps are produced, **then** they are
  UTC and serialized as ISO-8601 `Z`.

---

## S6 — Creation rate limiting

**Description:** Add in-process sliding-window rate limiting on `POST /links`
only, per TECH_SPEC §6.

**Effort:** S
**Dependencies:** S1 (applies to S2's endpoint)

**Acceptance Criteria**
- **Given** fewer than `RATE_LIMIT_MAX` creates within the window from one IP,
  **when** another create arrives, **then** it succeeds (`201`).
- **Given** `RATE_LIMIT_MAX` creates already in the window from one IP, **when**
  another create arrives, **then** the response is `429` with the `rate_limited`
  envelope and a `Retry-After` header.
- **Given** the window rolls forward past old timestamps, **when** a previously
  limited IP retries, **then** it succeeds again.
- **Given** rate limiting, **when** `GET /{code}` and stats are called, **then**
  they are never limited.
- **Given** the limiter, **when** the client IP is determined, **then** the socket
  peer IP is used and `X-Forwarded-For` is not trusted by default.
- **Given** an IP whose deque empties, **when** memory is inspected, **then** the
  empty deque is evicted (bounded memory).

---

## S7 — Health endpoint and error envelope

**Description:** Add `GET /healthz` and the shared error envelope + exception
handlers per TECH_SPEC §3.5 and §4. The envelope is consumed by S2–S6.

**Effort:** S
**Dependencies:** S1

**Acceptance Criteria**
- **Given** the running app, **when** `GET /healthz` is called, **then** the
  response is `200` with `{"status": "ok"}`.
- **Given** any handled error (`422`/`404`/`410`/`429`), **when** it is returned,
  **then** the body matches the `{ "error": { "code", "message" } }` shape with
  the correct `error.code` per TECH_SPEC §4.

---

## Cycle Acceptance / Definition of Done

- All 7 stories' AC met.
- Test suite passes with ≥80% line coverage overall; redirect, TTL, and rate-limit
  paths ≥95% (per project testing rules).
- `ruff check` and `ruff format` clean; `bandit` shows no HIGH/MEDIUM findings.
- Endpoints documented via FastAPI's auto OpenAPI.

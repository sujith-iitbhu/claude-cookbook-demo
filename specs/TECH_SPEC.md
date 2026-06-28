# TECH SPEC: Tiny URL Shortener (in-memory)

Implements Approach 1 from `specs/PROJECT.md`: FastAPI + in-memory store +
in-process rate-limit middleware. JSON REST API only, single process,
demo-grade (no durability).

## 1. Architecture Overview

```
                      ┌─────────────────────────────────────────┐
   client ──HTTP──▶   │              FastAPI app                 │
                      │                                          │
                      │  RateLimitMiddleware (create path only)  │
                      │            │                             │
                      │            ▼                             │
                      │   ┌──────────────┐    ┌───────────────┐  │
                      │   │  routes/api  │───▶│   LinkStore   │  │
                      │   │  (handlers)  │    │ (dict in mem) │  │
                      │   └──────────────┘    └───────────────┘  │
                      │            │                  ▲          │
                      │            ▼                  │          │
                      │      code generator    expiry check      │
                      └─────────────────────────────────────────┘
```

- **RateLimitMiddleware** — applies only to the create endpoint; tracks per-IP
  request timestamps in an in-memory sliding window; rejects with `429` when the
  window is full.
- **routes/api** — request handlers; validate input via Pydantic, call the store,
  shape responses/redirects.
- **LinkStore** — the single source of truth: a dict mapping `code -> LinkRecord`,
  plus the code generator and expiry/analytics logic. Thread-safety via a lock
  (Uvicorn default is single worker but may use a threadpool for sync paths).
- **code generator** — 7-char base62 random code, retried on collision.

### Component / module layout (proposed)
```
src/
  main.py            # FastAPI app wiring, middleware registration
  store.py           # LinkStore, LinkRecord, code generation, expiry logic
  rate_limit.py      # sliding-window limiter + middleware
  schemas.py         # Pydantic request/response models (frozen)
  errors.py          # error envelope + exception handlers
tests/
  unit/              # store, code-gen, rate limiter, expiry
  integration/       # API endpoint flows via FastAPI TestClient
  conftest.py
```

## 2. Data Model

In-memory only. No tables, no migrations.

### LinkRecord (internal, frozen dataclass)
| Field         | Type            | Notes                                              |
|---------------|-----------------|----------------------------------------------------|
| `code`        | `str`           | 7-char base62 primary key.                         |
| `long_url`    | `str`           | Validated absolute `http(s)` URL.                  |
| `created_at`  | `datetime` (UTC)| Set at creation.                                   |
| `expires_at`  | `datetime|None` | `created_at + expires_in`; `None` = never expires. |
| `clicks`      | `int`           | Incremented on each successful redirect.           |
| `last_access` | `datetime|None` | UTC timestamp of most recent successful redirect.  |

> `clicks` and `last_access` are mutable counters. Per project style, the record
> is a frozen DTO; the store owns mutation by replacing the record (or holds the
> counters in a small separate mutable structure keyed by code). Implementer's
> choice — the contract is what matters.

### Store structure
- `links: dict[str, LinkRecord]` — primary index by code.
- `_lock: threading.Lock` — guards reads/writes and counter increments.

## 3. API Contracts

All responses are JSON except the redirect. All errors use the envelope in §4.
Base path: `/`.

### 3.1 Create short link
`POST /links`

Request body:
```json
{
  "url": "https://example.com/some/very/long/path?x=1",
  "expires_in": 3600
}
```
- `url` (required) — must be a valid absolute `http`/`https` URL. Validated with
  Pydantic (`AnyHttpUrl`). Reject other schemes.
- `expires_in` (optional, int seconds, > 0) — TTL. Omitted/null = never expires.
  Suggested upper bound: 1 year (reject larger as `422`).

`201 Created`:
```json
{
  "code": "aZ3kPq7",
  "short_url": "http://localhost:8000/aZ3kPq7",
  "long_url": "https://example.com/some/very/long/path?x=1",
  "expires_at": "2026-06-29T13:00:00Z",
  "created_at": "2026-06-29T12:00:00Z"
}
```
- `short_url` is built from a configured base URL (env `BASE_URL`, default
  `http://localhost:8000`).

Errors: `422` invalid/missing URL or bad `expires_in`; `429` rate limit exceeded.

### 3.2 Redirect
`GET /{code}`

- `302 Found` with `Location: <long_url>` when the code exists and is not expired.
  Increments `clicks`, updates `last_access`.
- `404 Not Found` when the code does not exist.
- `410 Gone` when the code exists but `expires_at` has passed. (Expired records
  may be lazily evicted on access.)

> `302` (temporary) is chosen over `301` so analytics keep counting and so
> intermediaries do not cache the mapping past expiry.

### 3.3 Stats
`GET /links/{code}/stats`

`200 OK`:
```json
{
  "code": "aZ3kPq7",
  "long_url": "https://example.com/...",
  "clicks": 42,
  "created_at": "2026-06-29T12:00:00Z",
  "expires_at": "2026-06-29T13:00:00Z",
  "last_access": "2026-06-29T12:45:10Z",
  "expired": false
}
```
Errors: `404` unknown code. Stats remain queryable for expired-but-not-evicted
links (`expired: true`); `404` once evicted.

### 3.4 Delete (optional, lean nice-to-have)
`DELETE /links/{code}` → `204 No Content`, or `404` if unknown. Include only if
trivial; not required by the core asks.

### 3.5 Health
`GET /healthz` → `200 {"status": "ok"}`. Used for liveness/smoke tests.

## 4. Error Envelope

All non-redirect error responses share one shape:
```json
{
  "error": {
    "code": "rate_limited",
    "message": "Too many create requests; retry later."
  }
}
```
| HTTP | `error.code`      | When                                            |
|------|-------------------|-------------------------------------------------|
| 422  | `invalid_request` | Bad/missing URL or invalid `expires_in`.        |
| 404  | `not_found`       | Unknown code.                                    |
| 410  | `expired`         | Code expired.                                    |
| 429  | `rate_limited`    | Per-IP creation window exceeded. Include `Retry-After` header. |

## 5. Code Generation

- Alphabet: base62 `[A-Za-z0-9]`, length **7** (≈3.5 × 10¹² space).
- Source of randomness: `secrets` (not `random`) to avoid predictable codes.
- Collision handling: generate, check `links`; retry up to N (e.g. 5) times; if
  still colliding, return `500 internal_error` (astronomically unlikely at demo
  scale).

## 6. Rate Limiting

- **Policy:** 20 create requests per 60s per client IP (sliding window).
- **Algorithm:** per-IP `deque` of request timestamps; on each create, drop
  timestamps older than the window, reject with `429` if the remaining count
  ≥ limit, else append now.
- **Client identity:** remote IP. Note `X-Forwarded-For` is spoofable and is NOT
  trusted by default (demo runs without a trusted proxy); use the socket peer IP.
- **Scope:** applies to `POST /links` only. Redirects and stats are not limited.
- **Headers on 429:** `Retry-After` (seconds until window frees).
- **Config:** `RATE_LIMIT_MAX` (default 20), `RATE_LIMIT_WINDOW_SECONDS`
  (default 60) via env.
- **Memory hygiene:** evict per-IP deques that empty out to bound memory.

## 7. Expiration (TTL)

- `expires_at = created_at + expires_in` (UTC) when `expires_in` provided.
- **Lazy enforcement:** every `GET /{code}` and stats read checks `now >=
  expires_at`; if so, treat as expired (`410` on redirect) and may evict.
- **Optional sweep:** a periodic background task (e.g. every 60s) removes expired
  records to bound memory. Lazy check is the correctness guarantee; sweep is an
  optimization.
- All time handling in **UTC**; timestamps serialized as ISO-8601 `Z`.

## 8. Edge Cases

- Empty / missing / non-string `url` → `422`.
- Non-`http(s)` scheme (e.g. `javascript:`, `ftp:`, `data:`) → `422`. This also
  mitigates open-redirect-to-dangerous-scheme abuse.
- `expires_in` ≤ 0, non-integer, or above max → `422`.
- Extremely long URL → enforce a max length (e.g. 2048 chars) → `422` over limit.
- Unknown code on redirect/stats → `404`.
- Expired code → `410` (redirect) / `expired: true` then `404` after eviction.
- Code-generation collision exhaustion → `500`.
- Concurrent redirects to same code → counter increments guarded by lock.
- Restart → all links/counters/limits reset (documented, expected).

## 9. Security Considerations

- **Open redirect:** the service redirects to arbitrary user-supplied URLs by
  design. Restrict schemes to `http`/`https` only; do not allow `javascript:`,
  `data:`, etc. Document that the redirect target is attacker-controllable and
  callers/consumers should treat short links as untrusted.
- **Input validation:** all input validated via Pydantic with
  `extra="forbid"`; URL length capped.
- **No secrets:** service holds no credentials; nothing to leak. No `.env`
  secrets required beyond optional `BASE_URL` / rate-limit config (non-secret).
- **Randomness:** codes from `secrets` to prevent enumeration/guessing of others'
  links (mild — links are not access-controlled, but avoids trivial scraping).
- **Rate limiting** is the primary abuse control for creation. Redirect endpoints
  are unauthenticated and unlimited by design (demo).
- **No SQL / no pickle:** no database and no deserialization of untrusted data,
  so those rule classes are N/A.
- **DoS note:** in-memory store grows unbounded without the sweep + deque
  eviction; both are specified in §6/§7 to bound memory.

## 10. Performance Considerations

- All operations are O(1) dict lookups / deque ops; no external I/O.
- Single lock is fine at demo scale; if it ever became a bottleneck, shard by
  code prefix (out of scope).
- No caching layer needed — the store *is* memory.

## 11. Configuration (env vars, all non-secret)

| Var                         | Default                 | Purpose                          |
|-----------------------------|-------------------------|----------------------------------|
| `BASE_URL`                  | `http://localhost:8000` | Prefix for `short_url`.          |
| `RATE_LIMIT_MAX`            | `20`                    | Max creates per window per IP.   |
| `RATE_LIMIT_WINDOW_SECONDS` | `60`                    | Rate-limit window length.        |
| `CODE_LENGTH`               | `7`                     | Short-code length.               |
| `MAX_URL_LENGTH`            | `2048`                  | Reject longer URLs.              |
| `MAX_TTL_SECONDS`           | `31536000`              | Upper bound for `expires_in`.    |

## 12. Testing Strategy

Per project rules (pytest, marks, ≥80% coverage, TDD):
- **Unit:** code generation (length/alphabet/collision retry), store CRUD + lock,
  expiry boundary (just-before vs just-after `expires_at`), sliding-window limiter
  (under/at/over limit, window roll-off).
- **Integration (TestClient):** create→redirect→stats happy path; `404`/`410`
  paths; `422` on bad URL/scheme/`expires_in`; `429` when exceeding the window;
  `Retry-After` present on `429`.
- **Edge:** max URL length, disallowed schemes, `expires_in` bounds.
- Critical paths (redirect correctness, rate limit, expiry) targeted ≥95%.
```

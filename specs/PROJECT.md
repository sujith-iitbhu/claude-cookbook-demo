# PROJECT: Tiny URL Shortener (in-memory)

## Problem Statement

We need a minimal URL-shortening service exposed as a JSON REST API. Callers
submit a long URL and receive a short code; requesting that code redirects to the
original URL. The service is demo-grade: state lives in memory (lost on restart),
access is open (no auth), and short codes are randomly generated (no custom
aliases). On top of the core shorten/redirect loop, three operational features
are in scope: per-link click analytics, optional link expiration (TTL), and
creation rate limiting per client IP.

## Scope

**In scope**
- `POST` to create a short link from a long URL (random 7-char base62 code).
- `GET /{code}` redirect to the original URL.
- Click analytics: click count (and first/last access timestamps) per code.
- Optional TTL: caller-supplied `expires_in` seconds; expired links return `410 Gone`.
- Rate limiting on creation: sliding window, 20 creates/min per IP, `429` over limit.
- JSON REST API only.

**Out of scope**
- Persistent storage (Postgres, Redis, SQLite) and durability across restarts.
- Authentication, user accounts, per-user link ownership, API keys.
- Custom aliases / vanity codes.
- HTML UI or web form.
- Horizontal scaling, multi-process coordination, distributed rate limiting.
- Link editing/deletion management UI (a delete endpoint is optional, see TECH_SPEC).

## Solution Approaches

### Approach 1 — FastAPI + in-memory dict + in-process middleware (RECOMMENDED)
A single FastAPI application. Links stored in a module-level dict keyed by code.
Click counts and timestamps stored alongside each record. TTL enforced lazily at
read time (compare `expires_at` to now) plus optional background sweep. Rate
limiting via lightweight in-process middleware tracking per-IP timestamps in a
deque.

- **Pros:** Smallest moving-parts footprint that still satisfies all four asks;
  FastAPI gives automatic request validation (Pydantic), OpenAPI docs, and clean
  redirect responses for free; matches the project's Python/FastAPI house style.
- **Cons:** Single-process only; rate-limit and analytics state are not shared
  across workers; everything lost on restart (acceptable per requirements).
- **Effort:** S (< 1 day).
- **Risk:** Low.

### Approach 2 — Flask + dict + decorator-based rate limit
Same in-memory model on Flask instead of FastAPI; rate limiting via a custom
decorator; manual JSON (de)serialization.

- **Pros:** Minimal dependencies; very small surface.
- **Cons:** No built-in request validation or OpenAPI; more hand-rolled
  boilerplate for input validation and error envelopes; diverges from the
  FastAPI/Pydantic conventions in the project rules.
- **Effort:** S.
- **Risk:** Low-Medium (more hand-written validation = more edge-case bugs).

### Approach 3 — FastAPI + SQLite via SQLAlchemy
Same API, but persist links and counters in a local SQLite file for durability.

- **Pros:** Survives restarts; trivial upgrade path to Postgres later.
- **Cons:** Explicitly out of scope (user chose in-memory); adds schema,
  migrations, and session management for no demo benefit; over-engineered.
- **Effort:** M.
- **Risk:** Low, but scope creep.

## Recommendation

**Approach 1 (FastAPI + in-memory dict + in-process middleware).** It is the
leanest design that still delivers analytics, TTL, and rate limiting, while
aligning with the project's FastAPI + Pydantic + frozen-DTO conventions and
giving validation and OpenAPI docs for free. Approaches 2 and 3 either add
hand-rolled boilerplate or pull in persistence that the requirements explicitly
exclude.

## Estimated Effort

**S — under 1 day.** Single application module plus a small store module and a
test suite; no new infrastructure, clear established pattern.

## Comm:ntator (case study)

<p align="center"><img src="assets/logo.png" alt="Comm:ntator" width="600"/></p>

Multi-tenant SaaS that runs end-to-end AI editorial pipelines.
Built solo since February 2026. Live production, alpha.
Scope: ~93K production LOC, ~130K test LOC, single maintainer.

This repository is a public case study. **Source code is private.**
What follows is the architecture, the engineering choices that
mattered, and the operational discipline that keeps it stable.

### Shape

Each tenant owns multiple isolated workspaces. Each workspace runs
its own pipeline:

```
Source monitoring
  -> RAG relevance scoring
  -> Multi-step content generation
  -> Scheduled publication
```

Pipeline steps are async agents. Per-project asyncio locks prevent
concurrent runs from racing on the same workspace. SSE streams
progress to the frontend in real time.

### Stack

| Layer       | Choice                                            |
|-------------|---------------------------------------------------|
| Backend     | FastAPI (Python 3.11), async SQLAlchemy + asyncpg |
| Frontend    | Next.js 15 (App Router), React 19, TypeScript     |
| Vector DB   | ChromaDB 1.5.1 (multilingual-e5-large embedder)   |
| Relational  | PostgreSQL                                        |
| Cache       | Redis                                             |
| Search      | SearXNG (self-hosted)                             |
| Proxy       | Caddy (auto HTTPS, zero-downtime reload)          |
| Auth        | PyJWT (HS256) + bcrypt + Google OAuth + TOTP MFA  |
| i18n        | next-intl (EN, RU, ES)                            |
| Deploy      | Docker Compose, self-hosted runners on Oracle Cloud |
| Backups     | Cloudflare R2, GPG-encrypted, GFS retention       |

### Engineering principles

#### 1. Architecture LAW (CI-enforced)

Eight invariants live as architecture tests in
`backend/tests/architecture/`. CI fails the build on violation.
This is mechanical enforcement, not developer discipline.

1. **Multi-tenant isolation.** `project_id` is required (not
   `Optional`) in every agent signature. Nullable typing in agent
   code is rejected at CI time.
2. **Model-id normalization.** Every write path for LLM keys
   passes through a per-provider normalizer. Triple defense:
   endpoint helper, SQLAlchemy `@validates`, runtime read-path.
3. **No module-level state.** New module-level `dict / list /
   set / Counter / Lock / ContextVar / TTLCache` in `app/`
   requires an inline marker with an explicit justification.
4. **Dual-write 3-lock.** Any integration that writes to Postgres
   plus an external store (Chroma, R2, external API) must have:
   a single canonical deletion path, a scheduled reconciler that
   reports drift, and per-row savepoint isolation in batch
   operations.
5. **`setInterval` frontend floor.** Numeric `setInterval` calls
   in `frontend/src/` must be >= 15000ms. Sub-15s requires an
   inline marker with a reason. Protects against accidental
   polling storms that translate into LLM cost.
6. **i18n locale parity.** `en.json`, `ru.json`, `es.json` must
   carry identical nested key sets. Adding or removing a key in
   one locale without updating all three fails CI.
7. **No vendor names in planning docs.** Prose in `.planning/`
   must not name specific providers or models. Architecture is
   expressed in provider-agnostic tiers (LITE, complex,
   user's key). The user owns vendor choice per project; the
   router enforces the routing matrix, not vendor identity.
8. **Soft-warning baseline.** `ruff`, `mypy`, `pip-audit`, and
   `npm audit` run as HARD CI gates via baseline diff. Any new
   warning absent from the frozen baseline fails CI. Baseline
   regeneration requires an explicit double-gate (CLI flag plus
   env var) to prevent accidental drift.

Each rule has a CI test. Each change to the LAW requires a
CHANGELOG entry with date, action, and incident reference.

#### 2. Cost-quality routing

Every change answers two questions: cheapest path for the user,
and highest quality the user expects from a professional tool.
Both axes are maximized together. Cheaper at the cost of quality
is a regression, not a fix. Quality at the cost of price is debt,
not a fix.

In practice:
- LITE-tier tasks (`lang_detect`, `relevance`, `dedup`,
  `source_filter`) route to the user's free-tier provider key.
  Never to the complex-tier provider.
- Complex-tier tasks (`generation`, `style_analysis`, `chatbot`,
  `translation`) route to the user's complex-tier provider key.
  `thinking_budget=0` for tasks listed in `NO_THINKING_TASKS`.
- Provider choice per project is owned by the user. The router
  enforces the routing matrix; vendor identity is configuration,
  not architecture.
- Per-user spend cap: $0.10/day hard, $0.05/day soft. Enforced
  in `token_tracker`. Returns `429 user_daily_cap` on breach.
- Every LLM loop has a batch limit, a per-project `asyncio.Lock`,
  and an inter-call sleep.
- Frontend `setInterval` minimum is 15s (LAW §5).

#### 3. Tenant context

- `TenantContextMiddleware` sets a `ContextVar` from the
  `X-Project-Id` header on every request.
- Every DB query is filtered by `project_id` or `user_id`. No
  exceptions.
- ChromaDB collections are scoped per project (`editor_p{id}`,
  `expertise_p{id}`). No global collections.
- SSE handlers re-set the `ContextVar` inside the event
  generator. The middleware resets it before the generator
  starts iterating, so the explicit set/reset inside `finally`
  is mandatory.

#### 4. Live-state verification before diagnosis

Operational rule: on any production complaint, the first action
is a direct `psql` query plus `docker logs --since Nh`, before
the first hypothesis. No diagnosis from cached subagent reports
or tech-debt tables. Past incidents traced ~75% misdiagnoses to
skipping this step.

### Selected technical decisions

**LLM key encryption.** Per-project LLM keys are Fernet-encrypted
at rest. Master key is derived from `SECRET_KEY`. Legacy XOR
blobs are read via a fallback path during in-place migration.
Future: external KMS, plaintext never resident in the process.

**Pipeline concurrency.** Complex-tier LLM calls run under
`Semaphore(3)` (raised from 1 once observability showed safe
headroom). DB pool 20 + 40 overflow. Per-project `asyncio.Lock`
on runaway-prone agents. Cancellation propagates through the
agent chain.

**Authentication.** JWT in HttpOnly cookie via a same-origin
Next.js proxy. SSE auth uses a short-lived HMAC ticket
(`POST /auth/sse-ticket`, 30s TTL). JWT in URL is rejected.
CSRF double-submit on mutating verbs. TOTP MFA with recovery
codes, step-up auth for sensitive operations, in-process replay
tracker (single-worker today, Redis migration planned).

**PII scanner.** Hybrid design: regex always (deterministic,
free, language-agnostic). Presidio NER conditionally for
`{EN, ES, DE, FR, IT, PT, NL}`. `detector_used` annotation is
part of the contract so downstream code never has to guess
which detector ran.

**Backups.** `pg_dump` plus ChromaDB snapshot, GPG-encrypted,
rclone to R2. Systemd timer 03:00 UTC daily. Separate verifier
05:00 UTC with Telegram alert on silent failure. GFS retention
7 daily + 4 weekly + 6 monthly. Restore-test runs on the staging
server with a dedicated compose override.

**Deploy gates.** `deploy.sh` refuses to ship if:
- CI is not green on the exact commit SHA, or
- staging has not passed within the last 24h for this commit, or
- the post-deploy e2e suite is not green for this commit.

A run still in progress on any of the three is distinguished from
a failed run, so the operator gets «retry in 2-3 min» instead of
a generic «missing» that would push them to override.

`FORCE_DEPLOY=1` requires `FORCE_DEPLOY_REASON='...'` and emits
a Telegram audit alert listing every gate bypassed (CI, staging,
e2e), diff stats, commit log since last staging-verified SHA, and
secret redaction (Fernet tokens, `SECRET_KEY`, `BOT_TOKEN`,
`API_KEY`, `PASSWORD`, `POSTGRES_PASSWORD` are replaced inline
before send).

Caddy reload is in-place: existing connections drain on the old
config, new ones land on the new one. Pre-build disk check
prunes Docker cache when free space < 5GB.

### CI

Self-hosted runners on a dedicated 50GB block volume, separate
from production storage. Pre-flight disk inventory before every
long job (peak estimate, headroom budget >=10GB, structural,
not "I'll remember").

100% line coverage gate (`--cov-fail-under=100`) on the coverage
branch, ~5700 tests across 22 architecture tests plus full
behavioral suite. Architecture tests run against the source tree.
Integration tests run against a real Postgres container. Frontend
type-check is mandatory. Dependabot runs weekly on 40-char
SHA-pinned actions.

### Roadmap (M2)

Migration to `pgvector` consolidates the vector store into
Postgres and removes the dual-write surface entirely. 29 phases
across 5 waves; Wave 1 (foundation hardening) in progress.
Roadmap is the single source of truth.

### License

Source is not public. This case study (text and diagrams) is
released under [CC BY-NC 4.0](LICENSE).

### Contact

Goumar Zainoulline. Granada, Spain. goumar.zainoulline@gmail.com.
LinkedIn: [linkedin.com/in/goumar](https://linkedin.com/in/goumar/).

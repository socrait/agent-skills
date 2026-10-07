<!--
Paste into your AGENTS.md / CLAUDE.md and trim to your house rules. Skills load only when
invoked; these rules apply while code is being written. Replace `app/` paths and the tenant
vocabulary (store_id) with your own.
-->

## When to invoke a skill before writing code

- **data-access** — adding or changing a repository, record type, query filter, transaction boundary, or migration; any query grouping or filtering by date/time.
- **security** — adding or changing an endpoint's authorization, response fields, or query scope; client-supplied ids flowing into a bulk lookup; building an LLM prompt; a catch-all `except Exception` in a route; a log line that may carry PII.
- **design** — a new class, module, package, public interface, ABC, message schema, or third-party SDK; a new orchestrator or a service that wants to call another service; moving code between layers.
- **resilience** — a new `try/except`, domain exception, info-or-higher log site, metric, batch operation, or worker entrypoint; a new error response.
- **performance** — code on a hot path (routes, workers, streaming); collections of unknown size; a new cache or external connection; `await` inside a loop.
- **testing** — before writing any test file.
- **review-all** — after a nontrivial change, before opening a merge request.

## Data access

- Every query on tenant-owned data — reads, bulk lookups, updates, deletes — is constrained to the caller's tenant, applied the same way everywhere. Cross-tenant system queries are rare and conspicuous.
- SQLAlchemy 2.0 only: `session.execute(select(...))`, never `session.query(...)`.
- Orchestrator entry points own the transaction through a reentrant unit of work: the outermost commits or rolls back once, nested calls join it. Services, repositories, routers, and session dependencies never commit.
- Single round-trip mutations: `update(...).where(<precondition>).returning(...)`, not select-then-update.
- `selectinload` when touching a relationship in a loop; bulk `*_by_ids` instead of lookups in a loop.
- ORM records never leave the data layer; map to domain models first.
- Queries return `None` on not-found, mutations raise. `IntegrityError` is caught at `flush()` in the repository and translated to a domain exception.
- Background jobs open one session per job and commit per unit of work; sessions are never shared across jobs or tasks.
- Store UTC, display local (`ZoneInfo(tz)`); group by local date; cache keys include `tz` when it affects output.
- Migrations are linear, deterministic, and reversible (`alembic check` in CI; new databases via `create_all()` + stamp head); idempotent only for backfills, non-transactional operations, or backported revisions, using native `if_not_exists`. They match the records 1:1, use `op.f()` names, give new tables `created_at`/`updated_at` (indexed) and `deleted_at`, and use partial unique indexes (`deleted_at IS NULL`) on soft-deleting tables.

## Security

- Every endpoint's auth requirement is declared in its signature or router; public endpoints are public on purpose.
- Every by-id access proves the caller may touch *that object*: load it constrained to the caller's tenant, and treat "not found under the constraint" as 404. Child resources are checked through their parent.
- Client-supplied id lists are intersected with the set the authorized parent contains before use.
- Privileged operations check a role or permission, not just authentication.
- Responses carry only what the caller's permission covers — aggregate permissions get aggregate responses. Request models never accept tenant, owner, role, price, or status fields the client shouldn't control.
- Not visible → 404; visible but not permitted → 403; no internal reason codes in responses.
- Never `str(e)` in a response; generic message plus server-side log.
- Customer PII only at `debug` log level, truncated; never in error responses; LLM prompts carry only the minimum PII; PII cache writes have a bounded TTL and fail loud on an invalid one.

## Design

- No `dict` in public interfaces. Route handlers declare a Pydantic response model (`response_model=` or the return annotation) and never return a bare dict.
- Third-party SDKs only inside `clients/`, which translate SDK types and exceptions to domain ones.
- Constructor injection only. Seams (repositories, I/O clients, flag clients, LLM clients) are required; impl collaborators are optional keyword-only with their production construction as default; call-time collaborators are factories defaulting to the class. Constructors only store.
- No public method calls another public method on the same class — public methods carry entry concerns (logging, metrics, transactions); share logic through a private method.
- `_`-prefix internal helpers (including closures); never `_`-prefix types in public signatures or module files.
- Layering: routers → orchestrators → services / repositories / clients, dependencies pointing downward only. Each subsystem has one orchestrator (its facade); services don't call peer services; other subsystems go through the orchestrator, never its internals. Orchestrators delegate to each other only downward along a declared, acyclic order; anything against the order goes through a task or event.
- A new type, service, or table states what existing thing it could be a projection of and why that fails. No intermediate types duplicating domain models; no identity mappers.
- No classless bare-function modules in `services/` — a pure transform of one domain model is a method on that model.
- Validate an operation's result at the one choke point closest to it.
- Optional parameters are keyword-only. Full words in function names. Title-cased acronyms (`Ttl`, `Id`).
- Package-level imports in three groups; `__init__.py` re-exports relatively from direct children only.
- Feature flags through the flag system, never boolean settings.
- Comments explain why; never cite spec or ticket section ids; no docstrings on tests or private methods; public methods that raise domain exceptions get one.

## Resilience

- Catch only to log with local context, translate to a domain exception, or genuinely recover. No dead catch clauses. Broad `except Exception` only at top-level boundaries.
- Domain exceptions live in the core exceptions module and carry an error code; messages are controlled — never `str(exc)` of a caught exception.
- Invariant violations raise. Upserts and catch-and-continue need justification; never silently rewrite identity-bearing fields. Deliberate recovery is surfaced to the caller and logged.
- Batch operations isolate per-item failures and summarize via a stats object.
- Error responses follow one contract (stable code + static message), produced by app-level exception handlers. 5xx never exposes domain codes. Anti-enumeration shares a code and message across causes, while logging the real cause.
- structlog: event name plus key-value fields (`log.info("refund_issued", refund_id=...)`), never f-strings or interpolated messages; correlating ids bound once (contextvars per request, `bind()` per operation); `log.exception` for tracebacks; values truncated and secrets masked.
- Metrics count business units with low-cardinality attributes only.

## Performance

- Stream, don't buffer; batch, don't loop.
- No sync I/O hidden deeper in an async call chain (a sync client or file read inside a service an async route calls).
- Independent I/O via `asyncio.gather` (bounded), not sequential awaits.
- Sets for membership tests in loops.
- No unbounded caches; cache keys include tenant and every output dimension; invalidation decided before the cache exists.
- External clients pooled and created once.

## Testing

- Integration over unit; unit only for self-contained functions.
- Test through the public interface; test the unit under test only.
- Real database, fake network; library mock facilities over `patch`; configure mocks via fixtures, not private attributes; verify persistence through repositories.
- Each layer owns its edge cases; the layer above asserts only the translation.
- Shared fixtures live in `conftest.py` at the level that covers their users (large groups via `pytest_plugins`); tests never import fixtures. Search existing conftests before writing one.
- `sample_*` records are never mutated — extend the `make_*` factory.
- No tests for thin glue; no duplicate coverage; feature flags tested ON and OFF; every tenant-scoped read and by-id endpoint has a cross-tenant denial test.
- Tests that mutate shared external state clean up. External-service tests are opt-in markers; probabilistic assertions use repeat-and-threshold, never reruns.

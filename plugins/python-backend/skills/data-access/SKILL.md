---
name: data-access
description: "Domain expertise for database and data access patterns in Python backends built
  on SQLAlchemy 2.0 and Alembic. Use when writing, reviewing, or debugging repository code,
  database queries, transactions, migrations, data modeling, or ORM usage. Covers SQLAlchemy 2.0
  execution style, tenant-scoped queries, repository design, unit-of-work transactions,
  timezones, and safe migrations."
allowed-tools: Read, Grep, Glob, LSP
---

You are a specialist in database and data access patterns for a multi-tenant Python backend built on PostgreSQL, SQLAlchemy 2.0, and Alembic.

## How to Help

Based on the request in $ARGUMENTS:

**If reviewing code** (file paths or a diff): read each file in full and check it against the checklist below. Report findings in the output format at the end.

**If guiding implementation** (a question or task): give specific guidance grounded in the patterns below, after reading the relevant source files so the advice fits this codebase.

The checklist states opinionated defaults. Where the project's `AGENTS.md`/`CLAUDE.md` or its established code sets a different convention, the project wins — note the difference as an Observation rather than a finding.

## Checklist

### Tenant Scoping (Critical)
- [ ] Queries against tenant-owned tables are constrained to the caller's tenant (and owner, where the app has per-user visibility) — including lookups by id, bulk `IN (...)` lookups, updates, and deletes
- [ ] The scoping is applied the same way everywhere (a shared helper, base repository, scoped session, or row-level security) rather than re-typed differently per query
- [ ] Unscoped queries exist only for system work (exports, retention jobs, migrations) and are conspicuous at the call site

### SQLAlchemy 2.0 Execution Style
- [ ] `session.execute(select(...))` — never the legacy `session.query(...)`
- [ ] `.scalar_one_or_none()` for single optional results, `.scalars().all()` for many, `.scalar_one()` for a single required value

### Single Round-Trip Operations
- [ ] **Check-then-set is a race.** Any code that reads a row, checks a field, then writes based on it — including the plain ORM form (`obj = session.get(...)`; `if obj.status != X: raise`; `obj.status = Y`; commit) — lets two concurrent requests both pass the check (double refunds, double redemptions, oversold stock). Fold the precondition into one statement: `update(...).where(id == ..., status == X).values(status=Y).returning(...)`, treating "no row returned" as the precondition failing (or lock the row with `with_for_update()` when the check can't be expressed in SQL)
- [ ] `insert().returning()` when the inserted row is needed; `delete().returning()` when confirmation is needed

### Transactions (Unit of Work)
- [ ] The orchestrator's entry point owns the transaction, through a reentrant unit of work: the outermost entry point commits or rolls back once; an entry point called from another orchestrator joins the caller's transaction. Services and repositories add, flush, and query — they never commit
- [ ] Routers and session dependencies never commit (they provide and close the session). Besides keeping HTTP out of transaction management, a request-scoped `yield` dependency's exit code runs *after* the response by default, so a commit there can fail after the client already saw success
- [ ] Failures roll back the whole unit — no partial commits mid-operation, and nested units never commit early
- [ ] Driver errors that signal a domain condition (`IntegrityError` on a unique key) surface at `flush()` inside the repository and are translated to domain exceptions there
- [ ] Savepoints (`begin_nested()`) only where a sub-step's failure is genuinely recoverable within the operation
- [ ] Background jobs open their own session per job and their own transaction per unit of work; sessions are never shared across jobs, threads, or concurrent tasks

### Repository Patterns
- [ ] No repository query methods called in loops — use a bulk method (add one if missing)
- [ ] Query methods return `None` for not-found; mutation methods raise a domain exception on not-found
- [ ] One flexible `list_*` with optional keyword filters rather than many `list_active_*`, `list_expired_*` variants

### Data Modeling
- [ ] Primary entities carry `created_at`, `updated_at`, and (if the app soft-deletes) a nullable `deleted_at`; derived/analytical tables (tallies, aggregates, caches) may hard delete since they can be recomputed
- [ ] Record → domain model conversion happens in mappers; ORM records don't escape the data layer, in return types *or* parameters

### Timezone Handling
- [ ] Timestamps stored UTC; converted with `ZoneInfo(tz)` before any user-facing formatting (API strings, emails, LLM prompts, reports)
- [ ] Date-based aggregation groups by *local* date, not UTC date; local date ranges are converted to UTC bounds before querying
- [ ] Cache keys include `tz` when timezone affects the cached output

### Performance
- [ ] No N+1 patterns — `selectinload` when accessing a relationship in a loop
- [ ] No over-fetching on large result sets (select only needed columns)
- [ ] Counts use `select(func.count())` in a dedicated `count_*` method that mirrors every WHERE clause of its `list_*` twin — never `len(list_*(...))`
- [ ] Pagination for potentially large result sets; new query shapes have supporting indexes

### Migrations
- [ ] Every change to records (table, column, type, constraint, index) has a migration, and the migration matches the record definition 1:1 — column types, nullability, defaults, FKs, indexes, constraints
- [ ] Both `upgrade()` and `downgrade()` are implemented and safe
- [ ] Migrations are linear and deterministic, relying on transactional DDL and the version table; model/migration drift is caught by `alembic check` in CI, and new databases are built with `create_all()` + `alembic stamp head` rather than replaying history
- [ ] Idempotency guards where re-running is a real possibility: data backfills (always), operations outside a transaction (`CREATE INDEX CONCURRENTLY`, enum `ADD VALUE` in an autocommit block), databases without transactional DDL, and teams that backport migrations — using Alembic's native `if_not_exists`/`if_exists` rather than hand-written inspector checks
- [ ] No blanket name-only existence guards elsewhere — they let a migration "succeed" against an object that exists with the wrong type or nullability, hiding drift
- [ ] Object names follow the metadata naming convention via `op.f()`
- [ ] Added columns are nullable or carry a server default (NOT NULL without default breaks existing rows); otherwise add → backfill → tighten
- [ ] New tables include `created_at` (indexed), `updated_at` (indexed), `deleted_at` (nullable), all `timezone=True`
- [ ] Unique constraints on soft-deleting tables are partial unique indexes with `postgresql_where=sa.text("deleted_at IS NULL")`
- [ ] Postgres-only operations (enum changes, GIN indexes, JSONB, type conversions) are guarded by `conn.dialect.name`; enum additions use `ADD VALUE IF NOT EXISTS`
- [ ] JSON columns use `sa.JSON().with_variant(postgresql.JSONB(...), "postgresql")`
- [ ] Complex backfills live in a companion idempotent `.sql` file next to the migration
- [ ] Downgrades that swap constraints/indexes create the replacement before dropping the original

## Key Principles

1. **Tenant scoping is not optional and not ad hoc** — every query on tenant-owned data is constrained, the same way everywhere.
2. **Repositories own data access** — all SQL lives in repositories. ORM types stay in the data layer; domain models cross layer boundaries.
3. **Orchestrator entry points own the transaction** — a business operation commits or rolls back as one reentrant unit of work; repositories, services, and the HTTP layer never commit.
4. **Atomicity via `update().returning()`** — single round-trip mutations with preconditions in the `WHERE` clause are both faster and race-free.

## Where to Look

- The records module (base classes, naming convention, table definitions)
- The session factory and the request/job session dependencies (where transactions begin and end)
- How tenant scoping is applied (helper, base repository, scoped session, or row-level security)
- The mappers module (record → domain model)
- The repositories package, and a well-established repository as the canonical example
- The migrations `versions/` directory (find an exemplary table creation, column swap, and backfill)

## Reference

Detailed patterns, examples, and pitfalls: [reference.md](reference.md)

## Output Format (Review Mode)

For each finding:

```
### [SEVERITY] Description
- **Location**: `file_path:line_number`
- **Principle**: Which checklist item or principle applies
- **Issue**: What's wrong and why it matters
- **Suggestion**: Concrete fix, with a code example if helpful
```

Severity levels:
- 🔴 **Critical**: unscoped queries on tenant data, data loss risks, partial commits, commits after the response, correctness bugs, unsafe migrations
- 🟠 **Important**: pattern violations that will spread, repositories committing, N+1 queries, check-then-set races on non-financial state (on money, stock, or anything that must happen once: 🔴 Critical)
- 🟡 **Suggestion**: consistency or clarity improvements
- 💡 **Observation**: questions, ideas for future improvement

# python-backend

Code-review skills for AI coding agents working on Python web backends built with
FastAPI, SQLAlchemy, and Pydantic. Also useful for the scripts and tooling around a backend:
a review only sweeps the domains a change actually touches.

This refund endpoint looks careful. It even checks that the order belongs to the caller's
store. It has ten problems:

```python
@router.post("/orders/{order_id}/refunds")
async def refund_items(
    order_id: str,
    body: RefundRequest,
    user: CurrentUser = Depends(get_current_user),                    # ❶
    session: Session = Depends(get_session),
):
    order = session.get(Order, order_id)                              # ❷
    if order is None or order.store_id != user.store_id:
        raise HTTPException(404, "Order not found")

    items = session.scalars(
        select(LineItem)
        .where(LineItem.store_id == user.store_id)                    # ❸
        .where(LineItem.id.in_(body.line_item_ids))
    ).all()
    amount = sum(i.price_cents * i.quantity for i in items)

    if order.status != "paid":                                        # ❹
        raise HTTPException(409, "Order is not refundable")
    order.status = "refunded"                                         # ❹
    session.commit()                                                  # ❺

    try:
        async with httpx.AsyncClient() as client:                     # ❻
            resp = await client.post(
                f"{GATEWAY_URL}/refunds", json={"charge": order.charge_id, "amount": amount}
            )
            resp.raise_for_status()
    except Exception as e:
        raise HTTPException(502, detail=str(e))                       # ❼

    log.info(f"Refunded {amount} cents to {order.customer_email}")    # ❽
    metrics.increment("refunds_issued", tags={"order_id": order.id})  # ❾
    return {                                                          # ❿
        "order_id": order.id,
        "refunded_cents": amount,
        "customer_email": order.customer_email,
    }
```

Run `/python-backend:review-all` on the change and the report flags all ten, each with a fix and
a severity, and ends with a merge verdict:

| | Problem | What goes wrong | Caught by |
|---|---|---|---|
| ❶ | No permission check | Any signed-in staff member can issue refunds; the endpoint only checks that someone is logged in | `security` |
| ❷ | Blocking DB calls in an async route | Every synchronous query freezes every other request on the worker until it returns | `performance` |
| ❸ | Unchecked line-item ids | A refund can include items from another customer's order in the same store | `security` |
| ❹ | Check-then-set race | Two concurrent requests both see `paid`, and the order is refunded twice | `data-access` |
| ❺ | Commit before the side effect | The gateway call fails, the order still says refunded, and every retry gets a 409 — the customer is never paid back | `data-access` |
| ❻ | Client per request | A new connection pool and TLS handshake on every refund, and no timeout on a payment call | `performance` |
| ❼ | `str(e)` sent to the client | Gateway URLs, response bodies, and connection details leak through the API | `resilience` |
| ❽ | PII in an info log | Customer emails land in log storage, in a message no one can query by field | `security`, `resilience` |
| ❾ | High-cardinality metric | One time series per order: the metrics bill grows with every refund, and the counter can't be aggregated | `resilience` |
| ❿ | Bare dict response | No contract for clients, and personal data goes out with nothing to stop it | `design` |

## Quick start

In Claude Code:

```
/plugin marketplace add socrait/agent-skills
/plugin install python-backend@socrait-agent-skills
```

Then, after implementing a change:

```
/python-backend:review-all
```

Not using the plugin system? Copy the directories under `skills/` into your project's
`.claude/skills/` (or `~/.claude/skills/` for every project). Any agent that reads the Agent
Skills `SKILL.md` format can load them; without the plugin they're invoked without the
`python-backend:` prefix.

## How it works

`review-all` is a coordinator. It reads the diff, decides which domains the change touches,
and loads the matching specialists — data access, design, security, resilience, testing, and
performance — before it reads any code, so their checklists shape the review instead of being
recalled afterward. Then it:

1. **Scopes the change** — the working tree or the branch diff, asking which if both are dirty.
2. **Reads every changed file in full**, tests and configuration included, not just the hunks.
3. **Judges the big picture** — is this the right approach, at the right level of abstraction?
4. **Sweeps each domain**, walking that specialist's checklist against the files in its territory.
5. **Looks for missing changes** — a new record without a migration, a new endpoint without
   authorization, a feature flag whose off path has no test.
6. **Reports** every finding with a location, the principle it breaks, and a concrete fix, as
   🔴 Critical / 🟠 Important (blocking) or 🟡 Suggestion / 💡 Observation (non-blocking), and
   ends with a verdict: *Ready to merge*, *Ready after minor changes*, or *Needs rework*.

You can scope it explicitly, or call a specialist directly:

```
/python-backend:review-all the last two commits
/python-backend:security src/app/routers/refunds_router.py
```

## The skills

| Skill | Use it for |
|---|---|
| `review-all` | Reviewing a change end to end: scope the diff, judge the big picture, sweep the domain checklists, report with a verdict |
| `data-access` | Repositories, SQLAlchemy 2.0 queries, tenant scoping, unit-of-work transactions, race-free updates, timezones, migrations |
| `design` | Interfaces and contracts, type safety, domain cohesion, dependency injection, layering and orchestrators, naming, comments |
| `security` | Object-, function-, and property-level authorization (OWASP API Top 10), tenant isolation, injection, secrets, PII and LLM prompt minimization |
| `resilience` | Catch discipline, fail-fast invariants, domain exceptions, safe error responses, structured logging, metrics |
| `testing` | Test strategy, layer ownership of edge cases, fixtures (`sample_*` / `make_*`), mocking, feature flags, probabilistic tests |
| `performance` | Streaming, batching, event-loop blocking, concurrency, caching, resource lifecycle |

Each specialist also works on its own — ask it to review specific files, or for guidance while
you implement.

## Conventions

The checklists assume this architecture:

```
routers          HTTP only: validate input, call one orchestrator, shape the response
   ↓
orchestrators    one per subsystem: its only entry point, and owner of its transactions
   ↓
services         domain logic, one concept each
models           Pydantic domain models and their pure behavior, domain exceptions, messages
   ↓
data layer       repositories (all SQL), ORM records, mappers, migrations
clients          wrappers around third-party SDKs and external APIs
```

- **Calls point downward only.** A layer may call any layer below it, never one above. Services
  don't call peer services; orchestrators delegate to each other only along a declared, acyclic
  order, and anything against that order goes through a task or event.
- **Domain models are the shared vocabulary.** Every layer may import them — routers map
  them into response models, mappers in the data layer produce them — and they depend on nothing
  but each other.
  Behavior that is a pure transform of one model lives on that model, not in a service.
- **The data layer keeps its types to itself.** ORM records never leave it; mappers convert them
  to domain models, which are what cross layers.
- **Transactions belong to orchestrators.** Each orchestrator entry point runs a reentrant unit of
  work; services, the data layer, and routers never commit.
- **Dependencies are constructor-injected** and built at the top (a `dependencies.py`,
  entrypoints).
- **Multi-tenant:** tenant-owned data is always queried within the caller's tenant.

Most of the guidance (error handling, logging, testing strategy, design) applies to any Python
service. The architecture rules are the most opinionated part: where your codebase is organized
differently, write that down in your `AGENTS.md` and the skills defer to it.

## Tech stack

| Area | Assumption |
|---|---|
| Language | Python 3.12+ |
| Web | FastAPI, with authorization expressed as FastAPI dependencies |
| Models | Pydantic v2 for domain models and request/response schemas |
| Database | SQLAlchemy 2.0 (execution style), Alembic migrations, PostgreSQL in production, SQLite or Postgres in tests |
| Tests | pytest against a real database; fakes at network boundaries |
| HTTP | httpx (async) |
| Observability | structlog (event name + key-value fields), OpenTelemetry metrics |
| Cache | Redis |
| Packaging | uv |

## Make it yours

- **Your conventions win.** Every specialist treats its checklist as defaults: where your
  `AGENTS.md`/`CLAUDE.md` or established code says otherwise, it reports the difference as an
  observation instead of a finding. Writing your conventions down is the cheapest customization.
- **Fork the checklists.** They're plain markdown. Delete what doesn't apply, add your house
  rules, and keep the ❌/✅ pairs in `reference.md` — the examples do much of the teaching.
- **Always-on invariants.** Skills load when invoked; the rules you want applied while code is
  being *written* belong in your agent instructions. [templates/AGENTS-invariants.md](templates/AGENTS-invariants.md)
  is a condensed version of all six domains to paste into your `AGENTS.md` or `CLAUDE.md` and
  trim.
- **Mechanize what you can.** Several checks (an auth dependency on every protected route, no
  legacy `session.query`, no `commit()` in repositories, no sync HTTP client in async code, a
  cross-tenant 404 test for every by-id route) are good lint rules or test helpers. Review is for
  what those can't see.

## About the examples

All examples use a fictional multi-tenant marketplace: **stores** are tenants; **staff** manage
**orders** containing **line items**, **refunds**, and **shipments**; **customers** carry PII;
an LLM drafts support replies. Names like `AcmePay` are invented.

## License

[MIT](../../LICENSE)

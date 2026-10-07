# Data Access Reference

Detailed patterns, examples, and pitfalls for the data access layer. Examples use a
multi-tenant marketplace: each **store** is a tenant (`store_id`); staff manage **orders**,
which contain **line items**, **refunds**, and **shipments**.

## SQLAlchemy 2.0 Patterns

```python
from sqlalchemy import delete, func, insert, select, update
from sqlalchemy.orm import Session

# Single result (None if not found)
order = session.execute(
    select(OrderRecord).where(OrderRecord.store_id == store_id).where(OrderRecord.id == order_id)
).scalar_one_or_none()

# Multiple results
items = session.execute(
    select(LineItemRecord).where(LineItemRecord.order_id == order_id)
).scalars().all()

# Count
count = session.execute(
    select(func.count()).select_from(OrderRecord).where(OrderRecord.store_id == store_id)
).scalar_one()

# Update with returning (preferred over select-then-update)
refund = session.execute(
    update(RefundRecord)
    .where(RefundRecord.store_id == store_id)
    .where(RefundRecord.id == refund_id)
    .where(RefundRecord.status == RefundStatus.PENDING)
    .values(status=RefundStatus.APPROVED)
    .returning(RefundRecord)
).scalar_one_or_none()
```

Legacy vs modern:

```python
# ❌ Legacy Query API — never use
session.query(OrderRecord).filter(OrderRecord.id == order_id).first()

# ✅ 2.0 execution style
session.execute(select(OrderRecord).where(OrderRecord.id == order_id)).scalar_one_or_none()
```

`filter()` → `where()`; `.first()` → `.scalar_one_or_none()`; `.all()` → `.scalars().all()`.

## Tenant Scoping

Every query on tenant-owned data carries the tenant constraint — reads, bulk lookups, updates,
deletes. The failure mode is rarely "no scoping anywhere"; it's one query among fifty that forgot.

```python
# ❌ Scoped on the read path, forgotten on the write path
def get_order(self, store_id: str, order_id: str) -> OrderRecord | None:
    return self._session.execute(
        select(OrderRecord).where(OrderRecord.store_id == store_id).where(OrderRecord.id == order_id)
    ).scalar_one_or_none()

def cancel_order(self, order_id: str) -> OrderRecord | None:
    return self._session.execute(
        update(OrderRecord).where(OrderRecord.id == order_id).values(status=OrderStatus.CANCELLED).returning(OrderRecord)
    ).scalar_one_or_none()
```

Ways to make the constraint hard to forget, roughly from lightest to strongest:
- A repository helper every query starts from (`self._scoped(OrderRecord, store_id)`).
- A repository constructed per tenant (`OrderRepository(session, store_id=...)`) so methods can't omit it.
- A session-level filter (`with_loader_criteria` in a `do_orm_execute` event) applied to every SELECT.
- Postgres row-level security keyed on a per-transaction setting — enforced even for raw SQL.

Pick one and use it everywhere; review then checks the exceptions instead of every query.
System jobs that legitimately span tenants (nightly exports, retention purges) should be
conspicuous — a separately named method or repository — so reviewers can see each one.

## Repository Patterns

### Query Method (returns None for not-found)

```python
class OrderRepository:
    def __init__(self, session: Session):
        self._session = session

    def get_order(self, store_id: str, order_id: str) -> OrderRecord | None:
        return self._session.execute(
            select(OrderRecord).where(OrderRecord.store_id == store_id).where(OrderRecord.id == order_id)
        ).scalar_one_or_none()
```

### List Method with Optional Filters

```python
def list_orders(
    self,
    store_id: str,
    *,
    customer_id: str | None = None,
    statuses: list[OrderStatus] | None = None,
    placed_after: datetime | None = None,
) -> list[OrderRecord]:
    stmt = select(OrderRecord).where(OrderRecord.store_id == store_id).where(OrderRecord.deleted_at.is_(None))
    if customer_id:
        stmt = stmt.where(OrderRecord.customer_id == customer_id)
    if statuses:
        stmt = stmt.where(OrderRecord.status.in_(statuses))
    if placed_after:
        stmt = stmt.where(OrderRecord.placed_at >= placed_after)
    return list(self._session.execute(stmt).scalars().all())
```

### Bulk Query Instead of a Loop

```python
# ❌ N+1 — one query per id
products = [repo.get_product(store_id, pid) for pid in product_ids]

# ✅ One query with IN
def list_products_by_ids(self, store_id: str, product_ids: list[str]) -> list[ProductRecord]:
    return list(self._session.execute(
        select(ProductRecord).where(ProductRecord.store_id == store_id).where(ProductRecord.id.in_(product_ids))
    ).scalars().all())
```

### Check-Then-Set Races

The most common form doesn't look like a query at all — it's plain ORM attribute access:

```python
# ❌ Two concurrent requests both load status == "paid", both pass the check, both refund
order = session.get(OrderRecord, order_id)
if order.status != OrderStatus.PAID:
    raise OrderNotRefundableError(order_id)
order.status = OrderStatus.REFUNDED
session.flush()
```

The check and the write must be one statement, so the database arbitrates between concurrent
callers. The mutation method below is the fix: the precondition lives in the `WHERE` clause and
"no row returned" means it failed. When the condition can't be expressed in SQL, lock the row
first with `select(...).with_for_update()`.

### Mutation Method (no commit)

```python
def approve_refund(self, store_id: str, refund_id: str) -> RefundRecord:
    refund = self._session.execute(
        update(RefundRecord)
        .where(RefundRecord.store_id == store_id)
        .where(RefundRecord.id == refund_id)
        .where(RefundRecord.status == RefundStatus.PENDING)
        .values(status=RefundStatus.APPROVED)
        .returning(RefundRecord)
    ).scalar_one_or_none()
    if refund is None:
        raise RefundNotFoundError(refund_id)
    return refund
```

The `status == PENDING` guard in the WHERE clause makes the transition and its precondition one
atomic statement, so two concurrent approvals can't both succeed.

### Create Method (flush to surface integrity errors)

```python
def create_refund(self, *, store_id: str, order_id: str, amount_cents: int, idempotency_key: str) -> RefundRecord:
    refund = RefundRecord(store_id=store_id, order_id=order_id, amount_cents=amount_cents, idempotency_key=idempotency_key)
    self._session.add(refund)
    try:
        self._session.flush()
    except IntegrityError as e:
        raise DuplicateRefundError(order_id) from e
    return refund
```

`flush()` sends the INSERT inside the current transaction, so the unique-constraint violation
surfaces here — where it can be translated to a domain exception — rather than at a commit far
away. The failed flush leaves the transaction unusable; the unit of work's rollback cleans up.

## Transactions: Orchestrator Entry Points Own the Unit of Work

A business operation commits or rolls back as one unit, and the domain layer — not HTTP —
decides where that unit begins and ends: each orchestrator (subsystem facade) entry point.
Repositories and services participate; they never commit.

### A Reentrant Unit of Work

```python
class UnitOfWork:
    """One per session. The outermost transaction() commits or rolls back; nested ones join it."""

    def __init__(self, session: Session):
        self._session = session
        self._depth = 0

    @contextmanager
    def transaction(self) -> Iterator[None]:
        if self._depth:                      # called from inside another unit of work: join it
            self._depth += 1
            try:
                yield
            finally:
                self._depth -= 1
            return
        self._depth = 1
        try:
            yield
            self._session.commit()
        except BaseException:
            self._session.rollback()
            raise
        finally:
            self._depth = 0
```

Every orchestrator built for the same request (or job) shares one `UnitOfWork` over one session —
construct both once in the dependency graph. It never calls `session.begin()`: a loader
dependency's earlier query has usually auto-begun a transaction already, and a second `begin()`
raises. It commits what's there.

```python
class RefundOrchestrator:
    def issue_refund(self, user: CurrentUser, order_id: str, request: CreateRefundRequest) -> Refund:
        with self._uow.transaction():
            order = self._orders.get_order(user, order_id)              # OrderOrchestrator — joins this unit
            refund = self._refunds.create_refund(store_id=user.store_id, order_id=order.id, ...)
            self._orders.add_refunded_amount(user, order.id, refund.amount_cents)   # joins too
        return map_refund_to_model(refund)

class OrderOrchestrator:
    def add_refunded_amount(self, user: CurrentUser, order_id: str, amount_cents: int) -> None:
        with self._uow.transaction():                                   # commits when called alone
            self._orders.increment_refunded(user.store_id, order_id, amount_cents)
```

`add_refunded_amount` commits when a router calls it directly, and joins the refund's transaction
when `RefundOrchestrator` calls it — neither side needs to know who called whom.

```python
# ❌ Each repository method commits — a failure in the second step leaves the first committed
refund = refunds.create_refund(...)       # commits
orders.add_refunded_amount(...)           # raises → refund exists, order total wrong

# ❌ A nested unit that commits on exit — the same bug, hidden inside a helper
@contextmanager
def transaction(self):
    yield
    self._session.commit()                # an inner block commits the outer block's work early
```

The second ❌ is easy to write by accident and hard to spot: a "nested" context that still
commits on exit looks atomic in every single-level test. Cover it with a test that fails an outer
step *after* an inner block exits and asserts the inner write is gone.

### Keep HTTP Out of It

Session dependencies provide and close the session, and nothing more:

```python
def get_session() -> Iterator[Session]:
    with SessionFactory() as session:
        yield session
```

```python
# ❌ The router layer decides atomicity — and commits after the response went out
def get_session() -> Iterator[Session]:
    with SessionFactory() as session, session.begin():
        yield session
```

Committing in a dependency makes every entry point (router, task handler, CLI) responsible for a
business guarantee. And FastAPI runs a request-scoped `yield` dependency's exit code **after**
the response is sent, so a commit there can fail after the client already received a 200.

### External Calls and Transactions

Don't hold a database transaction open across slow network calls — locks and connections stay
held for the whole round-trip. Read and validate, commit, then call out; or record the intent
in the transaction (an outbox row) and let a worker make the call.

### Background Jobs

```python
def export_store_orders(store_id: str) -> None:
    with SessionFactory() as session:
        make_export_orchestrator(session).export(store_id)   # its entry points own the units of work
```

One session (and one `UnitOfWork`) per job; one transaction per unit of work inside it (per
store, per batch). Never share a session across threads, concurrent tasks, or jobs.

## Eager Loading

```python
# ❌ Lazy load per order — N+1
orders = session.execute(select(OrderRecord)).scalars().all()
for o in orders:
    total = sum(li.price_cents for li in o.line_items)

# ✅ Eager load up front
orders = session.execute(select(OrderRecord).options(selectinload(OrderRecord.line_items))).scalars().all()
```

## Counting

When a `count_*` method and its `list_*` twin must agree, mirror every WHERE clause —
especially `deleted_at IS NULL`. Any divergence and the two silently disagree.

## Mappers

```python
def map_order_to_model(record: OrderRecord) -> Order:
    return Order(
        id=record.id,
        store_id=record.store_id,
        status=OrderStatus(record.status),
        total=Money(amount=record.total_cents, currency=record.currency),
        placed_at=record.placed_at,
    )
```

Map before data leaves the data layer. The same applies to parameters: a service that accepts
`list[ProductRecord]` leaks the ORM type just as much as one that returns it.

## Migrations

### Deterministic by Default, Idempotent Where Re-runs Are Real

On Postgres, DDL is transactional: a migration that fails rolls back entirely, and the version
table records exactly which revisions ran. That makes plain, linear migrations the default:

```python
def upgrade() -> None:
    op.create_table("refund", ...)
    op.add_column("order", sa.Column("refunded_total_cents", sa.Integer(), nullable=False, server_default="0"))
```

Keep the history honest instead of making every step defensive:
- **New databases**: `metadata.create_all()` then `alembic stamp head` — never replay history.
- **Drift**: `alembic check` (Alembic 1.9+) in CI fails when models and migrations disagree.

Guard with existence checks only where a re-run is a real possibility:

```python
# Data backfills — always written to be re-runnable
op.execute("UPDATE \"order\" SET refunded_total_cents = 0 WHERE refunded_total_cents IS NULL")

# Operations outside a transaction: a failure here leaves partial state behind
def upgrade() -> None:
    with op.get_context().autocommit_block():
        op.create_index(
            op.f("ix_order_placed_at"), "order", ["placed_at"],
            postgresql_concurrently=True, if_not_exists=True,
        )
```

The same applies when the database lacks transactional DDL (MySQL), or when a team backports
migrations across branches and can't fully trust the version table. Prefer Alembic's native
`if_not_exists` / `if_exists` (on `create_index`/`drop_index` from 1.12, `create_table`/
`drop_table` from 1.13.3, `add_column`/`drop_column`/`drop_constraint` from 1.16) over
hand-written inspector checks.

```python
# ❌ Blanket name-only guards: if "refunded_total_cents" already exists as a String, this
#    "succeeds" and the schema silently disagrees with the model forever after
columns = {c["name"] for c in sa.inspect(op.get_bind()).get_columns("order")}
if "refunded_total_cents" not in columns:
    op.add_column("order", sa.Column("refunded_total_cents", sa.Integer(), nullable=False, server_default="0"))
```

### Table Creation

```python
op.create_table(
    "refund",
    sa.Column("id", sa.String(), nullable=False),
    sa.Column("store_id", sa.String(), nullable=False),
    sa.Column("order_id", sa.String(), nullable=False),
    sa.Column("amount_cents", sa.Integer(), nullable=False),
    sa.Column("created_at", sa.DateTime(timezone=True), nullable=False, index=True),
    sa.Column("updated_at", sa.DateTime(timezone=True), nullable=False, index=True),
    sa.Column("deleted_at", sa.DateTime(timezone=True), nullable=True),
    sa.ForeignKeyConstraint(["store_id"], ["store.id"], name=op.f("fk_refund_store_id_store")),
    sa.ForeignKeyConstraint(["order_id"], ["order.id"], name=op.f("fk_refund_order_id_order")),
    sa.PrimaryKeyConstraint("id", name=op.f("pk_refund")),
)
```

### Partial Unique Indexes (Soft Deletes)

```python
# ✅ A soft-deleted row doesn't block a new live one
op.create_index(
    op.f("uq_refund_order_id_idempotency_key"),
    "refund",
    ["order_id", "idempotency_key"],
    unique=True,
    postgresql_where=sa.text("deleted_at IS NULL"),
)

# ❌ Plain unique constraint — a deleted refund blocks re-creation forever
op.create_unique_constraint("uq_refund", "refund", ["order_id", "idempotency_key"])
```

### JSON and Enums

```python
sa.Column("metadata", sa.JSON().with_variant(postgresql.JSONB(astext_type=sa.Text()), "postgresql"))

def upgrade() -> None:
    if op.get_bind().dialect.name != "postgresql":
        return
    op.execute("ALTER TYPE refundstatus ADD VALUE IF NOT EXISTS 'PARTIALLY_REFUNDED'")
```

Removing an enum value on downgrade means recreating the type (create `_prev`, cast the column
with a `CASE` mapping removed values, drop the old type, rename).

### Data Backfills

Keep complex backfills in a sibling `.sql` file read by the migration; strip `BEGIN;`/`COMMIT;`
(Alembic owns the transaction); write it idempotently (`IF NOT EXISTS`, `ON CONFLICT DO NOTHING`,
`WHERE new_col IS NULL`).

### Safe Downgrade Ordering

When swapping an index or constraint, the downgrade recreates the old one *before* dropping the
new one, so the data is never unprotected in between.

## Timezones: Store UTC, Display Local

```python
from zoneinfo import ZoneInfo

# ✅ Convert before formatting anything a human reads
local = order.placed_at.astimezone(ZoneInfo(store.timezone))
formatted = local.strftime("%b %d, %Y %H:%M")

# ❌ Formatting UTC directly — "placed yesterday" for a customer who ordered tonight
formatted = order.placed_at.strftime("%b %d, %Y %H:%M")
```

Where it matters: date-range queries (convert local day bounds to UTC before querying),
per-day aggregation (group by local date), customer emails and receipts, LLM prompts, report
strings, and cache keys whose output depends on `tz`.

Where it doesn't: repository internals (always UTC), audit columns, machine-to-machine APIs
(ISO 8601 with `Z`).

## Common Pitfalls

1. **One unscoped query among many.** Scoping drift is per query; review the write paths and bulk lookups, not only the obvious reads.
2. **Repositories that commit.** Multi-step operations become partially applied on failure.
3. **Committing after the response.** A `yield` dependency's exit code runs after the response by default.
4. **Check-then-set.** Load, check a field, assign, commit — another transaction can change the row in between, and both pass the check. Use `update().where(<precondition>).returning()`.
5. **Leaking ORM records** above the data layer, in either direction.
6. **Repository calls in loops.** Add a bulk method.
7. **Dialect-specific statements** (`insert().on_conflict_do_update()`) imported unconditionally when tests run on SQLite — choose the dialect's `insert` from configuration.

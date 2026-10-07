# Design Reference

Detailed patterns and examples for code design. Examples use a multi-tenant marketplace:
**stores** (tenants) sell **products**; staff manage **orders** with **line items**,
**refunds**, and **shipments**; **customers** place orders. The assumed layout:

```
app/
├── api.py, worker.py        # entrypoints (dependency roots)
├── dependencies.py          # make_* factories that build object graphs
├── routers/                 # HTTP only
├── services/                # domain layer: focused services + orchestrators
├── clients/                 # third-party SDK wrappers
├── data/                    # records, repositories, mappers, migrations
├── core/                    # domain models, messages, exceptions (pure)
├── libs/                    # app-agnostic mechanisms the app binds
└── utils/                   # layer-agnostic helpers only
```

## Loose Coupling

```python
# ❌ Depends on a concrete implementation
class InvoiceService:
    def __init__(self, s3: S3Storage):
        self._s3 = s3

# ✅ Depends on the role
class InvoiceService:
    def __init__(self, storage: DocumentStorage):
        self._storage = storage
```

Watch for: hidden temporal coupling, shared mutable state, chatty interfaces between modules,
god objects.

## Separation of Concerns and Single Responsibility

Divide code by business domain, feature, and use case — not by technical concern. A unit of
code has one clearly-defined purpose with obvious inputs, outputs, and side effects.

```python
# ❌ One method fuses pricing, persistence, notification, and analytics
class CheckoutHandler:
    def complete(self, cart):
        total = self._price(cart)
        order = self._save(cart, total)
        self._email_receipt(order)
        self._track(order)

# ✅ Orchestrator delegates to single-purpose collaborators
class CheckoutOrchestrator:
    def complete(self, user: CurrentUser, cart: Cart) -> Order:
        quote = self._pricing.quote(cart)
        order = self._orders.place(user, cart, quote)
        self._receipts.send(order)
        return order
```

## Encapsulation

Hide internal state; expose intention-revealing operations.

```python
# ❌ Caller inspects internal state
if not self._gateway._session_token:
    raise GatewayNotReadyError()

# ✅ Component enforces its own invariants
self._gateway.capture(payment_intent_id)  # raises internally if not ready
```

### Don't Leak Implementation Concerns

If a service owns a cache, the cache key is its business — callers pass logical parameters.

```python
# ❌ Caller builds the cache key and now participates in caching
async def draft_reply(self, *, cache_key: str, tone: str) -> str: ...

# ✅ Service derives its own key from the logical inputs
async def draft_reply(self, *, ticket_id: str, tone: ReplyTone) -> str:
    cache_key = self._cache_key(ticket_id, tone)
    ...
```

### Underscore Prefix: Symbols, Never Files

`_` marks a *symbol* as outside the public contract. Never prefix module or package files —
`pricing/_shared.py` misleads; name it `pricing/pricing_common.py` and let package exports
express scope.

Apply the prefix in both directions:

```python
# ✅ Internal helpers are _-prefixed — including closures in registration functions
def register_export_task(queue):
    @queue.task()
    def _export_orders_task(store_id: str) -> None:
        ...

# ❌ A _-prefixed type in a public signature contradicts itself
class _ReceiptLine(Protocol): ...
class ReceiptService:
    def render(self, *, lines: list[_ReceiptLine]) -> str: ...

# ✅ Public contract types are un-prefixed
class ReceiptLine(Protocol): ...
```

## Dependency Injection

High-level modules never instantiate their own seams. Construct at the top; inject through
constructors.

```python
# ❌ Service constructs its own dependencies
class RefundService:
    def __init__(self, session: Session):
        self._refund_repo = RefundRepository(session)
        self._gateway = AcmePayClient(settings.payment_api_key)

# ✅ Dependencies injected; built in dependencies.py
class RefundService:
    def __init__(self, refund_repo: RefundRepository, gateway: PaymentGatewayClient):
        self._refund_repo = refund_repo
        self._gateway = gateway

def make_refund_service(session: Session) -> RefundService:
    return RefundService(refund_repo=RefundRepository(session), gateway=make_payment_gateway_client())
```

Never inject through config objects; never instantiate a seam in a method body.

### Seams, Impl Collaborators, Factories, Values

Every constructor parameter is one of four kinds:

| Kind | Examples | How it's taken |
|---|---|---|
| **Seam** | repository, I/O client, transaction manager, task scheduler, feature-flag client, LLM client | required parameter |
| **Impl collaborator** | pure calculators, parsers, internal helpers built with no arguments (or only from collaborators the owner already holds), replaced only by test doubles | optional keyword-only, defaulting to the production construction |
| **Call-time collaborator** | needs data only known per call | a factory, defaulting to the class itself |
| **Value** | models, settings dataclasses, per-request builders | not injected — built where its data is |

```python
class CheckoutOrchestrator:
    def __init__(
        self,
        order_repo: OrderRepository,                     # seam
        gateway: PaymentGatewayClient,                   # seam
        *,
        tax_calculator: TaxCalculator | None = None,     # impl collaborator
        receipt_renderer_factory: Callable[[Store], ReceiptRenderer] = ReceiptRenderer,  # call-time
    ):
        self._order_repo = order_repo
        self._gateway = gateway
        self._tax = tax_calculator or TaxCalculator()
        self._receipt_renderer_factory = receipt_renderer_factory

    def complete(self, user: CurrentUser, cart: Cart, store: Store) -> Order:
        renderer = self._receipt_renderer_factory(store)
        ...
```

The smell this prevents: a class whose signature lists every internal helper as a required
parameter, so every factory repeats the same no-argument constructions and the signature reads
as choices where there are none.

## Law of Demeter

```python
# ❌ Reaching through collaborators
country = order.customer.addresses[0].country

# ✅ Ask the immediate collaborator
country = order.shipping_country()
```

## No Unexpected Side Effects

Functions compute and return; procedures act. Watch for read paths that write, getters that
cache as a side effect, and query-sounding names that mutate.

### Constructors Only Store

```python
# ❌ Network call in __init__
class InventoryCache:
    def __init__(self, redis: Redis):
        self._redis = redis
        self._redis.ping()

# ✅ Store only; I/O happens when methods run
class InventoryCache:
    def __init__(self, redis: Redis):
        self._redis = redis
```

Not every method call is I/O — e.g. `redis.register_script()` only hashes the source locally.
Check what a call actually does before adding lazy-init complexity.

## Contract Design

### Exceptions vs Result Codes

**Use domain exceptions by default.** They propagate, carry context, integrate with metrics
via `error_code`, and are handled at boundaries (routers, worker entrypoints).

**Use result codes** (enums, small dataclasses) when:
- **The caller decides policy** — a fraud check returns `FraudVerdict` so the orchestrator can
  choose fail-open vs fail-closed per feature flag.
- **Every outcome maps to the same external response** — `request_password_reset()` returns
  `ResetResult` because the router always answers 200 (anti-enumeration).
- **Per-item failures must not abort a batch** — `bulk_import_products()` returns per-row
  outcomes.

The caller then *translates* a result into an exception when *its* caller needs to
distinguish it (the orchestrator turns `FraudVerdict.BLOCK` into `PaymentBlockedError` so the
router can return a distinct status).

**Never return `None` from multiple distinct exit paths.**

```python
# ❌ Caller can't tell which path ran
def request_password_reset(self, email: str) -> None:
    user = self._users.get_by_email(email)
    if not user:
        return
    if user.is_locked:
        return
    self._send_email(user)

# ✅ Every path is distinct and testable
def request_password_reset(self, email: str) -> ResetResult:
    user = self._users.get_by_email(email)
    if not user:
        return ResetResult.UNKNOWN_EMAIL
    if user.is_locked:
        return ResetResult.ACCOUNT_LOCKED
    return self._send_email(user)  # SENT or SEND_FAILED
```

### Flexible vs Rigid Interfaces

```python
# ❌ Rigid — will proliferate
class CouponRepository:
    def list_active_coupons(self, store_id): ...
    def list_expired_coupons(self, store_id): ...
    def list_archived_coupons(self, store_id): ...

# ✅ Flexible — one method, optional filters
class CouponRepository:
    def list_coupons(self, store_id: str, *, include_expired: bool = False, include_archived: bool = False) -> list[CouponRecord]: ...
```

Balance against single responsibility: `search_coupons(code_prefix)` with its own semantics is
its own method.

### Message Design

```python
class OrderPlacedV1(DomainModel):
    id: str
    type: MessageType
    version: MessageVersion
    occurred_at: datetime
    store_id: str
    order_id: str
    total_cents: int
```

Version, contextual ids, timestamp, room to extend.

### Private by Default

Internal unless deliberately promoted. Watch for internal types or endpoints exposed without
intent, and data structures leaking across layers.

## Type Safety

### Typed Models over Dicts

```python
# ❌ The shape is an implicit contract
class ShippingClient(ABC):
    @abstractmethod
    def quote(self, parcels: list[dict]) -> list[dict]: ...

# ✅ The contract is self-documenting and checkable
class ShippingClient(ABC):
    @abstractmethod
    def quote(self, parcels: list[Parcel]) -> list[ShippingQuote]: ...
```

Consumers of a dict inherit the problem — `q.get("carrier")` has no autocomplete and fails
silently on a typo.

## Domain Cohesion

### Domain Functions Belong with Their Domain

```python
# ❌ Domain logic in a general utils module
# utils/math_utils.py
def refundable_amount(order_total: int, refunded: int, shipping: int) -> int: ...

# ❌ Classless bare-function module in services/ — not a service, just a homeless function
# services/refund_math.py
def refundable_amount(order: Order) -> int: ...

# ✅ Behavior lives on the model that owns the concept
# core/models.py
class Order(DomainModel):
    def refundable_cents(self) -> int:
        return self.total_cents - self.refunded_cents
```

Heuristic: if removing the function from `utils/` makes `utils/` *more* cohesive, it didn't
belong there. If a `services/` module would hold no class, no state, and no injected
dependency, it isn't a service.

### Dissolve, Don't Duplicate

```python
# ❌ A new type mirroring an existing model, plus an identity mapper
class RefundSummary(DomainModel):
    id: str
    order_id: str
    amount_cents: int
    status: RefundStatus

def to_refund_summary(refund: Refund) -> RefundSummary:
    return RefundSummary(id=refund.id, order_id=refund.order_id, amount_cents=refund.amount_cents, status=refund.status)

# ✅ Use Refund. When Refund gains `reason`, nothing silently drops it.
```

A mapper is justified only when it transforms (record → model with type conversion, a
deliberately narrower public projection with a stated reason).

### Validate at the Choke Point

```python
# ❌ Every caller remembers (or forgets) to check
class SupportReplies:
    async def draft(self, ...):
        text = await self._llm.generate(...)
        assert_no_card_numbers(text)

class ReviewSummaries:
    async def summarize(self, ...):
        text = await self._llm.generate(...)
        # forgot

# ✅ One place covers every caller
class LlmPrompter:
    async def _generate(self, ..., *, customer_facing: bool) -> str:
        text = await self._client.generate(...)
        if customer_facing:
            assert_no_card_numbers(text)
        return text
```

## Method Organization

```python
# ✅ Ordered by call hierarchy, high to low
class RefundService:
    def issue(self, ...):
        self._validate(...)
        amount = self._compute(...)
        return self._persist(...)

    def _validate(self, ...): ...
    def _compute(self, ...): ...
    def _persist(self, ...): ...
```

Don't group by visibility. Honor any existing section structure in the file.

### No Public-Calls-Public

```python
# ❌ One entry point calls another
class RefundOrchestrator:
    def issue_full(self, user: CurrentUser, order_id: str) -> Refund:
        log.info("full_refund_requested", order_id=order_id)
        order = self._orders.get_order(user.store_id, order_id)
        return self.issue_partial(user, order_id, order.refundable_cents())

    def issue_partial(self, user: CurrentUser, order_id: str, amount_cents: int) -> Refund:
        log.info("partial_refund_requested", order_id=order_id, amount_cents=amount_cents)
        if not self._policy.within_partial_refund_window(order_id):
            raise RefundWindowClosedError(order_id)
        with self._uow.transaction():
            refund = self._refunds.create_refund(...)
        metrics.increment_counter(MetricName.REFUNDS_ISSUED, attributes={"kind": "partial"})
        return refund

# ✅ Both entry points share a private step
class RefundOrchestrator:
    def issue_full(self, user: CurrentUser, order_id: str) -> Refund:
        order = self._orders.get_order(user.store_id, order_id)
        return self._issue(user, order, order.refundable_cents(), kind=RefundKind.FULL)

    def issue_partial(self, user: CurrentUser, order_id: str, amount_cents: int) -> Refund:
        if not self._policy.within_partial_refund_window(order_id):
            raise RefundWindowClosedError(order_id)
        order = self._orders.get_order(user.store_id, order_id)
        return self._issue(user, order, amount_cents, kind=RefundKind.PARTIAL)

    def _issue(self, user: CurrentUser, order: Order, amount_cents: int, *, kind: RefundKind) -> Refund: ...
```

This is a strong opinion, and it earns its keep:

- **Entry points carry entry concerns.** A public method is where a call enters the class, so it
  is where validation, logging, metrics, and the unit of work belong. In the ❌ version a full
  refund logs twice, is counted as a *partial* refund, and is rejected by the *partial*-refund
  window — a business rule that was never meant to apply to it, inherited silently because the
  entry point's checks came along with its body.
- **Contracts stay independent.** When `issue_full` calls `issue_partial`, every change to
  `issue_partial`'s contract (a new precondition, a new side effect, a renamed metric) silently
  changes `issue_full` too. Nobody reviewing a change to one sees that it altered the other.
- **The class's internal call graph stays flat and legible.** Entry points sit on top, private
  steps below, and calls only go downward — the same shape the layering rules give the whole
  application. Readers can tell at a glance which methods are contracts and which are mechanism.

The cost is one private method. When two entry points share most of their body, that private
method is the real operation and the public ones are thin, honest adapters around it.

## Naming

- No redundant suffixes: `get_status`, not `get_status_endpoint`.
- Consistent vocabulary: don't mix "customer"/"buyer" or "order"/"purchase" for the same thing.
- ABCs named for the role: `PaymentGatewayClient`, not `AcmePayClientABC`.
- Full words in function names (`make_transaction_manager`); conventional abbreviations fine in
  variables (`tx_manager`, `db`, `ff_client`). A reader of a call site has context; a reader of
  a public interface doesn't.
- Title-cased acronyms: `HttpUrl`, `CacheTtl`, `OrderId` — `HTTPURLTTL` doesn't parse.

## Imports

```python
from datetime import datetime

import httpx
from fastapi import APIRouter, Depends

from app.core.models import Order
from app.services import RefundService

from .refund_common import RefundPolicy
```

- Package-level imports (`from app.services import RefundService`), not internal reach-ins.
- A package `__init__.py` re-exports with **relative** imports, and only from its **direct
  children** — never past a sub-package's own `__init__.py`.

## Package Management

- `uv` only — no `pip`, `uv pip`, or `requirements.txt`.
- Runtime deps pinned with `~=` (`uv add 'httpx~=0.28.1'`); dev tools may use `>=`.
- Don't add a package when an existing one does the job (`httpx`, not also `requests`).

## Comments and Docstrings

```python
# ❌ Restates the code
total = sum(li.price_cents for li in items)  # sum line item prices

# ✅ Explains a non-obvious why
total = sum(li.price_cents for li in items)  # integer cents: float totals drift on large carts

# ❌ Cites a spec id — meaningless without the spec open, rots on renumbering
# Refunds can't exceed captured amount (D4).

# ✅ States the constraint
# Refunds can't exceed the captured amount.
```

Docstrings on public methods when they raise domain exceptions, have non-obvious return
semantics, or have important preconditions — and not otherwise:

```python
async def issue_refund(self, user: CurrentUser, order_id: str, amount_cents: int) -> Refund:
    """Issue a refund against the order's captured payment.

    Raises:
        OrderNotFoundError: The order doesn't exist in the user's store.
        RefundExceedsCaptureError: amount_cents exceeds what remains refundable.
    """
```

## Architecture Layers

```
Entrypoints (api.py, worker.py)
    │  build graphs via dependencies.py
Routers        — HTTP only: validate input, call one orchestrator, shape the response
    │
Orchestrators  — one per subsystem: its facade, its entry points, its transactions
    │
Services       — domain logic, one concept each
Models         — Pydantic domain models and their pure behavior, domain exceptions, messages
    │
Data layer     — repositories (all SQL), ORM records, mappers, migrations
Clients        — wrappers around third-party SDKs and external APIs
```

Calls point downward only: a layer may call any layer below it, never one above. Domain models
(`core/`) are the exception by design — the shared vocabulary every layer imports, including the
data layer's mappers — so they depend on nothing but each other.

### Subsystems and Their Orchestrators

Organize the domain layer into **subsystems** — by resource (orders, catalog, customers) or by
operation type (checkout, fulfillment, reporting). Each subsystem has one **orchestrator**: its
facade and only public entry point. ("Orchestrator" is shorthand for *subsystem facade*; name it
for the subsystem, e.g. `OrderOrchestrator`, `FulfillmentOrchestrator`.)

**Inside a subsystem**, services stay focused — one concept each — and don't call their peers.
The orchestrator sequences them.

```python
# ❌ Services calling services: pricing → inventory → tax, each knowing about the next.
#    Changing checkout means tracing a path through several classes, and a cycle is one import away.
class PricingService:
    def quote(self, cart: Cart) -> Quote:
        self._inventory.reserve(cart)            # pricing now depends on inventory
        tax = self._tax.compute(cart)            # ...and on tax
        ...

# ✅ The orchestrator owns the sequence; each service knows only its own concept
class CheckoutOrchestrator:
    def complete(self, user: CurrentUser, cart: Cart) -> Order:
        with self._uow.transaction():
            quote = self._pricing.quote(cart)
            self._inventory.reserve(user.store_id, cart)
            tax = self._tax.compute(cart, quote)
            order = self._orders.place(user, cart, quote, tax)   # another subsystem's orchestrator
        self._receipts.send(order)                               # after commit: no network call inside the transaction
        return order
```

**Between subsystems**, orchestrators delegate to each other — that's how a checkout places an
order or a report reads fulfillment data — under three rules:

1. **Through the facade only.** Another subsystem's services, repositories, and internal classes
   are off limits; if you need its data, its orchestrator exposes it. Two subsystems injecting
   the same repository means that data has two owners.
2. **Downward along a declared order.** Decide which subsystems may call which, keep it acyclic,
   and write it down — then enforce it where you can (an import-linter `layers` contract over the
   orchestrator modules checks it on every commit). Reporting and ops/admin surfaces typically sit
   on top; core resources sit at the bottom.
3. **Against the order, go asynchronous.** When a lower subsystem's work should trigger a higher
   one (a fulfilled order kicking off a report refresh, one pipeline stage starting the next),
   it schedules a task or emits an event rather than importing the higher orchestrator.

```
reporting ─▶ orders ─▶ catalog
checkout  ─┬▶ orders
           └▶ catalog
fulfillment ─▶ orders            (fulfillment → reporting: via a task, never a call)
```

### Orchestrators Own Transactions

Each orchestrator entry point is a unit of work. The unit is *reentrant*: the outermost entry
point commits (or rolls back) once; an entry point called from another orchestrator joins the
caller's transaction instead of committing on its own. That keeps cross-subsystem delegation
atomic without either side knowing who called it, and keeps routers and dependencies out of
transaction management entirely. The unit-of-work implementation is in the data-access skill's
reference.

What this buys: coupling inside the domain is visible — one facade per subsystem, one declared
order between them — rather than spread through a web of service-to-service calls nobody chose on
purpose. Each service can be understood, tested, and reused alone, and the graph stays a shallow
DAG instead of an unchecked mesh.

### Client Wrappers

```python
# ❌ Service imports the SDK directly
from acmepay import AcmePay

class RefundService:
    async def refund(self, charge_id: str, cents: int):
        AcmePay(api_key=settings.acme_key).refunds.create(charge=charge_id, amount=cents)

# ✅ Client isolates the SDK; service depends on the ABC
class PaymentGatewayClient(ABC):
    @abstractmethod
    async def refund(self, *, charge_id: str, amount_cents: int) -> GatewayRefund: ...

class AcmePayGatewayClient(PaymentGatewayClient):
    def __init__(self, sdk: AcmePay):
        self._sdk = sdk

    async def refund(self, *, charge_id: str, amount_cents: int) -> GatewayRefund:
        try:
            r = await self._sdk.refunds.create(charge=charge_id, amount=amount_cents)
        except AcmePayError as e:
            raise PaymentGatewayError(provider_code=e.code) from e
        return GatewayRefund(id=r.id, status=GatewayRefundStatus(r.status))
```

### Data Type Boundaries

ORM records never cross out of the data layer — as return types or as parameters:

```python
# ❌ Leaks the ORM type into a service's input contract
class PriceCalculator:
    def quote(self, products: list[ProductRecord]) -> Quote: ...

# ✅ Caller maps first
class PriceCalculator:
    def quote(self, products: list[Product]) -> Quote: ...
```

### Utilities by Layer

`app/utils/` is for layer-agnostic helpers (text truncation, id generation). A helper that only
makes sense with SQL (LIKE escaping, query builders) lives in `app/data/utils/`.

### App-Agnostic Libraries

A `libs/<name>/` module holds a reusable mechanism (a task-queue consumer harness, a metrics
registry). It never imports app modules (`app.config`, `app.logger`, `app.metrics`) — it
declares a small Protocol and the app binds it in a top-level module (`app/metrics.py` binds
`libs/observability`'s registry to the app's metric names).

## Composition over Inheritance

```python
# ❌ Inheritance for code sharing
class BaseExporter:
    def export(self): ...
class CsvExporter(BaseExporter):
    def export(self): ...  # overrides most of it

# ✅ Compose
class OrderExporter:
    def __init__(self, formatter: RowFormatter, sink: ExportSink):
        ...
```

## FastAPI Conventions

Every route declares its response model — `response_model=` or the return annotation, one
style per codebase. A handler returning a bare dict has no validated shape, no schema in the
OpenAPI spec, and no single place to stop a new field from leaking.

```python
# ❌ Bare dict — models write this by default
@router.get("/products/{product_id}/availability")
async def get_availability(product: ProductDep):
    return {"id": product.id, "title": product.title, "available": product.stock - product.reserved}

# ✅ Declared response model
class AvailabilityResponse(DomainModel):
    id: str
    title: str
    available: int

@router.get("/products/{product_id}/availability")
async def get_availability(product: ProductDep) -> AvailabilityResponse:
    return AvailabilityResponse(id=product.id, title=product.title, available=product.stock - product.reserved)
```

OpenAPI metadata belongs on the decorator:

```python
@router.post(
    "/refunds",
    response_model=RefundResponse,
    summary="Issue a refund against an order",
    responses={404: {"description": "Order not found"}, 409: {"description": "Exceeds refundable amount"}},
)
async def create_refund(request: CreateRefundRequest, ...): ...
```

Multi-field constraints as a model validator (returns FastAPI's standard 422 and shows in the
schema):

```python
class CreateRefundRequest(DomainModel):
    order_id: str
    amount_cents: int | None = None
    full_refund: bool = False

    @model_validator(mode="after")
    def _exactly_one_amount(self) -> "CreateRefundRequest":
        if self.full_refund == (self.amount_cents is not None):
            raise ValueError("pass exactly one of amount_cents or full_refund")
        return self
```

`AfterValidator` doesn't emit schema constraints — pair it with `Field(...)`:

```python
Sku = Annotated[str, AfterValidator(_validate_sku), Field(min_length=3, max_length=64)]
```

## Feature Flags vs Configuration

`Settings` holds values that differ between environments (URLs, credentials, timeouts).
Runtime behavior toggles belong in the feature-flag system — granular, per capability:

```python
# ❌ A flag masquerading as config — all or nothing, needs a deploy
class Settings(BaseSettings):
    refunds_enabled: bool = False

# ✅ Granular flags
class FeatureFlag(StrEnum):
    REFUNDS_SELF_SERVICE = "refunds_self_service"
    REFUNDS_AUTO_APPROVE = "refunds_auto_approve"
```

## Continuous Refactoring

Revisit design assumptions as requirements change; leave code better than you found it. Watch
for abstractions that no longer match the domain and code that grew without being reshaped.

# Testing Reference

Detailed patterns and examples for testing. Examples use a multi-tenant marketplace:
**stores**, **staff**, **orders** with **line items** and **refunds**, **customers**, a
payment gateway client, and an LLM-drafted support reply.

## Strategy

### Integration Tests (Default)

```python
# ✅ Real service, real repositories, real database; fake network
def test_issue_refund(make_refund_service, sample_paid_order, staff_user, fake_gateway):
    service = make_refund_service()
    refund = service.issue_refund(staff_user, sample_paid_order.id, amount_cents=500)
    assert refund.status == RefundStatus.PENDING
    assert refund.amount_cents == 500
    assert fake_gateway.refunds == [("ch_1", 500)]
```

### Unit Tests (Pure Functions)

```python
@pytest.mark.parametrize("raw,expected", [
    ("", ""),
    ("sku-001", "SKU-001"),
    ("  sku 001 ", "SKU-001"),
    ("SKU_001", "SKU-001"),
])
def test_normalize_sku(raw, expected):
    assert normalize_sku(raw) == expected
```

### When Not to Add a Test

- Trivial getters/setters.
- Tests that only verify a mock was called.
- Variations of an already-covered path with different data and the same assertion.
- A dedicated test for a default value — add `assert record.is_gift is False` to the existing
  happy-path test instead.
- Thin glue:

```python
# ❌ Three patches, zero logic under test
def test_export_orders_task():
    service = MagicMock()
    with (
        patch("app.tasks.export.SessionFactory"),
        patch("app.tasks.export.make_export_service", return_value=service),
    ):
        _export_orders_task("st_1")
    service.export.assert_called_once()

# ✅ Test the service the task delegates to
def test_export_writes_paid_orders(export_service, sample_paid_order, fake_sink):
    export_service.export(sample_paid_order.store_id)
    assert [row.order_id for row in fake_sink.rows] == [sample_paid_order.id]
```

## Organization

```
app/services/refund_service.py         → tests/services/test_refund_service.py
app/services/tax_calculator.py         → tests/services/test_tax_calculator.py
app/data/repositories/order_repository.py → tests/data/repositories/test_order_repository.py
```

### Test the Unit Under Test, Only

```python
# ❌ In test_checkout_orchestrator.py — arranges checkout, but asserts the tax calculator's rounding
def test_checkout_rounds_tax_half_even(make_checkout_orchestrator, ...):
    order = make_checkout_orchestrator().complete(...)
    assert order.tax_cents == 13   # TaxCalculator's rule; belongs in test_tax_calculator.py
```

When modules are merged or renamed, re-home each old test to the file of the interface it pins —
don't move a dead module's tests wholesale into the survivor's file.

### Layered Coverage

Each layer owns the edge cases it implements. The layer above exercises the integrated path and
re-tests a lower edge case only where it surfaces in *its own* contract — and then it asserts the
translation:

```python
# Service test — owns WHY and WHEN the domain raises
def test_refund_over_capture_raises(make_refund_service, staff_user, sample_paid_order):
    with pytest.raises(RefundExceedsCaptureError):
        make_refund_service().issue_refund(staff_user, sample_paid_order.id, amount_cents=10**9)

# Router test — owns only the mapping to HTTP
def test_create_refund_over_capture_returns_409(client, sample_paid_order):
    response = client.post(f"/orders/{sample_paid_order.id}/refunds", json={"amount_cents": 10**9})
    assert response.status_code == 409
    assert response.json()["code"] == "refund_exceeds_capture"

# ❌ Router test re-asserting business logic
def test_create_refund_sets_pending_and_reserves_amount_via_router(client, ...):
    ...
    assert body["status"] == "pending"
    assert body["remaining_refundable_cents"] == 4500   # service logic, already covered
```

The same relation holds at every pair: orchestrator/service, service/repository.

### Module-Level Functions

```python
# ❌
class TestRefundService:
    def test_issue(self): ...

# ✅
def test_issue_refund(make_refund_service): ...
```

## Quality

### Public Interface Only

```python
# ❌
def test_line_total(service):
    assert service._line_total(item) == 1500

# ✅
def test_quote_totals_line_items(service, sample_cart):
    assert service.quote(sample_cart).subtotal_cents == 1500
```

### No Ceremony

```python
# ❌
def test_normalize_sku():
    """Test SKU normalization."""
    # Arrange
    raw = "sku-001"
    # Act
    result = normalize_sku(raw)
    # Assert
    assert result == "SKU-001"

# ✅
def test_normalize_sku():
    assert normalize_sku("sku-001") == "SKU-001"
```

### Test Users Match the Authorization

If only admins can reach an operation through the API, its service tests use an admin user. A
staff user tests a path the API makes unreachable — and may pass for the wrong reason.

### Denial Tests

Every tenant-scoped read and every protected by-id endpoint gets a test proving another tenant
can't see or change the object. Cheap to write, and the regression it catches is the worst one
a multi-tenant app can ship:

```python
def test_get_order_from_another_store_is_not_found(client_for, sample_order, other_store_user):
    response = client_for(other_store_user).get(f"/orders/{sample_order.id}")
    assert response.status_code == 404

def test_get_order_scoped_to_store(order_repo, sample_order, other_store):
    assert order_repo.get_order(other_store.id, sample_order.id) is None
```

If visibility is finer than the tenant (users see only records they own, or teams only their
region), add the same test for another user in the same tenant. A parametrized helper over
"every by-id route" makes the cross-tenant case nearly free to keep complete.

## Mocking

### At Network Boundaries

```python
# ✅ Library facility
transport = httpx.MockTransport(lambda req: httpx.Response(200, json={"id": "rf_1", "status": "succeeded"}))
client = AcmePayGatewayClient(http=httpx.AsyncClient(transport=transport))

# ✅ A fake implementing the client ABC
class FakePaymentGateway(PaymentGatewayClient):
    def __init__(self):
        self.refunds: list[tuple[str, int]] = []

    async def refund(self, *, charge_id: str, amount_cents: int) -> GatewayRefund:
        self.refunds.append((charge_id, amount_cents))
        return GatewayRefund(id=f"rf_{len(self.refunds)}", status=GatewayRefundStatus.SUCCEEDED)

# ❌ Patching internals
with patch.object(service._gateway, "refund") as m: ...
```

### Never Mock the Database

```python
# ❌
with patch.object(OrderRepository, "get_order", return_value=OrderRecord(...)):
    ...

# ✅
def test_get_order(order_repo, sample_order):
    assert order_repo.get_order(sample_order.store_id, sample_order.id).id == sample_order.id
```

### Verify via Repositories

```python
# ❌ Raw SQL bypasses the read interface (and hides ORM/schema drift)
rows = session.execute(select(RefundRecord).where(RefundRecord.order_id == order.id)).scalars().all()

# ✅
refunds = refund_repo.list_refunds(order.store_id, order_id=order.id)
```

Cleanup fixtures may use direct SQL where no repository method exists.

### Configure Mocks via Fixtures

```python
# ❌
def test_x(refund_service):
    refund_service._gateway.refund.return_value = ...

# ✅
def test_x(refund_service, gateway: MagicMock):
    gateway.refund.return_value = ...
```

## Fixtures

### Naming and Tiers

```python
@pytest.fixture
def make_customer(db, sample_store):
    def _make(*, email: str = "pat@example.test", is_guest: bool = False) -> CustomerRecord:
        record = CustomerRecord(store_id=sample_store.id, email=email, is_guest=is_guest)
        db.add(record)
        db.commit()
        return record
    return _make

@pytest.fixture
def sample_customer(make_customer) -> CustomerRecord:
    return make_customer()

@pytest.fixture
def sample_paid_order(make_order, sample_customer, sample_staff) -> OrderRecord:
    return make_order(customer=sample_customer, owner=sample_staff, status=OrderStatus.PAID)
```

`sample_*` = a ready record graph at a meaningful domain state, composed from lower `sample_*`
fixtures, each independently addressable. `make_*` = one variant of one record.

### Extend Factories; Don't Mutate Samples

```python
# ❌ Hides the variant in test code, risks stale reads, leaks into other tests in the scope
@pytest.fixture
def guest_customer(sample_customer, db):
    sample_customer.is_guest = True
    db.commit()
    return sample_customer

# ✅ Declarative and reusable
@pytest.fixture
def guest_customer(make_customer):
    return make_customer(is_guest=True)
```

If factories have typed Protocols, update the Protocol with the new parameter.

### Factory Fixtures for the Class Under Test

```python
# ❌ Repeated inline construction
def test_allows_under_limit(redis):
    limiter = RateLimiter(redis, max_attempts=3, window_seconds=10)

# ✅
@pytest.fixture
def make_rate_limiter(redis):
    def _make(*, max_attempts: int = 5, window_seconds: int = 60) -> RateLimiter:
        return RateLimiter(redis, max_attempts=max_attempts, window_seconds=window_seconds)
    return _make
```

### Where Fixtures Live

pytest discovers fixtures from `conftest.py` files; a fixture is available to every test at or
below the directory its `conftest.py` lives in. Place each fixture at the lowest level that
covers all its users:

```
tests/
├── conftest.py                 # suite-wide: session, sample_store, make_customer, fake_gateway, markers
├── fixtures/
│   ├── orders.py               # large fixture groups, loaded from the root conftest:
│   └── payments.py             #   pytest_plugins = ["tests.fixtures.orders", "tests.fixtures.payments"]
├── routers/
│   └── conftest.py             # router-only: app, client, client_for(user), autouse fake flags
└── services/
    └── test_refund_service.py  # uses fixtures by name — never imports them
```

- Search the existing `conftest.py` files before writing a fixture; a fixture needed by two test
  modules moves up to the `conftest.py` that covers both, never copied.
- Never `from tests.fixtures import *` in a test or conftest — use `pytest_plugins` in the root
  `conftest.py` so pytest's discovery, scoping, and override rules still apply.
- A module-local override of a shared fixture (same name, different scope or setup) is fine with a
  comment saying why.

### Records Deleted by the Test

Tracking factories that delete their records at teardown need an opt-out when the test itself
deletes the record (e.g. testing a delete endpoint): `make_customer(skip_teardown=True)`.

## Feature Flags

```python
@pytest.mark.parametrize("enabled", [True, False])
def test_refund_auto_approve(make_refund_service, fake_flags, staff_user, sample_paid_order, enabled):
    fake_flags.set(FeatureFlag.REFUNDS_AUTO_APPROVE, enabled)
    refund = make_refund_service().issue_refund(staff_user, sample_paid_order.id, amount_cents=500)
    assert refund.status == (RefundStatus.APPROVED if enabled else RefundStatus.PENDING)
```

Router test modules autouse a fake flag client so no test ever calls a real flag service.

## External Services (Opt-In)

```python
@pytest.mark.llm_validation   # custom marker: skipped unless selected with -m
async def test_support_reply_declines_to_promise_refund(prompter, sample_ticket):
    ...
```

### Probabilistic Assertions: Repeat and Threshold

```python
async def assert_pass_rate(check: Callable[[], Awaitable[bool]], *, runs: int, min_passes: int) -> None:
    results = await asyncio.gather(*(check() for _ in range(runs)))
    passes = sum(results)
    assert passes >= min_passes, f"{passes}/{runs} passed (need {min_passes})"

@pytest.mark.llm_validation
@pytest.mark.parametrize("ticket_text", [...])
async def test_reply_never_promises_refund(prompter, ticket_text):
    async def _check() -> bool:
        reply = await prompter.draft_support_reply(ticket_text)
        return "refund" not in reply.commitments
    await assert_pass_rate(_check, runs=8, min_passes=7)
```

State N and the floor in the test; run the draws concurrently.

### Cleanup

```python
# Per-fixture teardown
@pytest.fixture
def cart_key(redis):
    key = f"test:cart:{uuid4().hex}"
    yield key
    redis.delete(key)

# Autouse when every test in the module needs it
@pytest.fixture(autouse=True)
def _reset_fakes(fake_sink, fake_gateway):
    yield
    fake_sink.reset()
    fake_gateway.reset()
```

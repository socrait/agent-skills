# Security Reference

Detailed patterns and examples for security, multi-tenancy, and data protection. Examples use
a multi-tenant marketplace: **stores** are tenants; store **staff** and **admins** sign in;
**orders** contain **line items**, **refunds**, and **shipments**; **customers** carry PII.

The authenticated principal in the examples is a small model resolved by a FastAPI dependency:

```python
class CurrentUser(DomainModel):
    id: str
    store_id: str
    permissions: frozenset[Permission]

async def get_current_user(token: Annotated[str, Depends(oauth2_scheme)], ...) -> CurrentUser: ...

def require(permission: Permission) -> Callable[..., CurrentUser]:
    def _check(user: Annotated[CurrentUser, Depends(get_current_user)]) -> CurrentUser:
        if permission not in user.permissions:
            raise HTTPException(status_code=403, detail="Forbidden")
        return user
    return _check
```

Map these onto whatever your codebase uses (a session cookie, a JWT claims object, a policy
library). The checks below are about *what* must be proven, not the mechanism.

## Declare Authorization Where It's Visible

```python
# ✅ The requirement is part of the route's signature
@router.post("/orders/{order_id}/refunds")
async def create_refund(
    order_id: str,
    request: CreateRefundRequest,
    user: Annotated[CurrentUser, Depends(require(Permission.ISSUE_REFUNDS))],
    service: Annotated[RefundService, Depends(get_refund_service)],
) -> RefundResponse: ...

# ❌ Authorization buried in the body, easy to forget on the next endpoint
@router.post("/orders/{order_id}/refunds")
async def create_refund(order_id: str, request: CreateRefundRequest, request_ctx: Request):
    user = request_ctx.state.user
    if user is None:
        ...
```

Router-level dependencies (`APIRouter(dependencies=[Depends(get_current_user)])`) make
"authenticated by default" structural, so a forgotten dependency fails closed.

## Object Level Authorization

Every by-id access proves the caller may touch that object. The simplest robust form is to load
the object through a query constrained to the caller, and treat "not found under the
constraint" as not found:

```python
# ❌ BOLA: any authenticated user of any store can read any order by guessing ids
@router.get("/orders/{order_id}")
async def get_order(order_id: str, user: CurrentUserDep, db: DbDep) -> OrderResponse:
    order = db.get(OrderRecord, order_id)
    ...

# ✅ Constrained to the caller's tenant; another store's order is indistinguishable from a missing one
def get_order_for_user(db: Session, user: CurrentUser, order_id: str) -> OrderRecord | None:
    return db.execute(
        select(OrderRecord)
        .where(OrderRecord.id == order_id)
        .where(OrderRecord.store_id == user.store_id)
    ).scalar_one_or_none()
```

Reusing one loader dependency per resource type keeps the check identical everywhere:

```python
def load_order(order_id: str, user: CurrentUserDep, db: DbDep) -> Order:
    record = get_order_for_user(db, user, order_id)
    if record is None:
        raise HTTPException(status_code=404, detail="Order not found")
    return map_order_to_model(record)

OrderDep = Annotated[Order, Depends(load_order)]
```

If your app has visibility finer than the tenant (users see only records they own, or teams
see only their region), the same constraint carries that dimension too — and it should live in
one place, not be re-typed per query.

### Child Resources

A refund is authorized through its order: the check proves both that the caller can access the
order and that the refund belongs to it.

```python
# ❌ Loads the refund by id alone; the order in the path is decorative
@router.get("/orders/{order_id}/refunds/{refund_id}")
async def get_refund(order: OrderDep, refund_id: str, db: DbDep):
    return db.get(RefundRecord, refund_id)

# ✅ The refund must belong to the authorized order
    refund = db.execute(
        select(RefundRecord).where(RefundRecord.id == refund_id).where(RefundRecord.order_id == order.id)
    ).scalar_one_or_none()
```

### Unchecked Id Lists

```python
# ❌ The order is authorized, but line items load by id within the store.
#    A caller passes line_item_ids from a different order and refunds another customer's purchase.
async def create_refund(order: OrderDep, request: CreateRefundRequest, db: DbDep):
    items = db.execute(
        select(LineItemRecord)
        .where(LineItemRecord.store_id == order.store_id)
        .where(LineItemRecord.id.in_(request.line_item_ids))
    ).scalars().all()

# ✅ Derive the in-scope set from the authorized parent; intersect
async def create_refund(order: OrderDep, request: CreateRefundRequest, line_items: LineItemRepoDep):
    in_scope = {li.id: li for li in line_items.list_for_order(order.id)}
    requested = set(request.line_item_ids)
    if not requested <= in_scope.keys():
        raise HTTPException(status_code=404, detail="Line item not found")
    items = [in_scope[i] for i in requested]
```

Reviewer cue: any client-supplied id that flows into an `IN (...)` lookup is suspect. Ask which
authorized parent it's supposed to belong to, and where that's checked.

## Function Level Authorization

```python
# ❌ Any signed-in staff member can approve refunds
@router.post("/refunds/{refund_id}/approve")
async def approve_refund(refund_id: str, user: CurrentUserDep, ...): ...

# ✅ The privileged operation states its permission
@router.post("/refunds/{refund_id}/approve")
async def approve_refund(
    refund_id: str,
    user: Annotated[CurrentUser, Depends(require(Permission.APPROVE_REFUNDS))],
    ...
): ...
```

Admin-only routes, exports, bulk operations, and settings changes are the usual places this is
missed — especially when they're added to a router whose existing endpoints are all low-privilege.

## Object Property Level Authorization

### Excessive Data Exposure

```python
# ❌ Authorized for sales *aggregates*, but returns per-order customer detail
@router.get("/reports/sales")
async def sales_report(
    user: Annotated[CurrentUser, Depends(require(Permission.VIEW_SALES_AGGREGATES))],
    db: DbDep,
):
    orders = list_orders_for_store(db, user.store_id)
    return SalesReport(rows=[
        SalesRow(customer=o.customer_name, email=o.customer_email, total_cents=o.total_cents) for o in orders
    ])

# ✅ An aggregate-shaped query and a response model with no per-row identity
async def sales_report(...):
    return SalesAggregateReport(days=sum_sales_by_day(db, user.store_id, tz=store_tz))
```

Fix it at the query and the response model, so the handler never even holds the detail rows.
Filtering fields out of a full object at serialization time is fragile — the next field added to
the model leaks by default.

### Mass Assignment

```python
# ❌ The request model mirrors the record, so the client can set store_id, status, or total
class UpdateOrderRequest(DomainModel):
    store_id: str | None = None
    status: OrderStatus | None = None
    shipping_address: Address | None = None

# ✅ The request model contains only what the client may change
class UpdateShippingAddressRequest(DomainModel):
    shipping_address: Address
```

Tenant, owner, and creator values on new rows come from `user`, never from the payload:

```python
refund = RefundRecord(store_id=user.store_id, created_by=user.id, order_id=order.id, amount_cents=request.amount_cents)
```

## Denial Shape

- Not visible to the caller (other tenant, not theirs) → **404**. Don't confirm existence.
- Visible, but the operation isn't permitted → **403**, bland body.
- Internal authorization reason codes go to logs only.

## Input Validation

```python
class CreateRefundRequest(DomainModel):
    line_item_ids: list[str] = Field(min_length=1, max_length=100)
    reason: RefundReason
    note: str | None = Field(default=None, max_length=500)
```

Validate at the boundary; don't re-validate the same constraint at every layer. Membership (does
this id belong to this parent?) is not schema validation — it needs the intersection above.

## Injection Prevention

When user input feeds a `LIKE`/`ILIKE` pattern, escape `%` and `_` in it as well as binding it
as a parameter — otherwise a search for `50%` matches far more than intended.

```python
# ❌ User input in a shell
subprocess.run(f"convert {upload_name} thumb.png", shell=True)

# ✅ List form, validated path
subprocess.run(["convert", str(validated_path), "thumb.png"], check=True)
```

## Secret Handling

Type secrets in settings as `SecretStr`, so an accidental `repr()` or log line prints `**********`.

```python
# ❌
log.info("calling_gateway", api_key=settings.payment_gateway_api_key.get_secret_value())

# ✅
log.debug("calling_gateway", key_present=bool(settings.payment_gateway_api_key))
```

### Catch-All Handlers

```python
# ❌ Forwards whatever the exception says — SQL, paths, PII
except Exception as e:
    raise HTTPException(status_code=500, detail=f"Refund failed: {e}") from e

# ✅ Log server-side, generic message to the client
except Exception:
    log.exception("refund_failed")
    raise HTTPException(status_code=500, detail="An internal error occurred")
```

## PII Protection

### What Counts

- Customer names, emails, phone numbers, shipping/billing addresses
- Customer identifiers that link to the above (internal and external ids)
- Order history, browsing/purchase behavior, support-ticket contents
- Payment metadata (last four, expiry, tokens). Full card numbers never enter the service.

### LLM Prompt Minimization

Every piece of PII in a prompt can surface in the output, whatever the instructions say.

```python
# ❌ The whole customer list in context to draft one reply
prompt = render_support_prompt(ticket=ticket, customers=list_customers(db, user.store_id))

# ✅ Only the subject — and only the fields the reply needs
prompt = render_support_prompt(
    ticket=ticket,
    customer=SupportCustomerContext(first_name=customer.first_name, recent_orders=recent_orders),
)
```

### Logging PII

```python
# ❌
log.info("refund_issued", email=customer.email)

# ✅ ids at info; PII only at debug, truncated
log.info("refund_issued", customer_id=customer.id, refund_id=refund.id)
log.debug("refund_recipient", email=truncate_text(customer.email))
```

### Error Responses

```python
# ❌
raise HTTPException(status_code=404, detail=f"No order for {customer.email}")

# ✅
raise HTTPException(status_code=404, detail="Order not found")
```

### Caches Holding PII

```python
# ✅ Bounded TTL, fails loud on a bad value
def cache_customer_profile(self, key: str, profile: CustomerProfile, *, ttl: timedelta) -> None:
    if ttl <= timedelta(0) or ttl > MAX_PII_RETENTION:
        raise InvalidCacheTtlError(ttl)
    self._redis.set(key, profile.model_dump_json(), ex=ttl)

# ❌ A helper that silently skips the expiry when handed None — the data now lives forever
if ttl:
    self._redis.set(key, value, ex=ttl)
else:
    self._redis.set(key, value)
```

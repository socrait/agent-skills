# Resilience Reference

Detailed patterns and examples for error handling, logging, and observability. Examples use a
multi-tenant marketplace: **stores**, **orders**, **refunds**, **customers**, a payment
gateway, an email provider, and a nightly order-export worker.

## Error Handling

### When to Catch

```python
# ✅ (a) Log with context only the catch site knows
try:
    capture = await self._gateway.capture(intent_id)
except PaymentGatewayError:
    log.exception("capture_failed", order_id=order.id, intent_id=intent_id)
    raise

# ✅ (b) Translate to a domain exception
try:
    self._session.flush()
except IntegrityError as e:
    raise DuplicateRefundError(order_id) from e

# ✅ (c) Recover — and the recovery is real
try:
    rates = await self._shipping.quote(parcel)
except ShippingTimeoutError:
    log.warning("shipping_quote_timeout_using_flat_rate", order_id=order.id)
    rates = [FLAT_RATE]

# ❌ Catch-and-rethrow that adds nothing
try:
    return self._service.process(order)
except Exception as e:
    raise e
```

### No Dead Catch Clauses

```python
# ❌ The body only calls httpx — EmailSendError can't be raised here
try:
    response = await self._http.post(url, json=payload)
    response.raise_for_status()
except EmailSendError:
    raise
except Exception as exc:
    raise EmailSendError(str(exc)) from exc

# ✅ Catch what can actually be thrown; controlled messages
try:
    response = await self._http.post(url, json=payload)
    response.raise_for_status()
except httpx.HTTPStatusError as exc:
    raise EmailSendError("email provider rejected the request", provider_status=exc.response.status_code) from exc
except httpx.TransportError as exc:
    raise EmailSendError("email provider unreachable") from exc
```

### Exception Scope by Layer

```python
# ✅ A top-level boundary — the one place a broad catch belongs (here: a worker entrypoint)
async def handle_export_requested(message: ExportRequested) -> None:
    try:
        await export_service.export(message.store_id)
    except Exception:
        log.exception("export_job_failed", store_id=message.store_id)
        raise  # let the queue's retry/dead-letter policy decide

# ❌ A service swallowing everything — callers get None and no one learns why
class TaxService:
    def rate_for(self, address: Address) -> Decimal | None:
        try:
            return self._client.lookup(address)
        except Exception:
            return None
```

### Domain Exceptions

```python
class AppError(Exception):
    """Base for errors the application knows how to describe."""
    code: ErrorCode = ErrorCode.INTERNAL_ERROR
    status_code: int = 500
    public_message: str = "An internal error occurred"

class RefundNotFoundError(AppError):
    code = ErrorCode.REFUND_NOT_FOUND
    status_code = 404
    public_message = "Refund not found"

    def __init__(self, refund_id: str):
        super().__init__(f"Refund not found: {refund_id}")   # internal message, for logs
        self.refund_id = refund_id
```

Two messages on purpose: the exception's own message is for logs and may carry ids; the
`public_message` is static and safe to send to a client. HTTP-specific exceptions stay in the
router layer, never in the core module.

### Query vs Mutation Semantics

```python
# Query: None for not-found
def get_refund(self, store_id: str, refund_id: str) -> RefundRecord | None: ...

# Mutation: raise on not-found
def cancel_refund(self, store_id: str, refund_id: str) -> RefundRecord:
    ...
    if refund is None:
        raise RefundNotFoundError(refund_id)
```

Idempotent exception: a soft delete may succeed when the row is already deleted — the "already"
state is genuinely the requested state.

```python
def soft_delete_product(self, store_id: str, product_id: str) -> None:
    self._session.execute(
        update(ProductRecord)
        .where(ProductRecord.store_id == store_id)
        .where(ProductRecord.id == product_id)
        .where(ProductRecord.deleted_at.is_(None))
        .values(deleted_at=func.now())
    )
```

### Fail Fast on Internal Invariants

Defensive programming belongs at *untrusted input boundaries*. Inside the trust boundary, a
violated invariant should raise early rather than be papered over. The common smell is an
upsert that rewrites a field the system treats as stable identity — it looks like idempotency,
but the row is *changing*, not settling.

```python
# ❌ A saved payment token silently moves to a different customer.
#    A client bug? A stolen token? The code can't tell — and now neither can you.
self._session.execute(
    insert(PaymentMethodRecord)
    .values(token=token, customer_id=customer_id, store_id=store_id)
    .on_conflict_do_update(index_elements=["token"], set_={"customer_id": customer_id})
)

# ✅ The violation surfaces
existing = self._session.execute(
    select(PaymentMethodRecord).where(PaymentMethodRecord.token == token)
).scalar_one_or_none()
if existing and existing.customer_id != customer_id:
    raise PaymentTokenOwnershipMismatchError(token_id=existing.id)

# 🟡 If recovery is deliberately allowed, the contract says so and callers must log it
def save_payment_method(...) -> tuple[PaymentMethodRecord, bool]:
    """Returns (record, did_rebind). Callers log when did_rebind is True."""
```

Why:
1. **Bugs get localized** — the loud failure points at the cause instead of a downstream symptom.
2. **Silent recovery erodes invariants** — every "just handle it" branch declares that state legal.
3. **Security-adjacent invariants deserve paranoia** — tokens, identity, tenancy: benign and
   malicious explanations are indistinguishable in code; you want a signal for both.

If you catch yourself writing code that "handles any state the database might be in", ask
whether that state *should* be possible. If not, raise. If yes, put it in the contract.

### Batch Error Isolation

```python
@dataclass
class ExportStats:
    exported: int = 0
    failed_stores: int = 0

# ❌ One store's failure stops every store after it
for store in stores:
    rows = self._build_rows(store)
    self._sink.write(store.id, rows)

# ✅ Each item attempted independently; one summary
stats = ExportStats()
for store in stores:
    try:
        rows = self._build_rows(store)
        self._sink.write(store.id, rows)
        stats.exported += len(rows)
    except (ExportSinkError, SQLAlchemyError):
        log.exception("store_export_failed", store_id=store.id)
        stats.failed_stores += 1
log.info("order_export_completed", exported=stats.exported, failed_stores=stats.failed_stores)
metrics.increment_counter(MetricName.ORDERS_EXPORTED, value=stats.exported)
return stats
```

The stats object keeps tracking out of the loop body and gives one place to log, emit metrics,
and return outcomes to callers and tests.

### External Calls

```python
self._http = httpx.AsyncClient(timeout=httpx.Timeout(connect=5.0, read=30.0, write=10.0, pool=5.0))
```

A call without a timeout can hold a worker or request slot forever. Retry only operations that
are safe to repeat (idempotent, or with an idempotency key).

### Safe Messages in Domain Exceptions

`str(exc)` of a caught exception can contain hostnames, SQL, tokens, or a traceback. Don't embed
it — in client responses *or* in domain exceptions that flow into logs and metrics. Capture
provider error codes as typed attributes:

```python
class PaymentGatewayError(AppError):
    code = ErrorCode.PAYMENT_GATEWAY_ERROR
    status_code = 502
    public_message = "Payment provider error"

    def __init__(self, message: str, *, provider_code: str | None = None):
        super().__init__(message)
        self.provider_code = provider_code  # None for transport failures

# In the client
except AcmePayCardError as e:
    raise PaymentGatewayError("card declined by gateway", provider_code=e.code) from e
except httpx.TransportError as e:
    raise PaymentGatewayError("payment gateway unreachable") from e
```

## Error Responses

### One Contract, One Place

Clients get the same shape for every error: a stable machine-readable code they can branch on,
and a static message safe to display. The translation from domain exception to response happens
in exception handlers registered once on the app — not in each route.

```python
@app.exception_handler(AppError)
async def handle_app_error(request: Request, exc: AppError) -> JSONResponse:
    if exc.status_code >= 500:
        log.error("app_error", code=exc.code, exc_info=exc)
        return JSONResponse(status_code=exc.status_code, content={"code": ErrorCode.INTERNAL_ERROR, "message": "An internal error occurred"})
    log.info("app_error", code=exc.code, status_code=exc.status_code)
    return JSONResponse(status_code=exc.status_code, content={"code": exc.code, "message": exc.public_message})

@app.exception_handler(Exception)
async def handle_unexpected(request: Request, exc: Exception) -> JSONResponse:
    log.error("unhandled_error", exc_info=exc)
    return JSONResponse(status_code=500, content={"code": ErrorCode.INTERNAL_ERROR, "message": "An internal error occurred"})
```

Routes then just raise domain exceptions (or let services raise them):

```python
# ✅
@router.post("/refunds/{refund_id}/cancel")
async def cancel_refund(refund_id: str, user: CurrentUserDep, service: RefundServiceDep) -> RefundResponse:
    return service.cancel(user, refund_id)    # RefundNotFoundError → 404 {"code": "refund_not_found", ...}

# ❌ Per-route translation, leaking internals
except RefundNotFoundError as e:
    raise HTTPException(status_code=404, detail=str(e))   # "Refund not found: rf_123 for store st_9"
```

Use the same field names as your validation errors (FastAPI's 422s) or reshape those too, so
clients see one error format. If you already follow a standard such as RFC 9457 (problem
details), the code maps to `type` and the message to `title`.

### Rules

- **Never `str(e)` in a response.** Static messages only; the internal message goes to logs.
- **The code comes from the exception type**, never from parsing its message.
- **5xx never exposes domain codes or internals** — always the generic internal-error code.
- **Anti-enumeration:** distinct failures that must look identical to a client (wrong password vs.
  unknown email) share one code and message. Log the real cause; don't swallow it.

```python
class InvalidCredentialsError(AppError):
    code = ErrorCode.INVALID_CREDENTIALS
    status_code = 401
    public_message = "Invalid credentials"

class UnknownEmailError(InvalidCredentialsError): ...      # same code and message on the wire
class WrongPasswordError(InvalidCredentialsError): ...
```

## Logging (structlog)

### Events, Not Sentences

Each log call is an event: a short, stable, `snake_case` name plus key-value fields. Fields stay
queryable as fields in your log store; the event name is something you can count and alert on.

```python
# ❌ Interpolated sentences — fields are trapped inside a string
log.info(f"Issued refund {refund.id} for order {order.id} amount {amount}")
log.info("Issued refund %s for order %s", refund.id, order.id)

# ✅ Event name + fields
log.info("refund_issued", refund_id=refund.id, order_id=order.id, amount_cents=amount)
```

Key-value logging is also lazy where it matters: fields are only rendered if the event passes
the level filter, so there's no reason to pre-format.

### Binding Context

Correlating identifiers should appear on every event without being repeated at every call:

```python
# Request middleware: bound once per request, carried by every event in it
@app.middleware("http")
async def bind_request_context(request: Request, call_next):
    structlog.contextvars.clear_contextvars()
    structlog.contextvars.bind_contextvars(request_id=request.headers.get("x-request-id") or new_request_id())
    return await call_next(request)

# Within an operation: bind what you learn, as early as you learn it
def issue_refund(self, user: CurrentUser, order_id: str, ...) -> Refund:
    log = logger.bind(store_id=user.store_id, user_id=user.id, order_id=order_id)
    log.debug("refund_requested", amount_cents=request.amount_cents)
    ...
    log.info("refund_issued", refund_id=refund.id)
```

`structlog.contextvars.merge_contextvars` must be in the processor chain for contextvars to
appear. Bind identifiers that trace the operation; put one-off values (a threshold, a count) on
the event that needs them.

```python
# ❌ A threshold isn't an identifier; binding it attaches it to every later event
log = logger.bind(min_order_cents=min_order_cents)

# ✅
logger.debug("abandoned_cart_sweep_started", min_order_cents=min_order_cents)
```

### Levels

| Level | Use for | Example |
|---|---|---|
| `debug` | intermediate steps, state changes | `log.debug("line_items_resolved", count=len(items))` |
| `info` | top-level entry/exit, summaries | `log.info("refund_issued", refund_id=refund.id)` |
| `warning` | recoverable, unlikely-but-handled | `log.warning("inventory_cache_miss", product_id=pid)` |
| `error` / `exception` | failures | `log.exception("gateway_capture_failed", order_id=order.id)` |

### Exceptions

```python
# ✅ Traceback captured as structured data
except PaymentGatewayError:
    log.exception("gateway_capture_failed", order_id=order.id)

# ❌ Traceback flattened into the message
except PaymentGatewayError as e:
    log.error(f"Capture failed: {traceback.format_exc()}")
```

`log.exception(...)` reads the exception *currently being handled*, so it only captures a
traceback when called inside an `except` block (or code awaited from one). Anywhere else —
an exception handler you don't control the caller of, a callback, a helper called after the
`except` block ends — it logs the event with **no traceback and no warning**. When you hold the
exception object, pass it explicitly:

```python
# ✅ Works wherever the exception object is in hand
async def handle_unexpected(request: Request, exc: Exception) -> JSONResponse:
    log.error("unhandled_error", exc_info=exc)

# ❌ Depends on being called from inside the framework's except block
async def handle_unexpected(request: Request, exc: Exception) -> JSONResponse:
    log.exception("unhandled_error")
```

Use `structlog.processors.dict_tracebacks` (JSON output) or `format_exc_info` in the chain so
`exc_info` renders.

### Bounded Values

```python
log.debug("line_items_selected", line_item_ids=truncate_list(line_item_ids))   # first N + count
log.debug("customer_note_received", note=truncate_text(note, max_length=120))
log.debug("payment_token_saved", token=mask_secret(token))                     # "tok_8f***"
log.debug("order_loaded", placed_at=order.placed_at.isoformat() if order.placed_at else None)
```

Unbounded values blow up log volume and can smuggle PII and secrets into the log store. If your
codebase lacks the helpers, they are a few lines each; a JSON renderer configured with a
`default` that ISO-formats datetimes removes the last line's ceremony.

### Batch Logging

Log what you're about to attempt and what happened — otherwise "sent 0" is ambiguous between
"nothing to send" and "everything failed":

```python
log.debug("order_confirmations_sending", count=len(messages))
...
log.info("order_confirmations_sent", sent=sent, attempted=len(messages))
```

### Canonical Log Lines

One rich summary event per top-level operation, carrying everything needed to understand what
happened (the bound ids come along automatically):

```python
log.info(
    "checkout_completed",
    order_id=order.id,
    line_items=len(order.line_items),
    total_cents=order.total_cents,
    payment_status=capture.status,
    duration_ms=elapsed_ms,
)
```

## Metrics

### Count the Business Unit

```python
# ❌ +1 per API call — one call may send 1 email or 500
metrics.increment_counter(MetricName.EMAILS_SENT)

# ✅ +N per batch
metrics.increment_counter(MetricName.EMAILS_SENT, value=len(batch))
```

Ask: "If this goes up by 1, what happened?" If the answer is ambiguous, it's the wrong unit.

### Low Cardinality

```python
# ❌ One time series per order
metrics.increment_counter(MetricName.REFUNDS_ISSUED, attributes={"order_id": order.id})

# ✅ Bounded attribute values
metrics.increment_counter(MetricName.REFUNDS_ISSUED, attributes={"reason": refund.reason})
```

### Error Categorization

Exceptions carrying an error code give metrics a bounded attribute for free — the boundary that
catches them records `code=exc.code` without per-site mapping.

## Monitoring Philosophy

- Instrumentation has cost — log what aids debugging, measure what you'll query.
- Levels exist so noise doesn't drown signal.
- Every metric corresponds to a question on a dashboard or an alert.
- Canonical log lines per top-level operation ([brandur.org/canonical-log-lines](https://brandur.org/canonical-log-lines)).
- Classic reading: Jim Shore, "Fail Fast" (IEEE Software, 2004).

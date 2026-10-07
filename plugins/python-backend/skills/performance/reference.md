# Performance Reference

Detailed patterns and examples for application-level performance. Examples use a multi-tenant
marketplace: **stores**, **orders**, **products**, nightly **exports**, a payment gateway, and a
Redis-backed cache. For database performance (N+1, indexes, eager loading, counting), see the
data-access skill.

## Streaming and Generators

```python
# ❌ Buffers every row of a store's history
def export_rows(orders: Iterable[Order]) -> list[str]:
    rows = []
    for order in orders:
        rows.append(format_csv_row(order))
    return rows

# ✅ Constant memory
def export_rows(orders: Iterable[Order]) -> Iterator[str]:
    for order in orders:
        yield format_csv_row(order)
```

### Chunking

```python
from itertools import batched  # 3.12+

for chunk in batched(order_ids, 500):
    process(chunk)
```

### Streaming Responses

```python
@router.get("/exports/orders.csv")
async def download_orders(...):
    return StreamingResponse(export_service.stream_csv(user.store_id), media_type="text/csv")
```

## Batching

```python
# ❌ One gateway call per order
for order in orders:
    status = await gateway.get_charge(order.charge_id)

# ✅ One call if the API supports it
statuses = await gateway.get_charges([o.charge_id for o in orders])

# ❌ One INSERT per line item
for item in items:
    line_item_repo.create_line_item(item)

# ✅ Bulk insert
line_item_repo.create_line_items(items)
```

## Async

### Don't Block the Event Loop

On request paths and worker loops, any synchronous I/O stalls every other coroutine on that
worker until it returns. It doesn't matter for one-shot scripts where nothing else is waiting.

```python
# ❌ Sync client inside an async path
class ShippingClient:
    def __init__(self):
        self._http = httpx.Client(timeout=10)

    def quote(self, parcel: Parcel) -> ShippingQuote:
        return parse(self._http.post(URL, json=parcel.model_dump()))

@router.post("/checkout/quote")
async def quote(...):
    return shipping.quote(parcel)   # the whole worker freezes for the round-trip

# ✅ Async end to end
class ShippingClient:
    def __init__(self, http: httpx.AsyncClient):
        self._http = http

    async def quote(self, parcel: Parcel) -> ShippingQuote:
        return parse(await self._http.post(URL, json=parcel.model_dump()))
```

Common culprits: `httpx.Client` / `requests`; `open()` / `Path.read_text()` on large files;
`time.sleep()`; `subprocess.run()`; sync SDKs. The rule covers the whole chain — a sync client
three layers down blocks the loop as surely as one in the handler. CPU-heavy work (PDF
rendering, image resizing) goes to `asyncio.to_thread` or a worker.

The most common culprit in FastAPI code is the database:

```python
# ❌ async route, sync Session — every query stalls every other request on the worker
@router.get("/orders/{order_id}")
async def get_order(order_id: str, session: Session = Depends(get_session)):
    return session.get(OrderRecord, order_id)

# ✅ Either a plain def route (FastAPI runs it in a threadpool)...
@router.get("/orders/{order_id}")
def get_order(order_id: str, session: Session = Depends(get_session)): ...

# ✅ ...or async end to end
@router.get("/orders/{order_id}")
async def get_order(order_id: str, session: AsyncSession = Depends(get_async_session)):
    return await session.get(OrderRecord, order_id)
```

### Concurrent Independent I/O

```python
# ❌ Total time = sum of calls
quotes = []
for carrier in carriers:
    quotes.append(await carrier.quote(parcel))

# ✅ Total time ≈ slowest call
quotes = await asyncio.gather(*(c.quote(parcel) for c in carriers))

# ✅ Bounded when N is large
sem = asyncio.Semaphore(10)
async def _bounded(store_id: str) -> Report:
    async with sem:
        return await build_report(store_id)
reports = await asyncio.gather(*(_bounded(s) for s in store_ids))
```

### Async Generators

```python
async def stream_order_events(store_id: str) -> AsyncIterator[OrderEvent]:
    async for message in consumer.subscribe(f"orders:{store_id}"):
        yield OrderEvent.model_validate_json(message.data)
```

## Hot Paths

```python
# ❌ Recomputed every iteration
for item in cart.items:
    rates = load_tax_table(store.region)
    item.tax = rates.for_category(item.category)

# ✅ Hoisted
rates = load_tax_table(store.region)
for item in cart.items:
    item.tax = rates.for_category(item.category)
```

```python
# ❌ O(n) membership per check
blocked = list(blocked_customer_ids)
for order in orders:
    if order.customer_id in blocked: ...

# ✅ O(1)
blocked = set(blocked_customer_ids)
```

## Caching

### Bounded In-Process Caches

```python
# ❌ Grows forever
_PRICE_CACHE: dict[str, Money] = {}

def price_for(sku: str) -> Money:
    if sku not in _PRICE_CACHE:
        _PRICE_CACHE[sku] = compute_price(sku)
    return _PRICE_CACHE[sku]

# ✅ Bounded
@lru_cache(maxsize=10_000)
def tax_rate_for(region: str, category: TaxCategory) -> Decimal: ...
```

Note the first example has a second bug: the same SKU in two stores can carry different prices.

### Keys Carry Every Dimension

```python
# ❌ Store A's catalog served to store B
key = f"catalog:{category_id}"

# ✅
key = f"catalog:v2:{store_id}:{category_id}:{locale}"
```

Include tenant, user (if per-user), locale, timezone (if output depends on it), and a schema
version you can bump to invalidate everything.

### Invalidation First

```python
async def update_product(self, store_id: str, product_id: str, patch: ProductPatch) -> Product:
    product = self._repo.update_product(store_id, product_id, patch)
    await self._cache.delete(f"product:v2:{store_id}:{product_id}")
    return product
```

Decide how an entry becomes stale before adding the cache: explicit delete on write, TTL, or a
version bump.

### Redis Read-Through

```python
async def get_catalog(self, store_id: str, category_id: str) -> Catalog:
    key = f"catalog:v2:{store_id}:{category_id}"
    if (cached := await self._redis.get(key)) is not None:
        return Catalog.model_validate_json(cached)
    catalog = await self._build_catalog(store_id, category_id)
    await self._redis.set(key, catalog.model_dump_json(), ex=300)
    return catalog
```

## Resource Management

```python
# ✅
with SessionFactory() as session: ...
async with aiofiles.open(path, "rb") as f: ...
with tempfile.TemporaryDirectory() as tmp: ...

# ❌ Leaks on any exception
session = SessionFactory()
result = OrderRepository(session).get_order(store_id, order_id)
```

### Pooling

```python
# ✅ Created once in the app lifespan, injected everywhere
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.http = httpx.AsyncClient(
        limits=httpx.Limits(max_connections=50, max_keepalive_connections=20),
        timeout=httpx.Timeout(10.0),
    )
    yield
    await app.state.http.aclose()

# ❌ New connection pool per request
async def fetch_rates():
    async with httpx.AsyncClient() as http:
        return await http.get(URL)
```

## Anti-Patterns Summary

| Anti-pattern | Better |
|---|---|
| Buffering a whole collection | Generator / chunks / streaming response |
| N calls in a loop | One batch call |
| List for membership tests | Set |
| Recomputing a loop-invariant value | Hoist it |
| Client per request | Pooled client from app lifespan |
| Sequential independent awaits | `asyncio.gather` (bounded) |
| Unbounded dict cache | `lru_cache(maxsize=…)` / TTL cache |
| Cache key missing tenant | Key on every output dimension |
| Sync I/O in async code | Async-native APIs / `to_thread` |
| Manual resource cleanup | `with` / `async with` |

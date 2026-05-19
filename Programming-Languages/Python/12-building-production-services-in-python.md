---
title: "12 - Building Production Services in Python"
created: 2026-05-19
updated: 2026-05-19
tags: []
aliases: []
---

# 12 - Building Production Services in Python

[toc]

> **TL;DR:** Production Python services in 2025 are built on FastAPI (async HTTP), Pydantic v2 (validation), async database drivers (asyncpg, SQLAlchemy async), structured logging with OpenTelemetry spans, and graceful shutdown via `asyncio` signal handlers. The stack is fully async, statically typed, and instrumented end-to-end — enabling the observability required to operate a service at scale.

## Vocabulary

**FastAPI**: An async Python web framework built on Starlette and Pydantic. Generates OpenAPI docs automatically from type annotations. Routing, dependency injection, and request validation are first-class.

---

**Starlette**: The ASGI framework that FastAPI is built on. Handles HTTP request/response, WebSockets, background tasks, and middleware. Can be used directly for lower-level control.

---

**ASGI (Asynchronous Server Gateway Interface)**: The async equivalent of WSGI. The interface between an async Python web application and an async HTTP server (uvicorn, hypercorn).

---

**Uvicorn**: An ASGI server built on `uvloop` (libuv-backed event loop). The standard production server for FastAPI. Typically run behind nginx or a load balancer.

---

**Pydantic v2**: A data validation library that uses Python type annotations as the schema. Rewritten in Rust (pydantic-core) for v2 — 5–50× faster than v1. `BaseModel` subclasses auto-validate on instantiation.

---

**asyncpg**: A PostgreSQL client for asyncio. Does not use SQLAlchemy's ORM — raw SQL with typed result rows. The fastest Python PostgreSQL driver.

---

**SQLAlchemy async**: SQLAlchemy 2.0's async session API (`AsyncSession`). Uses `asyncpg` or `aiosqlite` as the backend. Provides the ORM on top of async I/O.

---

**Structured logging**: Log records as JSON objects with consistent fields (`timestamp`, `level`, `service`, `trace_id`, `span_id`, `message`). Machine-parseable by log aggregators (Datadog, CloudWatch, ELK).

---

**OpenTelemetry**: A vendor-neutral observability standard for traces, metrics, and logs. Python SDK: `opentelemetry-sdk`. Traces propagate across HTTP via `traceparent` header.

---

**Graceful shutdown**: On `SIGTERM`, the server stops accepting new connections, drains in-flight requests, closes database pools, and exits cleanly. Kubernetes sends `SIGTERM` before `SIGKILL`; missing graceful shutdown causes 502 errors during rolling deploys.

---

**Dependency injection** (FastAPI): A pattern where route handlers declare their dependencies as function parameters. FastAPI resolves and caches them per-request. Enables clean separation of database sessions, auth context, and feature flags from handler logic.

---

## Intuition

A production Python HTTP service is an async program that: receives HTTP requests on a Starlette ASGI app, validates input with Pydantic, queries PostgreSQL via asyncpg, emits OpenTelemetry spans for every operation, and streams structured JSON logs. The async model means a single process can handle hundreds of in-flight requests with minimal thread overhead. Pydantic v2 makes the API boundary safe — invalid requests are rejected before they touch business logic.

The pieces fit together because they are all async-native. The event loop runs the Starlette request handler, which awaits the asyncpg database call, which runs inside an OpenTelemetry span. Nothing blocks. The production concern is not concurrency — it is observability: can you explain why request P99 spiked at 2 AM?

## Project Structure for a Production Service

```
myservice/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── src/
│   └── myservice/
│       ├── __init__.py
│       ├── main.py          # FastAPI app + lifespan
│       ├── config.py        # Settings via pydantic-settings
│       ├── db.py            # asyncpg pool / SQLAlchemy async engine
│       ├── models.py        # Pydantic request/response models
│       ├── routers/
│       │   └── items.py     # Route handlers
│       └── telemetry.py     # OpenTelemetry setup
└── tests/
    └── test_items.py
```

## FastAPI Application

### Lifespan and Startup/Shutdown

FastAPI's `lifespan` context manager (preferred over deprecated `startup`/`shutdown` events) manages the application lifecycle.

```python
# src/myservice/main.py
from __future__ import annotations

import asyncio
import logging
import signal
from collections.abc import AsyncGenerator
from contextlib import asynccontextmanager

import uvicorn
from fastapi import FastAPI

from myservice.db import init_db_pool, close_db_pool
from myservice.routers import items
from myservice.telemetry import setup_telemetry

logger = logging.getLogger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    """Application lifespan: startup → yield → shutdown."""
    logger.info("Starting up: initialising DB pool and telemetry")
    await init_db_pool()
    setup_telemetry()
    yield
    logger.info("Shutting down: closing DB pool")
    await close_db_pool()


app = FastAPI(
    title="My Service",
    version="0.1.0",
    lifespan=lifespan,
)

app.include_router(items.router, prefix="/items", tags=["items"])


@app.get("/health")
async def health() -> dict[str, str]:
    return {"status": "ok"}
```

### Route Handlers with Pydantic Validation

```python
# src/myservice/routers/items.py
from __future__ import annotations

from typing import Annotated

from fastapi import APIRouter, Depends, HTTPException, status
from pydantic import BaseModel, Field, field_validator

from myservice.db import get_db_session

router = APIRouter()


class ItemCreate(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    price: float = Field(..., gt=0)
    tags: list[str] = Field(default_factory=list)

    @field_validator("name")
    @classmethod
    def name_must_not_be_blank(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("name must not be blank")
        return v.strip()


class ItemResponse(BaseModel):
    id: int
    name: str
    price: float
    tags: list[str]


@router.post("/", response_model=ItemResponse, status_code=status.HTTP_201_CREATED)
async def create_item(
    item: ItemCreate,
    db: Annotated[object, Depends(get_db_session)],
) -> ItemResponse:
    """Create a new item. Validates input via Pydantic before touching the DB."""
    # db would be an asyncpg connection or SQLAlchemy AsyncSession
    # row = await db.fetchrow("INSERT INTO items(name, price) VALUES ($1, $2) RETURNING *",
    #                         item.name, item.price)
    return ItemResponse(id=1, name=item.name, price=item.price, tags=item.tags)
```

> [!IMPORTANT]
> FastAPI calls Pydantic validation **before** the route handler runs. An `ItemCreate(name="", price=-1)` raises a `RequestValidationError` and returns HTTP 422 with a structured JSON error body — the handler never executes. This is the correct place for input validation. Do not re-validate inside the handler body.

## Pydantic v2 Models

Pydantic v2's core is implemented in Rust (`pydantic-core`). Validation is 5–50× faster than v1. Key v2 changes: `model_validator`, `field_validator` (class methods), `model_config` (dict or `ConfigDict`).

```python
from pydantic import BaseModel, ConfigDict, Field, model_validator
from typing import Self
import re


class UserCreate(BaseModel):
    model_config = ConfigDict(str_strip_whitespace=True)

    username: str = Field(pattern=r"^[a-zA-Z0-9_]{3,32}$")
    email: str
    password: str = Field(min_length=8)
    confirm_password: str

    @model_validator(mode="after")
    def passwords_must_match(self) -> Self:
        if self.password != self.confirm_password:
            raise ValueError("Passwords do not match")
        return self

    @staticmethod
    def _is_valid_email(email: str) -> bool:
        return bool(re.match(r"[^@]+@[^@]+\.[^@]+", email))


# Validation happens on instantiation
try:
    user = UserCreate(
        username="alice",
        email="alice@example.com",
        password="secret123",
        confirm_password="secret123",
    )
    print(user.model_dump())
except Exception as exc:
    print(exc)
```

## Async Database with asyncpg

`asyncpg` is the fastest Python PostgreSQL driver. It uses a connection pool and native PostgreSQL binary protocol.

```python
# src/myservice/db.py
from __future__ import annotations

import asyncpg  # type: ignore[import-untyped]
from fastapi import Request

_pool: asyncpg.Pool | None = None


async def init_db_pool(dsn: str = "postgresql://user:pass@localhost/mydb") -> None:
    global _pool
    _pool = await asyncpg.create_pool(
        dsn=dsn,
        min_size=2,
        max_size=10,
        command_timeout=60,
    )


async def close_db_pool() -> None:
    global _pool
    if _pool is not None:
        await _pool.close()
        _pool = None


async def get_db_session(request: Request) -> asyncpg.Connection:
    """FastAPI dependency: acquire a connection from the pool per request."""
    if _pool is None:
        raise RuntimeError("Database pool not initialised")
    async with _pool.acquire() as conn:
        yield conn  # type: ignore[misc]
```

```python
# Example query with typed results
async def get_items(conn: asyncpg.Connection, limit: int = 100) -> list[dict[str, object]]:
    rows = await conn.fetch(
        "SELECT id, name, price FROM items ORDER BY id LIMIT $1",
        limit,
    )
    return [dict(row) for row in rows]
```

> [!TIP]
> `asyncpg`'s prepared statements are cached per-connection automatically. The first call to `conn.fetch("SELECT ... $1", val)` prepares the statement; subsequent calls reuse the prepared plan. For hot paths, this eliminates query parsing overhead. Use `$1`, `$2` positional parameters (PostgreSQL syntax) not `%s` (psycopg2 syntax).

## Structured Logging

```python
# src/myservice/logging_config.py
import logging
import json
import sys
from typing import Any


class JSONFormatter(logging.Formatter):
    """Format log records as JSON objects."""

    def format(self, record: logging.LogRecord) -> str:
        log_data: dict[str, Any] = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
        }
        if record.exc_info:
            log_data["exception"] = self.formatException(record.exc_info)
        # Attach extra fields injected via logger.info("msg", extra={...})
        for key in ("trace_id", "span_id", "request_id", "user_id"):
            if hasattr(record, key):
                log_data[key] = getattr(record, key)
        return json.dumps(log_data)


def configure_logging(level: str = "INFO") -> None:
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(JSONFormatter())
    logging.basicConfig(level=level, handlers=[handler], force=True)
```

## OpenTelemetry Instrumentation

```python
# src/myservice/telemetry.py
from opentelemetry import trace  # type: ignore[import-untyped]
from opentelemetry.sdk.trace import TracerProvider  # type: ignore[import-untyped]
from opentelemetry.sdk.trace.export import BatchSpanProcessor  # type: ignore[import-untyped]
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter  # type: ignore[import-untyped]
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor  # type: ignore[import-untyped]
from opentelemetry.instrumentation.asyncpg import AsyncPGInstrumentor  # type: ignore[import-untyped]


def setup_telemetry(otlp_endpoint: str = "http://localhost:4317") -> None:
    provider = TracerProvider()
    exporter = OTLPSpanExporter(endpoint=otlp_endpoint, insecure=True)
    provider.add_span_processor(BatchSpanProcessor(exporter))
    trace.set_tracer_provider(provider)

    # Auto-instrument FastAPI routes and asyncpg queries
    FastAPIInstrumentor.instrument()
    AsyncPGInstrumentor().instrument()
```

## Graceful Shutdown

Kubernetes sends `SIGTERM` before `SIGKILL`. A graceful shutdown handler: stops accepting new connections, drains in-flight requests, then exits.

```python
# In main.py, when running directly (not via uvicorn CLI)
import asyncio
import signal
import sys


def install_signal_handlers(shutdown_event: asyncio.Event) -> None:
    """Install SIGTERM and SIGINT handlers that set a shutdown event."""
    loop = asyncio.get_event_loop()

    def _handle_signal(sig: signal.Signals) -> None:
        print(f"Received {sig.name}, shutting down...")
        shutdown_event.set()

    for sig in (signal.SIGTERM, signal.SIGINT):
        loop.add_signal_handler(sig, _handle_signal, sig)


# uvicorn handles graceful shutdown automatically when started via:
# uvicorn myservice.main:app --host 0.0.0.0 --port 8000 --timeout-graceful-shutdown 30
```

> [!WARNING]
> The Kubernetes default `terminationGracePeriodSeconds` is 30 seconds. If your service takes longer than 30 seconds to drain, Kubernetes sends `SIGKILL` and requests in-flight are dropped. Set `--timeout-graceful-shutdown` in uvicorn to 25 seconds (slightly under the Kubernetes limit) so uvicorn finishes before `SIGKILL` arrives.

## Real-world Example

A complete minimal FastAPI service with Pydantic v2 validation, asyncpg connection pool, structured logging, and a health endpoint.

```python
# Full minimal service — one file for clarity
from __future__ import annotations

import asyncio
import logging
import sys
from collections.abc import AsyncGenerator
from contextlib import asynccontextmanager
from typing import Annotated

import uvicorn
from fastapi import Depends, FastAPI, HTTPException, status
from pydantic import BaseModel, Field

# --- Logging setup ---
logging.basicConfig(
    level=logging.INFO,
    format='{"time":"%(asctime)s","level":"%(levelname)s","msg":"%(message)s"}',
    stream=sys.stdout,
)
logger = logging.getLogger("myservice")

# --- In-memory store (replace with asyncpg in production) ---
_items: dict[int, dict[str, object]] = {}
_next_id: int = 1


# --- Models ---
class ItemCreate(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    price: float = Field(..., gt=0.0)


class ItemResponse(BaseModel):
    id: int
    name: str
    price: float


# --- Lifespan ---
@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    logger.info("Service starting up")
    yield
    logger.info("Service shutting down")


# --- App ---
app = FastAPI(title="Items API", version="0.1.0", lifespan=lifespan)


@app.get("/health")
async def health() -> dict[str, str]:
    return {"status": "ok"}


@app.post("/items", response_model=ItemResponse, status_code=201)
async def create_item(item: ItemCreate) -> ItemResponse:
    global _next_id
    item_id = _next_id
    _next_id += 1
    _items[item_id] = {"id": item_id, "name": item.name, "price": item.price}
    logger.info("Created item id=%d name=%r", item_id, item.name)
    return ItemResponse(id=item_id, name=item.name, price=item.price)


@app.get("/items/{item_id}", response_model=ItemResponse)
async def get_item(item_id: int) -> ItemResponse:
    if item_id not in _items:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Item not found")
    data = _items[item_id]
    return ItemResponse(id=int(data["id"]), name=str(data["name"]), price=float(data["price"]))


if __name__ == "__main__":
    uvicorn.run(
        "myservice.main:app",
        host="0.0.0.0",
        port=8000,
        reload=False,
        log_level="info",
        timeout_graceful_shutdown=25,
    )
```

> [!NOTE]
> `uvicorn.run("myservice.main:app", ...)` takes a string import path, not the app object, when `reload=True` (hot reload). For production (`reload=False`), passing the object directly is fine. In Kubernetes deployments, the `CMD` in the Dockerfile is `["uvicorn", "myservice.main:app", "--host", "0.0.0.0", "--port", "8000"]` — not `python -m myservice`.

## In Practice

**Connection pool sizing.** For a PostgreSQL-backed service, the asyncpg pool `max_size` should be: `max_size = (postgres_max_connections - reserved) / num_app_instances`. A PostgreSQL instance typically handles 100–300 connections. With 5 app instances and 20 reserved for admin, `max_size = 16` per instance. Too many connections → PostgreSQL OOM; too few → queue buildup under load.

**Pydantic v2 performance.** Pydantic v2 validates in Rust, but the Python model class still has overhead for creating instances. For ultra-hot paths (millions of objects/second), use `model.model_validate(dict, from_attributes=True)` with `model_config = ConfigDict(from_attributes=True)` to avoid building an intermediate dict. For batch ingestion, `TypeAdapter.validate_json(raw_bytes)` skips the intermediate Python dict entirely.

**`py-spy` for production profiling.** `py-spy top --pid $(pgrep uvicorn)` attaches to a running uvicorn worker and shows live hot functions. No code changes, no restart, no overhead. Capture a flame graph with `py-spy record -o profile.svg --pid <pid> --duration 30`.

> [!CAUTION]
> Never log request bodies or response bodies at `DEBUG` level in production without scrubbing PII. Request bodies can contain passwords, credit card numbers, and personally identifiable information. Use a logging middleware that scrubs known sensitive field names (e.g. `password`, `token`, `ssn`) before logging. FastAPI's `Request.body()` is an `async` call — do not call it in a synchronous logging hook.

## Pitfalls

- **"FastAPI is WSGI-compatible."** — FastAPI is ASGI only. Deploying it behind a WSGI server (gunicorn without `uvicorn.workers.UvicornWorker`) will not work. Use `gunicorn -k uvicorn.workers.UvicornWorker` for multi-worker deployments.
- **"Pydantic models are free."** — Each instantiation runs full validation. For performance-critical loops over millions of records, profile Pydantic's contribution. `model_validate` with trusted data (internal service-to-service) is faster than the default validator.
- **"asyncpg auto-commits."** — No. `asyncpg` is in autocommit mode *unless* you use a transaction: `async with conn.transaction():`. Multiple `conn.execute` calls without a transaction may interleave with concurrent connections. Always use a transaction for multi-statement writes.
- **"I can run CPU-bound code in a FastAPI handler."** — You can, but it blocks the event loop for all other requests. Use `asyncio.get_event_loop().run_in_executor(None, cpu_func, args)` to run it in a thread pool, or `ProcessPoolExecutor` for heavy work.
- **"Graceful shutdown is handled by the container runtime."** — The container runtime sends `SIGTERM`; your process must handle it. If your lifespan context manager does not close the DB pool and drain in-flight requests, you will have connection leaks and 502 errors during rolling deploys.

## Exercises

### Exercise 1 — FastAPI dependency injection

Write a FastAPI dependency that provides a per-request request ID (UUID4) and injects it into a logger.

#### Solution

```python
from __future__ import annotations

import logging
import uuid
from typing import Annotated

from fastapi import Depends, FastAPI, Request


def get_request_id(request: Request) -> str:
    """Extract or generate a request ID."""
    return request.headers.get("X-Request-ID", str(uuid.uuid4()))


RequestId = Annotated[str, Depends(get_request_id)]

app = FastAPI()
logger = logging.getLogger("myservice")


@app.get("/demo")
async def demo(request_id: RequestId) -> dict[str, str]:
    logger.info("Handling request", extra={"request_id": request_id})
    return {"request_id": request_id}
```

FastAPI calls `get_request_id` once per request and caches the result for the scope of that request. Downstream dependencies can also declare `Annotated[str, Depends(get_request_id)]` and get the same cached value.

---

### Exercise 2 — Pydantic v2 validator

Write a Pydantic v2 model for a payment record that validates: amount > 0, currency is a 3-letter ISO code, card_last4 is exactly 4 digits.

#### Solution

```python
from pydantic import BaseModel, Field, field_validator
import re


class PaymentRecord(BaseModel):
    amount: float = Field(..., gt=0, description="Amount in the specified currency")
    currency: str = Field(..., min_length=3, max_length=3)
    card_last4: str

    @field_validator("currency")
    @classmethod
    def currency_must_be_uppercase(cls, v: str) -> str:
        if not v.isalpha() or not v.isupper():
            raise ValueError(f"Currency must be 3 uppercase letters, got {v!r}")
        return v

    @field_validator("card_last4")
    @classmethod
    def card_last4_must_be_digits(cls, v: str) -> str:
        if not re.fullmatch(r"\d{4}", v):
            raise ValueError(f"card_last4 must be exactly 4 digits, got {v!r}")
        return v


# Valid
p = PaymentRecord(amount=99.99, currency="USD", card_last4="1234")
print(p.model_dump())

# Invalid — raises ValidationError
try:
    PaymentRecord(amount=-1, currency="usd", card_last4="12ab")
except Exception as exc:
    print(exc)
```

---

### Exercise 3 — Graceful shutdown test

Explain what happens to an in-flight request when `SIGTERM` is sent to a uvicorn process and `timeout_graceful_shutdown=25` is set.

#### Solution

1. uvicorn receives `SIGTERM` and stops calling `accept()` on the listening socket — no new connections are established.
2. The 25-second graceful shutdown timer starts.
3. In-flight requests continue executing. Their event loop is still running, their coroutines still awaited.
4. When each request handler completes, uvicorn sends the response and closes the connection.
5. If all requests complete before 25 seconds, uvicorn exits cleanly.
6. If requests are still in-flight at 25 seconds, uvicorn cancels them (sends `CancelledError` to all running tasks), runs the lifespan teardown, and exits. Those requests receive a connection-close with no response — the client typically sees an empty response or connection reset.
7. The Kubernetes pod exits. Kubernetes marks it terminated and routes traffic to the remaining pods.

The correct pattern to avoid dropped requests: set `terminationGracePeriodSeconds` in the Kubernetes `PodSpec` to 35, and `timeout_graceful_shutdown` in uvicorn to 25. The 10-second gap allows the lifespan teardown (DB pool close) to complete within the Kubernetes window.

---

### Exercise 4 — Identify the blocking call

Find the blocking call in this FastAPI handler and fix it.

```python
import requests  # sync HTTP library
from fastapi import FastAPI

app = FastAPI()

@app.get("/proxy")
async def proxy(url: str) -> dict[str, object]:
    resp = requests.get(url)  # THIS LINE
    return {"status": resp.status_code, "length": len(resp.content)}
```

#### Solution

`requests.get(url)` is a blocking I/O call. Inside an `async def` handler, it blocks the event loop thread for the duration of the HTTP request — all other requests queue up and cannot be served.

Fix using `httpx.AsyncClient` (the async equivalent of `requests`):

```python
import httpx
from fastapi import FastAPI

app = FastAPI()

@app.get("/proxy")
async def proxy(url: str) -> dict[str, object]:
    async with httpx.AsyncClient() as client:
        resp = await client.get(url)  # non-blocking: yields to event loop while waiting
    return {"status": resp.status_code, "length": len(resp.content)}
```

For better performance, share the `AsyncClient` across requests (create it in the lifespan context manager and store in `app.state`), rather than creating a new client per request (which creates a new connection pool each time).

## Sources

- FastAPI documentation — https://fastapi.tiangolo.com/
- Pydantic v2 documentation — https://docs.pydantic.dev/
- asyncpg documentation — https://magicstack.github.io/asyncpg/
- Uvicorn documentation — https://www.uvicorn.org/
- OpenTelemetry Python SDK — https://opentelemetry-python.readthedocs.io/
- Starlette documentation — https://www.starlette.io/
- PEP 3333 — WSGI / ASGI specifications — https://peps.python.org/pep-3333/
- Slatkin, B. *Effective Python* (3rd ed., 2024). Chapter 8 — Robustness and Performance.

## Related

- [6 - Type Hints and Static Typing](./6-type-hints-and-static-typing.md)
- [7 - Exceptions and Context Managers](./7-exceptions-and-context-managers.md)
- [9 - asyncio and Coroutines](./9-asyncio-and-coroutines.md)
- [11 - Packaging, Tooling, Modern Workflows](./11-packaging-tooling-modern-workflows.md)

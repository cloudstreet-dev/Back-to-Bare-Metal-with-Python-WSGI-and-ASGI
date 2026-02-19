# Lifespan Events (Startup, Shutdown, and Existential Dread)

Every long-running application has things it needs to do before it starts serving requests, and things it needs to do before it stops. Connect to a database. Load a model into memory. Start a background task. Flush a cache. Close connection pools gracefully rather than dropping them mid-operation.

WSGI has no solution for this. You initialize things at module import time (which works) or you use Gunicorn's worker hooks (which are server-specific). Neither is portable.

ASGI has first-class support for it: the lifespan protocol.

## The Lifespan Scope

When an ASGI server starts your application, before processing any HTTP or WebSocket connections, it sends a lifespan scope:

```python
scope = {
    "type": "lifespan",
    "asgi": {"version": "3.0"},
}
```

Your application is called with this scope and a `receive`/`send` pair. The lifespan coroutine then runs for the entire application lifetime:

```
Server starts
    → calls app(lifespan_scope, receive, send)
    → app receives "lifespan.startup"
    → app does startup work
    → app sends "lifespan.startup.complete"
    → server starts accepting HTTP/WebSocket connections
    ...
    [Server receives SIGTERM]
    → server stops accepting new connections
    → finishes in-flight requests
    → sends "lifespan.shutdown" to the lifespan coroutine
    → app does cleanup
    → app sends "lifespan.shutdown.complete"
    → server exits
```

## Basic Lifespan Handler

```python
async def application(scope, receive, send):
    if scope["type"] == "lifespan":
        await handle_lifespan(receive, send)
    elif scope["type"] == "http":
        await handle_http(scope, receive, send)


async def handle_lifespan(receive, send):
    while True:
        event = await receive()

        if event["type"] == "lifespan.startup":
            try:
                await startup()
                await send({"type": "lifespan.startup.complete"})
            except Exception as e:
                await send({
                    "type": "lifespan.startup.failed",
                    "message": str(e),
                })
                return

        elif event["type"] == "lifespan.shutdown":
            try:
                await shutdown()
                await send({"type": "lifespan.shutdown.complete"})
            except Exception as e:
                await send({
                    "type": "lifespan.shutdown.failed",
                    "message": str(e),
                })
            return


async def startup():
    print("Application starting: connecting to database, loading caches...")
    # await db.connect()
    # await cache.warmup()


async def shutdown():
    print("Application shutting down: closing connections...")
    # await db.disconnect()
    # await cache.flush()
```

## Sharing State Between Lifespan and Handlers

Here's the practical problem: you initialize a database connection pool in `startup()`, but your HTTP handlers need to use it. How do you get it to them?

The common pattern: use a module-level state container.

```python
# state.py
from dataclasses import dataclass, field
from typing import Any, Optional


@dataclass
class AppState:
    db: Optional[Any] = None
    cache: Optional[Any] = None
    config: dict = field(default_factory=dict)


state = AppState()
```

```python
# app.py
import asyncio
import json
from state import state


async def startup():
    # In a real app, these would be actual async connections
    state.db = await create_db_pool()
    state.cache = await create_redis_client()
    state.config = await load_config()
    print(f"Started. DB: {state.db}, Cache: {state.cache}")


async def shutdown():
    if state.db:
        await state.db.close()
    if state.cache:
        await state.cache.close()
    print("Clean shutdown complete.")


async def create_db_pool():
    """Simulate creating a database connection pool."""
    await asyncio.sleep(0.1)  # Simulate async connection
    return {"pool": "connected", "size": 10}


async def create_redis_client():
    await asyncio.sleep(0.05)
    return {"redis": "connected"}


async def load_config():
    return {"env": "production", "debug": False}


async def application(scope, receive, send):
    if scope["type"] == "lifespan":
        await handle_lifespan(receive, send)
    elif scope["type"] == "http":
        await handle_http(scope, receive, send)


async def handle_lifespan(receive, send):
    while True:
        event = await receive()
        if event["type"] == "lifespan.startup":
            await startup()
            await send({"type": "lifespan.startup.complete"})
        elif event["type"] == "lifespan.shutdown":
            await shutdown()
            await send({"type": "lifespan.shutdown.complete"})
            return


async def handle_http(scope, receive, send):
    # Now we can use state.db, state.cache, etc.
    body = json.dumps({
        "db": str(state.db),
        "cache": str(state.cache),
        "config": state.config,
    }).encode()

    await send({
        "type": "http.response.start",
        "status": 200,
        "headers": [
            (b"content-type", b"application/json"),
            (b"content-length", str(len(body)).encode()),
        ],
    })
    await send({
        "type": "http.response.body",
        "body": body,
        "more_body": False,
    })
```

## Scope-Based State (The Starlette Pattern)

A cleaner pattern: store state in the ASGI scope itself. Starlette does this:

```python
async def application(scope, receive, send):
    if scope["type"] == "lifespan":
        await handle_lifespan(scope, receive, send)
    elif scope["type"] in ("http", "websocket"):
        # State from lifespan is available in scope["state"]
        await handle_http(scope, receive, send)


async def handle_lifespan(scope, receive, send):
    # Initialize a state dict in the scope
    scope["state"] = {}

    while True:
        event = await receive()
        if event["type"] == "lifespan.startup":
            scope["state"]["db"] = await create_db_pool()
            scope["state"]["started_at"] = time.time()
            await send({"type": "lifespan.startup.complete"})
        elif event["type"] == "lifespan.shutdown":
            await scope["state"]["db"].close()
            await send({"type": "lifespan.shutdown.complete"})
            return


async def handle_http(scope, receive, send):
    # Access state from scope
    db = scope.get("state", {}).get("db")
    # ...
```

Note: this works because Uvicorn passes the same `scope` object reference to both the lifespan call and each subsequent HTTP/WebSocket call. The state dict you attach in lifespan is available everywhere.

## Background Tasks

Lifespan is also where you start and stop background tasks — things that run continuously alongside request handling:

```python
import asyncio


background_tasks = set()


async def startup():
    # Start a periodic cleanup task
    task = asyncio.create_task(periodic_cleanup())
    background_tasks.add(task)
    task.add_done_callback(background_tasks.discard)

    # Start a health check task
    task = asyncio.create_task(report_health())
    background_tasks.add(task)
    task.add_done_callback(background_tasks.discard)


async def shutdown():
    # Cancel all background tasks
    for task in list(background_tasks):
        task.cancel()

    # Wait for them to finish
    if background_tasks:
        await asyncio.gather(*background_tasks, return_exceptions=True)


async def periodic_cleanup():
    while True:
        try:
            await asyncio.sleep(300)  # Every 5 minutes
            await cleanup_expired_sessions()
        except asyncio.CancelledError:
            break  # Graceful shutdown
        except Exception as e:
            print(f"Cleanup error: {e}")


async def report_health():
    while True:
        try:
            await asyncio.sleep(60)  # Every minute
            # Report to monitoring service
        except asyncio.CancelledError:
            break
        except Exception as e:
            print(f"Health report error: {e}")


async def cleanup_expired_sessions():
    pass  # Actual implementation would hit the database
```

The `asyncio.CancelledError` handling in the background tasks is important: when you call `task.cancel()` during shutdown, the task receives a `CancelledError`. If you don't catch it, the task exits with an exception. If you do catch it and don't re-raise, the task exits cleanly.

## What Happens If Lifespan Fails

If your `startup()` raises an exception and you send `lifespan.startup.failed`:

```python
await send({
    "type": "lifespan.startup.failed",
    "message": "Database connection refused",
})
```

Uvicorn will refuse to accept any connections and exit with an error. This is the right behavior — you don't want to serve requests with a broken database connection.

## Lifespan in Frameworks

**Starlette** (and FastAPI) have a `lifespan` parameter:

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    db = await create_db_pool()
    app.state.db = db
    yield
    # Shutdown
    await db.close()


app = FastAPI(lifespan=lifespan)
```

This is the modern FastAPI pattern. The `yield` separates startup (before yield) from shutdown (after yield). Under the hood, Starlette wraps this context manager in the lifespan protocol we've been discussing.

The older pattern used `@app.on_event("startup")` and `@app.on_event("shutdown")` decorators. These are now deprecated in favor of the `lifespan` context manager approach, which is cleaner because startup and shutdown code live together.

**Django** has `AppConfig.ready()` for initialization, but it runs synchronously at import time and has no shutdown hook. For ASGI Django, you'd use Starlette's lifespan middleware or a third-party library.

## Testing Lifespan

When testing ASGI apps, you need to send the lifespan events to initialize the app properly:

```python
import asyncio
import io


async def run_lifespan(app) -> asyncio.Event:
    """
    Trigger the app's startup sequence.
    Returns when startup is complete.
    """
    scope = {"type": "lifespan", "asgi": {"version": "3.0"}}
    events = asyncio.Queue()
    startup_complete = asyncio.Event()

    await events.put({"type": "lifespan.startup"})

    async def receive():
        return await events.get()

    async def send(event):
        if event["type"] == "lifespan.startup.complete":
            startup_complete.set()
        elif event["type"] == "lifespan.startup.failed":
            raise RuntimeError(event.get("message", "Startup failed"))

    asyncio.create_task(app(scope, receive, send))
    await startup_complete.wait()
    return events  # Return the queue so caller can send shutdown


async def test_with_lifespan():
    lifespan_queue = await run_lifespan(application)

    # Run your tests here
    # state is now initialized

    # Clean shutdown
    await lifespan_queue.put({"type": "lifespan.shutdown"})
```

In practice, test frameworks like `httpx.AsyncClient` with `app=` or Starlette's `TestClient` handle lifespan events automatically. The patterns chapter covers testing in detail.

## The Existential Dread Part

Here's the thing nobody warns you about: in production, your app will sometimes receive `SIGTERM` while handling a request. The lifespan shutdown is triggered, your background tasks are cancelled, and then — what happens to the in-flight request?

Uvicorn handles this with a graceful timeout: it stops accepting new connections, waits for in-flight requests to complete (up to a configurable timeout), then triggers the lifespan shutdown.

If you're doing something expensive in a request handler — a long database query, a slow external API call — you might hit the timeout. The request gets dropped, the client gets a connection error, and your cleanup runs anyway.

There's no perfect solution here. The best you can do is:
1. Set a reasonable graceful shutdown timeout (Uvicorn's `--timeout-graceful-shutdown`)
2. Make your handlers fast
3. Use database connection pools with timeouts
4. Accept that the occasional in-flight request will be dropped during deploys

Kubernetes rolling deployments, blue-green deploys, and load balancers with connection draining all help, but ultimately distributed systems are adversarial. Lifespan gives you a clean interface to handle it as well as possible.

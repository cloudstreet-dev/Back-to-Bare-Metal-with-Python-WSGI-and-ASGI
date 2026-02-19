# ASGI Middleware Deep Dive

ASGI middleware follows the same pattern as WSGI middleware: a callable that takes an app and returns an app. The difference is that everything is async, and the interface is `(scope, receive, send)` instead of `(environ, start_response)`.

The async nature introduces some subtleties worth understanding.

## The Simplest ASGI Middleware

```python
class DoNothingMiddleware:
    def __init__(self, app):
        self.app = app

    async def __call__(self, scope, receive, send):
        await self.app(scope, receive, send)
```

Or as a function:

```python
def do_nothing_middleware(app):
    async def wrapper(scope, receive, send):
        await self.app(scope, receive, send)
    return wrapper
```

Both are equivalent. Classes are slightly more common in ASGI middleware because they can hold configuration cleanly.

## Intercepting Requests and Responses

In WSGI, you intercept responses by wrapping `start_response`. In ASGI, you intercept them by wrapping `send`:

```python
class TimingMiddleware:
    def __init__(self, app):
        self.app = app

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return

        import time
        started = time.monotonic()
        status_holder = []

        async def send_with_timing(event):
            if event["type"] == "http.response.start":
                status_holder.append(event["status"])
            elif event["type"] == "http.response.body":
                if not event.get("more_body", False):
                    elapsed = (time.monotonic() - started) * 1000
                    method = scope.get("method", "?")
                    path = scope.get("path", "/")
                    status = status_holder[0] if status_holder else "?"
                    print(f"{method} {path} → {status} ({elapsed:.1f}ms)")
            await send(event)

        await self.app(scope, receive, send_with_timing)
```

The `send_with_timing` coroutine wraps `send` just like we wrapped `start_response` in WSGI. It intercepts the `http.response.start` event to capture the status code and the final `http.response.body` event to measure total time, then passes everything through.

## Modifying Requests

Intercept `receive` to modify incoming data:

```python
class RequestIDMiddleware:
    """Add a unique request ID to every request."""

    def __init__(self, app, header: str = "X-Request-ID"):
        self.app = app
        self.header = header.lower().encode("latin-1")

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return

        import uuid

        # Check for existing ID, generate one if missing
        existing_id = None
        for name, value in scope.get("headers", []):
            if name == self.header:
                existing_id = value.decode("latin-1")
                break

        request_id = existing_id or str(uuid.uuid4())

        # Inject into scope
        scope = dict(scope)
        headers = list(scope.get("headers", []))
        # Update or add the header
        new_headers = [(n, v) for n, v in headers if n != self.header]
        new_headers.append((self.header, request_id.encode("latin-1")))
        scope["headers"] = new_headers
        scope["request_id"] = request_id

        # Add request ID to response headers
        async def send_with_id(event):
            if event["type"] == "http.response.start":
                event = dict(event)
                event["headers"] = list(event.get("headers", [])) + [
                    (self.header, request_id.encode("latin-1"))
                ]
            await send(event)

        await self.app(scope, receive, send_with_id)
```

## Authentication Middleware

A complete authentication middleware that short-circuits the request if the token is invalid:

```python
import json
from typing import Optional


class BearerAuthMiddleware:
    """
    Validates Bearer tokens.
    Injects user information into scope["user"] if valid.
    Returns 401 if token is missing or invalid.
    Skips authentication for paths in exclude_paths.
    """

    def __init__(
        self,
        app,
        verify_token,  # async callable: token → user dict or None
        exclude_paths: list = None,
    ):
        self.app = app
        self.verify_token = verify_token
        self.exclude_paths = set(exclude_paths or [])

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return

        path = scope.get("path", "/")

        # Skip auth for excluded paths
        if path in self.exclude_paths:
            await self.app(scope, receive, send)
            return

        # Extract Bearer token
        token = self._extract_token(scope)

        if token is None:
            await self._send_401(send, "Missing Authorization header")
            return

        # Verify the token
        user = await self.verify_token(token)

        if user is None:
            await self._send_401(send, "Invalid or expired token")
            return

        # Inject user into scope
        scope = dict(scope)
        scope["user"] = user

        await self.app(scope, receive, send)

    def _extract_token(self, scope) -> Optional[str]:
        for name, value in scope.get("headers", []):
            if name == b"authorization":
                auth = value.decode("latin-1")
                if auth.startswith("Bearer "):
                    return auth[7:]
        return None

    async def _send_401(self, send, message: str) -> None:
        body = json.dumps({"error": message}).encode("utf-8")
        await send({
            "type": "http.response.start",
            "status": 401,
            "headers": [
                (b"content-type", b"application/json"),
                (b"content-length", str(len(body)).encode()),
                (b"www-authenticate", b'Bearer realm="API"'),
            ],
        })
        await send({
            "type": "http.response.body",
            "body": body,
            "more_body": False,
        })
```

Usage:

```python
async def verify_token(token: str):
    # Check your database or JWT
    if token == "valid-token":
        return {"id": 1, "username": "alice", "role": "admin"}
    return None


app = BearerAuthMiddleware(
    my_app,
    verify_token=verify_token,
    exclude_paths=["/health", "/login"],
)
```

## The GZIP Compression Middleware

A real-world example with non-trivial response manipulation:

```python
import gzip
import io


class GZipMiddleware:
    """
    Compress responses with gzip when:
    - Client sends Accept-Encoding: gzip
    - Response content type is compressible (text, JSON, etc.)
    - Response body is above minimum size threshold
    """

    COMPRESSIBLE_TYPES = {
        "text/html", "text/plain", "text/css", "text/javascript",
        "application/json", "application/javascript",
        "application/xml", "image/svg+xml",
    }

    def __init__(self, app, minimum_size: int = 500, compresslevel: int = 6):
        self.app = app
        self.minimum_size = minimum_size
        self.compresslevel = compresslevel

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return

        # Check if client accepts gzip
        accepts_gzip = False
        for name, value in scope.get("headers", []):
            if name == b"accept-encoding":
                accepts_gzip = b"gzip" in value
                break

        if not accepts_gzip:
            await self.app(scope, receive, send)
            return

        # We need to collect the full response to compress it
        response_started = []
        body_chunks = []

        async def collecting_send(event):
            if event["type"] == "http.response.start":
                response_started.append(event)
            elif event["type"] == "http.response.body":
                body_chunks.append(event.get("body", b""))
                if not event.get("more_body", False):
                    # All chunks collected — decide whether to compress
                    full_body = b"".join(body_chunks)
                    start_event = response_started[0]

                    content_type = ""
                    for name, value in start_event.get("headers", []):
                        if name == b"content-type":
                            content_type = value.decode("latin-1").split(";")[0].strip()
                            break

                    should_compress = (
                        len(full_body) >= self.minimum_size
                        and content_type in self.COMPRESSIBLE_TYPES
                    )

                    if should_compress:
                        compressed = gzip.compress(full_body,
                                                   compresslevel=self.compresslevel)

                        # Rebuild headers with encoding and new length
                        headers = [
                            (n, v) for n, v in start_event.get("headers", [])
                            if n not in (b"content-length", b"content-encoding")
                        ]
                        headers.append((b"content-encoding", b"gzip"))
                        headers.append((b"content-length",
                                        str(len(compressed)).encode()))

                        await send(dict(start_event, headers=headers))
                        await send({
                            "type": "http.response.body",
                            "body": compressed,
                            "more_body": False,
                        })
                    else:
                        # Send as-is
                        await send(start_event)
                        await send({
                            "type": "http.response.body",
                            "body": full_body,
                            "more_body": False,
                        })

        await self.app(scope, receive, collecting_send)
```

Note the limitation: we're collecting the entire response body before deciding whether to compress. For large streaming responses, this defeats the purpose of streaming. A streaming-compatible compressor would use `more_body=True` to send compressed chunks incrementally — at the cost of significantly more complexity.

## Building a Middleware Stack

```python
from functools import reduce
from typing import Callable, List


def build_middleware_stack(app: Callable, middleware: List) -> Callable:
    """
    Build an ASGI middleware stack.
    middleware = [A, B, C] → A(B(C(app)))
    Request order: A → B → C → app
    """
    for mw in reversed(middleware):
        if callable(mw) and not isinstance(mw, type):
            # It's a factory function, call it
            app = mw(app)
        elif isinstance(mw, type):
            # It's a class, instantiate it
            app = mw(app)
        else:
            # It's already an instance with __call__
            app = mw
    return app


# Usage
async def verify_token(token: str):
    return {"id": 1} if token == "secret" else None


application = build_middleware_stack(my_app, [
    TimingMiddleware,
    RequestIDMiddleware,
    GZipMiddleware,
    lambda app: BearerAuthMiddleware(app, verify_token=verify_token,
                                      exclude_paths=["/health"]),
])
```

## Middleware That Handles Lifespan

If your middleware needs its own startup/shutdown (e.g., opening a connection), handle the lifespan scope:

```python
class DatabaseMiddleware:
    """
    Opens a database connection pool at startup,
    injects it into each request's scope.
    """

    def __init__(self, app, database_url: str):
        self.app = app
        self.database_url = database_url
        self.pool = None

    async def __call__(self, scope, receive, send):
        if scope["type"] == "lifespan":
            await self._handle_lifespan(scope, receive, send)
            return

        # Inject pool into scope
        scope = dict(scope)
        scope["db"] = self.pool
        await self.app(scope, receive, send)

    async def _handle_lifespan(self, scope, receive, send):
        while True:
            event = await receive()
            if event["type"] == "lifespan.startup":
                self.pool = await self._connect()
                await self.app(scope, receive, send)  # Forward to inner app
                # Note: inner app handles the rest of lifespan
                return
            elif event["type"] == "lifespan.shutdown":
                if self.pool:
                    await self.pool.close()
                await send({"type": "lifespan.shutdown.complete"})
                return

    async def _connect(self):
        # asyncpg.create_pool(self.database_url), etc.
        print(f"Connected to {self.database_url}")
        return {"url": self.database_url, "status": "connected"}
```

This pattern — middleware that intercepts lifespan and injects resources into HTTP scopes — is how Starlette's `SessionMiddleware`, database integrations, and other resource-managing middleware work.

## Testing Middleware

Testing ASGI middleware directly, without a framework:

```python
import asyncio
import io


class MockASGI:
    """A fake ASGI server for testing middleware."""

    def __init__(self):
        self.received_events = []

    async def make_request(
        self,
        app,
        method: str = "GET",
        path: str = "/",
        headers: list = None,
        body: bytes = b"",
    ) -> tuple:
        scope = {
            "type": "http",
            "method": method,
            "path": path,
            "query_string": b"",
            "headers": headers or [],
            "server": ("testserver", 80),
        }

        request_events = [
            {"type": "http.request", "body": body, "more_body": False}
        ]
        event_index = [0]

        async def receive():
            idx = event_index[0]
            event_index[0] += 1
            if idx < len(request_events):
                return request_events[idx]
            return {"type": "http.disconnect"}

        response_events = []

        async def send(event):
            response_events.append(event)

        await app(scope, receive, send)

        return scope, response_events


async def test_timing_middleware():
    async def simple_app(scope, receive, send):
        await receive()
        await send({
            "type": "http.response.start",
            "status": 200,
            "headers": [(b"content-type", b"text/plain")],
        })
        await send({
            "type": "http.response.body",
            "body": b"OK",
            "more_body": False,
        })

    app = TimingMiddleware(simple_app)
    mock = MockASGI()
    scope, events = await mock.make_request(app, "GET", "/test")

    assert events[0]["type"] == "http.response.start"
    assert events[0]["status"] == 200
    assert events[1]["body"] == b"OK"
    print("TimingMiddleware test passed")


asyncio.run(test_timing_middleware())
```

## The Key Difference from WSGI Middleware

In WSGI middleware, you intercept `start_response` (a synchronous callable) to capture the status and headers before forwarding them. In ASGI middleware, you intercept the async `send` callable to capture `http.response.start` events.

The conceptual model is identical — wrap the callable, inspect and possibly modify events passing through — but the async nature means everything must be awaited. This is also why ASGI middleware can do things WSGI middleware can't: it can `await` during `send`, which means it can do async work (database calls, cache writes) as part of processing the response.

That's either powerful or terrifying, depending on how carefully your middleware is written.

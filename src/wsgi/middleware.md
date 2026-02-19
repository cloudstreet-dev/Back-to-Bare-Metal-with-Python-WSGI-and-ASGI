# Middleware: Turtles All the Way Down

Middleware is the part of Python web development that sounds complicated until you understand it, at which point it becomes almost disappointingly simple.

A WSGI middleware is a callable that:
1. Takes a WSGI application as its argument
2. Returns a WSGI application

That's the whole definition. It's a function that wraps a function to produce a function. Python has a name for this: a decorator.

## The Simplest Possible Middleware

```python
def do_nothing_middleware(app):
    """Middleware that does absolutely nothing."""
    def wrapper(environ, start_response):
        return app(environ, start_response)
    return wrapper
```

Use it:

```python
def my_app(environ, start_response):
    start_response("200 OK", [("Content-Type", "text/plain")])
    return [b"Hello"]

wrapped = do_nothing_middleware(my_app)
# wrapped is a WSGI app. Gunicorn can't tell the difference.
```

`wrapped` is indistinguishable from `my_app` from the server's perspective. It takes `environ` and `start_response`, calls `start_response`, and returns a body iterable. It's a valid WSGI app.

Now add some behavior:

## Request Logging Middleware

```python
import sys
import time


def logging_middleware(app):
    """Log method, path, status, and timing for every request."""
    def wrapper(environ, start_response):
        method = environ.get("REQUEST_METHOD", "?")
        path = environ.get("PATH_INFO", "/")
        started = time.monotonic()

        # Intercept start_response to capture the status code
        status_holder = []

        def capturing_start_response(status, headers, exc_info=None):
            status_holder.append(status)
            return start_response(status, headers, exc_info)

        result = app(environ, capturing_start_response)

        elapsed = (time.monotonic() - started) * 1000
        status = status_holder[0] if status_holder else "???"
        print(f"{method} {path} {status} ({elapsed:.1f}ms)", file=sys.stderr)

        return result

    return wrapper
```

The key insight: middleware can intercept `start_response` to inspect or modify the status and headers *before* passing them to the real `start_response`. This is how authentication middleware rejects requests with `401 Unauthorized`, how compression middleware adds `Content-Encoding: gzip`, and how CORS middleware adds `Access-Control-Allow-Origin` headers.

## Authentication Middleware

```python
import base64


def basic_auth_middleware(app, username: str, password: str):
    """
    HTTP Basic Authentication middleware.
    Rejects requests without valid credentials with 401.
    """
    # Pre-compute the expected auth header value
    credentials = f"{username}:{password}".encode("utf-8")
    expected = "Basic " + base64.b64encode(credentials).decode("ascii")

    def wrapper(environ, start_response):
        auth = environ.get("HTTP_AUTHORIZATION", "")

        if auth != expected:
            body = b"Unauthorized"
            start_response("401 Unauthorized", [
                ("Content-Type", "text/plain"),
                ("Content-Length", str(len(body))),
                ("WWW-Authenticate", 'Basic realm="Protected"'),
            ])
            return [body]

        # Credentials valid — call the real app
        return app(environ, start_response)

    return wrapper


# Usage:
protected_app = basic_auth_middleware(my_app, "admin", "secret")
```

Notice what happened: `basic_auth_middleware` takes the app *and* the credentials. It returns a closure. The closure has access to both `app` and `expected` via Python's closure mechanism.

## Stacking Middleware

Here's where the "turtles all the way down" name comes from. You can stack middleware:

```python
app = my_app
app = basic_auth_middleware(app, "admin", "secret")
app = logging_middleware(app)
```

When a request comes in, `app` is now the logging middleware. It calls `basic_auth_middleware`'s wrapper. That calls the real `my_app`. The stack unwinds on the way back.

The call stack during a request looks like:

```
logging wrapper
    → basic_auth wrapper
        → my_app
        ← response
    ← response (with status logged)
← response
```

This is exactly what Django's `MIDDLEWARE` setting builds. Each class in the list wraps the next.

## A Composable Middleware Builder

Rather than manually nesting, build a pipeline function:

```python
from typing import Callable, List


def build_middleware_stack(
    app: Callable,
    middleware: List[Callable],
) -> Callable:
    """
    Apply middleware in order, outermost last.
    middleware = [A, B, C] means: A wraps B wraps C wraps app
    Execution order: A → B → C → app → C → B → A
    """
    for mw in reversed(middleware):
        app = mw(app)
    return app


# Usage
stack = build_middleware_stack(my_app, [
    logging_middleware,
    basic_auth_middleware,  # This would need partial application
])
```

For middleware that takes configuration, use `functools.partial` or a factory function:

```python
import functools

stack = build_middleware_stack(my_app, [
    logging_middleware,
    functools.partial(basic_auth_middleware, username="admin", password="secret"),
])
```

## CORS Middleware

A real-world example: Cross-Origin Resource Sharing middleware adds the headers that browsers need for cross-origin requests.

```python
from typing import Optional


def cors_middleware(
    app,
    allow_origins: List[str] = ["*"],
    allow_methods: List[str] = ["GET", "POST", "PUT", "DELETE", "OPTIONS"],
    allow_headers: List[str] = ["Content-Type", "Authorization"],
    max_age: int = 86400,
):
    """
    CORS middleware. Adds Access-Control-* headers to all responses
    and handles OPTIONS preflight requests.
    """
    origin_header = ", ".join(allow_origins)
    methods_header = ", ".join(allow_methods)
    headers_header = ", ".join(allow_headers)

    cors_headers = [
        ("Access-Control-Allow-Origin", origin_header),
        ("Access-Control-Allow-Methods", methods_header),
        ("Access-Control-Allow-Headers", headers_header),
        ("Access-Control-Max-Age", str(max_age)),
    ]

    def wrapper(environ, start_response):
        # Handle CORS preflight
        if environ["REQUEST_METHOD"] == "OPTIONS":
            body = b""
            start_response("204 No Content", cors_headers + [
                ("Content-Length", "0"),
            ])
            return [body]

        # Inject CORS headers into every response
        def cors_start_response(status, headers, exc_info=None):
            return start_response(status, headers + cors_headers, exc_info)

        return app(environ, cors_start_response)

    return wrapper
```

## Request ID Middleware

Add a unique ID to every request — useful for correlating log lines across multiple services:

```python
import uuid


def request_id_middleware(app, header_name: str = "X-Request-ID"):
    """
    Assign a unique request ID to every incoming request.
    Uses the incoming header if present, generates one otherwise.
    Injects the ID into environ and adds it to the response headers.
    """
    environ_key = "HTTP_" + header_name.upper().replace("-", "_")

    def wrapper(environ, start_response):
        # Use existing ID or generate a new one
        request_id = environ.get(environ_key) or str(uuid.uuid4())
        environ[environ_key] = request_id
        environ["request_id"] = request_id  # Convenience key

        def id_start_response(status, headers, exc_info=None):
            return start_response(
                status,
                headers + [(header_name, request_id)],
                exc_info,
            )

        return app(environ, id_start_response)

    return wrapper
```

## Response Timing Header

```python
import time


def timing_middleware(app):
    """Add X-Response-Time header (milliseconds) to every response."""
    def wrapper(environ, start_response):
        started = time.monotonic()

        def timing_start_response(status, headers, exc_info=None):
            elapsed_ms = (time.monotonic() - started) * 1000
            return start_response(
                status,
                headers + [("X-Response-Time", f"{elapsed_ms:.2f}ms")],
                exc_info,
            )

        return app(environ, timing_start_response)

    return wrapper
```

## Putting the Stack Together

```python
# app.py
import functools

from tasks_app import application as tasks_app

app = build_middleware_stack(tasks_app, [
    timing_middleware,
    logging_middleware,
    request_id_middleware,
    functools.partial(cors_middleware, allow_origins=["https://myapp.com"]),
    functools.partial(basic_auth_middleware, username="admin", password="hunter2"),
])

# Run with: gunicorn app:app
```

The call order is: timing → logging → request_id → cors → basic_auth → tasks_app.

Each middleware layer handles one concern. None of them know about each other. They're composable because they all speak the same interface: WSGI.

## The Class-Based Pattern

You can also write middleware as classes, which is how Django does it:

```python
class LoggingMiddleware:
    def __init__(self, app):
        self.app = app

    def __call__(self, environ, start_response):
        method = environ.get("REQUEST_METHOD", "?")
        path = environ.get("PATH_INFO", "/")
        print(f"→ {method} {path}", file=sys.stderr)
        return self.app(environ, start_response)
```

The class is callable (via `__call__`), so `LoggingMiddleware(my_app)` returns a WSGI app. The behavior is identical to the function-based approach — Django's preference for classes is a stylistic choice, not a technical requirement.

## What Frameworks Add

When Flask does:

```python
from flask_cors import CORS
CORS(app)
```

It's wrapping `app.wsgi_app` with a CORS middleware. Not magic — WSGI middleware composition.

When Django processes your `MIDDLEWARE` list, it builds a chain of callables where each one wraps the next. The "middleware interface" Django defines (with `process_request`, `process_response`, `process_view`) is just a more structured way to write the same wrapping pattern.

The underlying mechanism is always: callables all the way down.

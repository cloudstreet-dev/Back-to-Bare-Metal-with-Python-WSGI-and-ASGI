# Roll Your Own Mini-Framework (For Fun and Understanding)

We have all the pieces. We've built a WSGI server, a router, middleware, and request/response objects. We've done the same for ASGI. Now let's assemble them into something coherent: a small, complete framework that you'd actually consider using for a personal project.

This chapter is about synthesis. The goal isn't to compete with Flask or Starlette — it's to make the jump from "I've built components" to "I understand how a framework is structured."

## What We're Building

A minimal ASGI framework called `bare` (fitting) with:
- Route registration via decorators
- Request and Response classes
- Middleware composition
- Lifespan event handling
- WebSocket support
- Zero dependencies beyond Python's standard library (plus `uvicorn` to run it)

The finished product will be small enough to fit in a single file and correct enough to run real applications.

## The Core Application Class

```python
# bare.py
import asyncio
import json
import re
import urllib.parse
import uuid
from dataclasses import dataclass, field
from typing import Any, Callable, Dict, List, Optional, Tuple


# ── Types ─────────────────────────────────────────────────────────────────────

Headers = List[Tuple[bytes, bytes]]
ASGIApp = Callable  # (scope, receive, send) -> None


# ── Request ───────────────────────────────────────────────────────────────────

class Request:
    def __init__(self, scope: dict, receive: Callable):
        self._scope = scope
        self._receive = receive
        self._body: Optional[bytes] = None

    @property
    def method(self) -> str:
        return self._scope["method"]

    @property
    def path(self) -> str:
        return self._scope["path"]

    @property
    def query_string(self) -> str:
        return self._scope.get("query_string", b"").decode("latin-1")

    @property
    def headers(self) -> Dict[str, str]:
        return {
            k.decode("latin-1"): v.decode("latin-1")
            for k, v in self._scope.get("headers", [])
        }

    def header(self, name: str, default: str = "") -> str:
        return self.headers.get(name.lower(), default)

    @property
    def content_type(self) -> str:
        return self.header("content-type")

    @property
    def path_params(self) -> Dict[str, str]:
        return self._scope.get("path_params", {})

    @property
    def query_params(self) -> Dict[str, List[str]]:
        return urllib.parse.parse_qs(self.query_string, keep_blank_values=True)

    def query(self, name: str, default: str = "") -> str:
        values = self.query_params.get(name, [])
        return values[0] if values else default

    async def body(self) -> bytes:
        if self._body is None:
            chunks = []
            while True:
                event = await self._receive()
                if event["type"] == "http.request":
                    chunks.append(event.get("body", b""))
                    if not event.get("more_body", False):
                        break
                elif event["type"] == "http.disconnect":
                    break
            self._body = b"".join(chunks)
        return self._body

    async def text(self) -> str:
        return (await self.body()).decode("utf-8")

    async def json(self) -> Any:
        return json.loads(await self.body())

    @property
    def app(self) -> "Bare":
        return self._scope["app"]

    def __repr__(self) -> str:
        return f"<Request {self.method} {self.path}>"


# ── Response ──────────────────────────────────────────────────────────────────

class Response:
    def __init__(
        self,
        body: Any = None,
        status: int = 200,
        headers: Optional[Dict[str, str]] = None,
        content_type: str = "text/plain; charset=utf-8",
    ):
        self.status = status
        self._headers = {"content-type": content_type}
        if headers:
            self._headers.update({k.lower(): v for k, v in headers.items()})

        if body is None:
            self._body = b""
        elif isinstance(body, bytes):
            self._body = body
        elif isinstance(body, str):
            self._body = body.encode("utf-8")
        else:
            self._body = str(body).encode("utf-8")

    def set_header(self, name: str, value: str) -> "Response":
        self._headers[name.lower()] = value
        return self

    def set_cookie(self, name: str, value: str, **attrs) -> "Response":
        cookie = f"{name}={value}"
        for attr, attr_val in attrs.items():
            cookie += f"; {attr.replace('_', '-')}={attr_val}"
        self._headers.setdefault("set-cookie", cookie)
        return self

    async def send(self, send: Callable) -> None:
        headers = list(self._headers.items())
        headers.append(("content-length", str(len(self._body))))

        raw_headers = [
            (k.encode("latin-1"), v.encode("latin-1"))
            for k, v in headers
        ]

        await send({
            "type": "http.response.start",
            "status": self.status,
            "headers": raw_headers,
        })
        await send({
            "type": "http.response.body",
            "body": self._body,
            "more_body": False,
        })


class JSONResponse(Response):
    def __init__(self, data: Any, status: int = 200, **kwargs):
        super().__init__(
            body=json.dumps(data, default=str),
            status=status,
            content_type="application/json",
            **kwargs,
        )


class HTMLResponse(Response):
    def __init__(self, html: str, status: int = 200, **kwargs):
        super().__init__(body=html, status=status,
                         content_type="text/html; charset=utf-8", **kwargs)


class RedirectResponse(Response):
    def __init__(self, location: str, status: int = 302):
        super().__init__(status=status, headers={"location": location})


# ── Routing ───────────────────────────────────────────────────────────────────

@dataclass
class Route:
    method: str
    pattern: re.Pattern
    handler: Callable
    param_names: List[str]


def compile_route(path: str) -> Tuple[re.Pattern, List[str]]:
    """Convert '/users/{id:int}' to a compiled regex and param names."""
    converters = {"str": r"[^/]+", "int": r"[0-9]+", "slug": r"[a-zA-Z0-9-]+"}
    param_names = []

    def replace(m):
        name, _, conv = m.group(1).partition(":")
        param_names.append(name)
        return f"(?P<{name}>{converters.get(conv or 'str', converters['str'])})"

    regex = "^" + re.sub(r"\{([^}]+)\}", replace, path) + "$"
    return re.compile(regex), param_names


# ── WebSocket ─────────────────────────────────────────────────────────────────

class WebSocket:
    def __init__(self, scope: dict, receive: Callable, send: Callable):
        self._scope = scope
        self._receive = receive
        self._send = send
        self.path_params: Dict[str, str] = scope.get("path_params", {})

    async def accept(self, subprotocol: str = None) -> None:
        event = await self._receive()
        assert event["type"] == "websocket.connect"
        await self._send({
            "type": "websocket.accept",
            "subprotocol": subprotocol,
        })

    async def receive_text(self) -> Optional[str]:
        event = await self._receive()
        if event["type"] == "websocket.receive":
            return event.get("text")
        return None  # disconnect

    async def receive_bytes(self) -> Optional[bytes]:
        event = await self._receive()
        if event["type"] == "websocket.receive":
            return event.get("bytes")
        return None

    async def receive_json(self) -> Any:
        text = await self.receive_text()
        return json.loads(text) if text is not None else None

    async def send_text(self, text: str) -> None:
        await self._send({"type": "websocket.send", "text": text})

    async def send_bytes(self, data: bytes) -> None:
        await self._send({"type": "websocket.send", "bytes": data})

    async def send_json(self, data: Any) -> None:
        await self.send_text(json.dumps(data, default=str))

    async def close(self, code: int = 1000) -> None:
        await self._send({"type": "websocket.close", "code": code})

    async def __aenter__(self) -> "WebSocket":
        await self.accept()
        return self

    async def __aexit__(self, *exc) -> None:
        await self.close()


# ── The Framework ─────────────────────────────────────────────────────────────

class Bare:
    def __init__(self):
        self._http_routes: List[Route] = []
        self._ws_routes: List[Route] = []
        self._startup_handlers: List[Callable] = []
        self._shutdown_handlers: List[Callable] = []
        self._middleware: List[Callable] = []
        self._built_app: Optional[ASGIApp] = None
        self.state: Dict[str, Any] = {}

    # ── Route registration ────────────────────────────────────────────────

    def route(self, path: str, methods: List[str] = None):
        """Decorator to register an HTTP route handler."""
        methods = [m.upper() for m in (methods or ["GET"])]

        def decorator(func: Callable) -> Callable:
            pattern, param_names = compile_route(path)
            for method in methods:
                self._http_routes.append(
                    Route(method, pattern, func, param_names)
                )
            return func
        return decorator

    def get(self, path: str):
        return self.route(path, ["GET"])

    def post(self, path: str):
        return self.route(path, ["POST"])

    def put(self, path: str):
        return self.route(path, ["PUT"])

    def patch(self, path: str):
        return self.route(path, ["PATCH"])

    def delete(self, path: str):
        return self.route(path, ["DELETE"])

    def websocket(self, path: str):
        """Decorator to register a WebSocket handler."""
        def decorator(func: Callable) -> Callable:
            pattern, param_names = compile_route(path)
            self._ws_routes.append(Route("WS", pattern, func, param_names))
            return func
        return decorator

    # ── Lifespan ──────────────────────────────────────────────────────────

    def on_startup(self, func: Callable) -> Callable:
        self._startup_handlers.append(func)
        return func

    def on_shutdown(self, func: Callable) -> Callable:
        self._shutdown_handlers.append(func)
        return func

    # ── Middleware ────────────────────────────────────────────────────────

    def add_middleware(self, middleware_class, **kwargs):
        self._middleware.append((middleware_class, kwargs))
        self._built_app = None  # Invalidate cache

    # ── ASGI interface ────────────────────────────────────────────────────

    async def __call__(self, scope, receive, send):
        if self._built_app is None:
            self._built_app = self._build_app()
        await self._built_app(scope, receive, send)

    def _build_app(self) -> ASGIApp:
        app = self._handle
        for mw_class, kwargs in reversed(self._middleware):
            app = mw_class(app, **kwargs)
        return app

    async def _handle(self, scope, receive, send):
        scope["app"] = self

        if scope["type"] == "lifespan":
            await self._handle_lifespan(receive, send)
        elif scope["type"] == "http":
            await self._handle_http(scope, receive, send)
        elif scope["type"] == "websocket":
            await self._handle_websocket(scope, receive, send)

    async def _handle_lifespan(self, receive, send):
        while True:
            event = await receive()
            if event["type"] == "lifespan.startup":
                try:
                    for handler in self._startup_handlers:
                        await handler() if asyncio.iscoroutinefunction(handler) else handler()
                    await send({"type": "lifespan.startup.complete"})
                except Exception as e:
                    await send({"type": "lifespan.startup.failed", "message": str(e)})
                    return
            elif event["type"] == "lifespan.shutdown":
                for handler in self._shutdown_handlers:
                    try:
                        await handler() if asyncio.iscoroutinefunction(handler) else handler()
                    except Exception:
                        pass
                await send({"type": "lifespan.shutdown.complete"})
                return

    async def _handle_http(self, scope, receive, send):
        method = scope["method"]
        path = scope["path"]

        matched = []
        for route in self._http_routes:
            m = route.pattern.match(path)
            if m:
                matched.append((route, m))

        if not matched:
            await JSONResponse({"error": "not found"}, 404).send(send)
            return

        for route, m in matched:
            if route.method == method:
                scope["path_params"] = m.groupdict()
                request = Request(scope, receive)
                try:
                    response = await route.handler(request)
                    if response is None:
                        response = Response()
                    if not isinstance(response, Response):
                        response = JSONResponse(response)
                    await response.send(send)
                except Exception as e:
                    await JSONResponse({"error": str(e)}, 500).send(send)
                return

        allowed = sorted({r.method for r, _ in matched})
        await JSONResponse(
            {"error": "method not allowed", "allowed": allowed},
            405,
            headers={"allow": ", ".join(allowed)},
        ).send(send)

    async def _handle_websocket(self, scope, receive, send):
        path = scope["path"]
        for route in self._ws_routes:
            m = route.pattern.match(path)
            if m:
                scope["path_params"] = m.groupdict()
                ws = WebSocket(scope, receive, send)
                try:
                    await route.handler(ws)
                except Exception:
                    await send({"type": "websocket.close", "code": 1011})
                return

        # No matching WebSocket route — reject
        event = await receive()
        if event["type"] == "websocket.connect":
            await send({"type": "websocket.close", "code": 4004})

    def run(self, host: str = "127.0.0.1", port: int = 8000, **kwargs):
        """Convenience method to run with uvicorn."""
        import uvicorn
        uvicorn.run(self, host=host, port=port, **kwargs)
```

## Using the Framework

```python
# example_app.py
from bare import Bare, JSONResponse, HTMLResponse, WebSocket

app = Bare()

# In-memory store
tasks = {}


@app.on_startup
async def startup():
    print("App started. Ready to serve.")
    # Connect to database, load config, etc.


@app.on_shutdown
async def shutdown():
    print("App stopping. Cleaning up.")


@app.get("/")
async def index(request):
    return HTMLResponse("<h1>Bare Framework</h1><p>It's just callables.</p>")


@app.get("/tasks")
async def list_tasks(request):
    done = request.query("done")
    result = list(tasks.values())
    if done == "true":
        result = [t for t in result if t["done"]]
    elif done == "false":
        result = [t for t in result if not t["done"]]
    return JSONResponse(result)


@app.post("/tasks")
async def create_task(request):
    data = await request.json()
    if "title" not in data:
        return JSONResponse({"error": "title is required"}, 400)
    task = {
        "id": str(__import__("uuid").uuid4()),
        "title": data["title"],
        "done": False,
    }
    tasks[task["id"]] = task
    return JSONResponse(task, 201)


@app.get("/tasks/{task_id}")
async def get_task(request):
    task_id = request.path_params["task_id"]
    task = tasks.get(task_id)
    if not task:
        return JSONResponse({"error": "not found"}, 404)
    return JSONResponse(task)


@app.patch("/tasks/{task_id}")
async def update_task(request):
    task_id = request.path_params["task_id"]
    task = tasks.get(task_id)
    if not task:
        return JSONResponse({"error": "not found"}, 404)
    data = await request.json()
    if "done" in data:
        task["done"] = bool(data["done"])
    if "title" in data:
        task["title"] = str(data["title"])
    return JSONResponse(task)


@app.delete("/tasks/{task_id}")
async def delete_task(request):
    task_id = request.path_params["task_id"]
    if task_id not in tasks:
        return JSONResponse({"error": "not found"}, 404)
    return JSONResponse(tasks.pop(task_id))


@app.websocket("/ws")
async def chat(ws: WebSocket):
    async with ws:  # accept on enter, close on exit
        await ws.send_text("Welcome! Type messages to see them echoed.")
        while True:
            message = await ws.receive_text()
            if message is None:  # disconnect
                break
            await ws.send_json({
                "type": "echo",
                "original": message,
                "upper": message.upper(),
            })


if __name__ == "__main__":
    app.run(reload=True)
```

```bash
python example_app.py

# Test it
curl http://localhost:8000/
curl -X POST http://localhost:8000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Built with bare"}'
```

## Adding Middleware to Our Framework

```python
import time
import sys


class LoggingMiddleware:
    def __init__(self, app):
        self.app = app

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return

        started = time.monotonic()
        status_code = [None]

        async def capturing_send(event):
            if event["type"] == "http.response.start":
                status_code[0] = event["status"]
            await send(event)

        await self.app(scope, receive, capturing_send)

        elapsed = (time.monotonic() - started) * 1000
        print(
            f"{scope['method']} {scope['path']} → "
            f"{status_code[0]} ({elapsed:.1f}ms)",
            file=sys.stderr,
        )


app.add_middleware(LoggingMiddleware)
```

## Testing Our Framework

```python
import asyncio
import json


async def test_bare_framework():
    from example_app import app

    # Build a minimal test harness
    async def request(method, path, body=b"", headers=None):
        scope = {
            "type": "http",
            "asgi": {"version": "3.0"},
            "method": method,
            "path": path,
            "query_string": b"",
            "headers": headers or (
                [(b"content-type", b"application/json")] if body else []
            ),
            "server": ("testserver", 8000),
        }
        events = [{"type": "http.request", "body": body, "more_body": False}]
        idx = [0]

        async def receive():
            e = events[idx[0]] if idx[0] < len(events) else {"type": "http.disconnect"}
            idx[0] += 1
            return e

        response_events = []
        async def send(event):
            response_events.append(event)

        await app(scope, receive, send)
        start = next(e for e in response_events if e["type"] == "http.response.start")
        body_data = b"".join(
            e.get("body", b"") for e in response_events
            if e["type"] == "http.response.body"
        )
        return start["status"], json.loads(body_data) if body_data else None

    # Test cases
    status, data = await request("GET", "/tasks")
    assert status == 200, f"Expected 200, got {status}"

    status, data = await request(
        "POST", "/tasks",
        body=json.dumps({"title": "Framework test"}).encode()
    )
    assert status == 201
    task_id = data["id"]

    status, data = await request("GET", f"/tasks/{task_id}")
    assert status == 200
    assert data["title"] == "Framework test"

    status, data = await request("GET", "/tasks/nonexistent")
    assert status == 404

    print("Framework tests passed.")


asyncio.run(test_bare_framework())
```

## What We Left Out

`bare` is small by design. Things a production framework would add:

**Static file serving** — map a URL prefix to a directory of files. Not hard, but involves MIME type detection and `If-Modified-Since` handling.

**Template rendering** — integrate with Jinja2 or similar. The `Response` class would gain a `from_template(name, context)` factory.

**Form parsing** — `multipart/form-data` for file uploads. The spec is in RFC 7578 and it's tedious.

**Cookie handling** — parsing the `Cookie` header and setting `Set-Cookie` on responses.

**Session middleware** — signed cookies or server-side sessions backed by Redis.

**Error pages** — catch exceptions in handlers and return friendly 500 pages rather than bare JSON.

**OpenAPI generation** — FastAPI's big feature: inspect route handlers' type annotations and build an OpenAPI schema automatically.

Each of these is a well-understood problem with known solutions. The framework doesn't do anything you couldn't do yourself — it just packages the common solutions conveniently.

## The Point

You've now built a framework. It's small, but it's real — the `tasks` app runs on it, the WebSocket handler works, middleware composes correctly. You understand every line of it because you wrote every line of it.

When you use FastAPI or Starlette next week, you'll recognize the patterns: the route decorators are building a route table. The `Request` object is wrapping the scope. The `Response` is sending events to `send`. The `lifespan` context manager is handling startup/shutdown events.

The framework isn't doing anything mysterious. It's doing exactly what you'd do if you wrote it yourself — which you just did.

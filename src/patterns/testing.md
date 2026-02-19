# Testing WSGI and ASGI Apps Without a Framework

Your application is a callable. Test it like one.

This is the payoff for understanding the interface directly: testing becomes a matter of calling your app with the right arguments and inspecting the result. No special test client required — though they're convenient, and we'll look at those too.

## Testing WSGI Applications

A WSGI application takes `environ` and `start_response`. To test it, provide both:

```python
import io
import json
from typing import Any, Dict, List, Optional, Tuple


def make_environ(
    method: str = "GET",
    path: str = "/",
    query_string: str = "",
    body: bytes = b"",
    content_type: str = "",
    headers: Optional[Dict[str, str]] = None,
    environ_overrides: Optional[Dict] = None,
) -> dict:
    """Build a WSGI environ dict for testing."""
    environ = {
        "REQUEST_METHOD": method.upper(),
        "PATH_INFO": path,
        "QUERY_STRING": query_string,
        "CONTENT_TYPE": content_type,
        "CONTENT_LENGTH": str(len(body)) if body else "",
        "SERVER_NAME": "testserver",
        "SERVER_PORT": "80",
        "HTTP_HOST": "testserver",
        "wsgi.input": io.BytesIO(body),
        "wsgi.errors": io.StringIO(),
        "wsgi.url_scheme": "http",
        "wsgi.version": (1, 0),
        "wsgi.multithread": False,
        "wsgi.multiprocess": False,
        "wsgi.run_once": False,
        "GATEWAY_INTERFACE": "CGI/1.1",
        "SERVER_PROTOCOL": "HTTP/1.1",
    }

    # Add custom headers
    if headers:
        for name, value in headers.items():
            key = "HTTP_" + name.upper().replace("-", "_")
            environ[key] = value

    if environ_overrides:
        environ.update(environ_overrides)

    return environ


class WSGITestResponse:
    """Holds the result of calling a WSGI application."""

    def __init__(self, status: str, headers: List[Tuple[str, str]], body: bytes):
        self.status = status
        self.status_code = int(status.split(" ")[0])
        self.headers = dict(headers)
        self.body = body

    @property
    def text(self) -> str:
        return self.body.decode("utf-8")

    def json(self) -> Any:
        return json.loads(self.body)

    def __repr__(self) -> str:
        return f"<WSGITestResponse {self.status}>"


def call_wsgi(app, environ: dict) -> WSGITestResponse:
    """Call a WSGI app and return a response object."""
    response_parts = []

    def start_response(status, headers, exc_info=None):
        if exc_info:
            raise exc_info[1].with_traceback(exc_info[2])
        response_parts.append((status, headers))

    result = app(environ, start_response)
    try:
        body = b"".join(result)
    finally:
        if hasattr(result, "close"):
            result.close()

    status, headers = response_parts[0]
    return WSGITestResponse(status, headers, body)


class WSGITestClient:
    """A test client for WSGI applications."""

    def __init__(self, app):
        self.app = app

    def get(self, path: str, **kwargs) -> WSGITestResponse:
        return self.request("GET", path, **kwargs)

    def post(self, path: str, **kwargs) -> WSGITestResponse:
        return self.request("POST", path, **kwargs)

    def put(self, path: str, **kwargs) -> WSGITestResponse:
        return self.request("PUT", path, **kwargs)

    def delete(self, path: str, **kwargs) -> WSGITestResponse:
        return self.request("DELETE", path, **kwargs)

    def request(
        self,
        method: str,
        path: str,
        body: bytes = b"",
        json: Any = None,
        headers: Optional[Dict[str, str]] = None,
        query_string: str = "",
    ) -> WSGITestResponse:
        content_type = ""
        if json is not None:
            import json as json_module
            body = json_module.dumps(json).encode("utf-8")
            content_type = "application/json"

        environ = make_environ(
            method=method,
            path=path,
            query_string=query_string,
            body=body,
            content_type=content_type,
            headers=headers,
        )
        return call_wsgi(self.app, environ)
```

## Writing WSGI Tests

```python
# test_wsgi_tasks.py
from tasks_app import application  # The tasks app from chapter 5


client = WSGITestClient(application)


def test_empty_task_list():
    response = client.get("/tasks")
    assert response.status_code == 200
    assert response.json() == []


def test_create_task():
    response = client.post("/tasks", json={"title": "Write tests"})
    assert response.status_code == 201
    data = response.json()
    assert data["title"] == "Write tests"
    assert data["done"] is False
    assert "id" in data
    return data["id"]


def test_get_task():
    # Create a task first
    task_id = test_create_task()

    response = client.get(f"/tasks/{task_id}")
    assert response.status_code == 200
    assert response.json()["id"] == task_id


def test_task_not_found():
    response = client.get("/tasks/does-not-exist")
    assert response.status_code == 404


def test_delete_task():
    task_id = test_create_task()

    response = client.delete(f"/tasks/{task_id}")
    assert response.status_code == 200

    response = client.get(f"/tasks/{task_id}")
    assert response.status_code == 404


def test_missing_title():
    response = client.post("/tasks", json={"description": "no title here"})
    assert response.status_code == 400
    assert "title" in response.json()["error"]


def test_wrong_method():
    response = client.request("PATCH", "/tasks")
    assert response.status_code == 405


if __name__ == "__main__":
    test_empty_task_list()
    test_create_task()
    test_get_task()
    test_task_not_found()
    test_delete_task()
    test_missing_title()
    test_wrong_method()
    print("All WSGI tests passed.")
```

Run with pytest (`pip install pytest`) or directly:

```bash
pytest test_wsgi_tasks.py -v
# or
python test_wsgi_tasks.py
```

## Testing ASGI Applications

ASGI apps are async, so tests need to be async too. Python's `asyncio` makes this straightforward:

```python
import asyncio
import io
import json
from typing import Any, Dict, List, Optional


def make_scope(
    method: str = "GET",
    path: str = "/",
    query_string: bytes = b"",
    headers: Optional[List] = None,
    scope_type: str = "http",
) -> dict:
    """Build an ASGI HTTP scope dict for testing."""
    return {
        "type": scope_type,
        "asgi": {"version": "3.0"},
        "http_version": "1.1",
        "method": method.upper(),
        "path": path,
        "raw_path": path.encode("latin-1"),
        "query_string": query_string,
        "root_path": "",
        "scheme": "http",
        "headers": headers or [],
        "server": ("testserver", 80),
        "client": ("127.0.0.1", 12345),
    }


class ASGITestResponse:
    """Holds the result of calling an ASGI application."""

    def __init__(self, status: int, headers: List, body: bytes):
        self.status_code = status
        self._headers = headers
        self.headers = {
            k.decode("latin-1"): v.decode("latin-1")
            for k, v in headers
        }
        self.body = body

    @property
    def text(self) -> str:
        return self.body.decode("utf-8")

    def json(self) -> Any:
        return json.loads(self.body)

    def __repr__(self) -> str:
        return f"<ASGITestResponse {self.status_code}>"


async def call_asgi(
    app,
    scope: dict,
    body: bytes = b"",
) -> ASGITestResponse:
    """Call an ASGI app with an HTTP scope and return a response."""
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

    start = next(e for e in response_events if e["type"] == "http.response.start")
    body_chunks = [
        e.get("body", b"")
        for e in response_events
        if e["type"] == "http.response.body"
    ]

    return ASGITestResponse(
        status=start["status"],
        headers=start.get("headers", []),
        body=b"".join(body_chunks),
    )


class ASGITestClient:
    """A test client for ASGI applications."""

    def __init__(self, app):
        self.app = app
        self._started = False

    async def _ensure_started(self):
        """Send lifespan startup if not already done."""
        if self._started:
            return
        self._started = True

        scope = {"type": "lifespan", "asgi": {"version": "3.0"}}
        events = asyncio.Queue()
        startup_done = asyncio.Event()

        await events.put({"type": "lifespan.startup"})

        async def receive():
            return await events.get()

        async def send(event):
            if event["type"] == "lifespan.startup.complete":
                startup_done.set()

        asyncio.create_task(self.app(scope, receive, send))
        try:
            await asyncio.wait_for(startup_done.wait(), timeout=5.0)
        except asyncio.TimeoutError:
            pass  # App may not handle lifespan

    async def get(self, path: str, **kwargs) -> ASGITestResponse:
        return await self.request("GET", path, **kwargs)

    async def post(self, path: str, **kwargs) -> ASGITestResponse:
        return await self.request("POST", path, **kwargs)

    async def delete(self, path: str, **kwargs) -> ASGITestResponse:
        return await self.request("DELETE", path, **kwargs)

    async def patch(self, path: str, **kwargs) -> ASGITestResponse:
        return await self.request("PATCH", path, **kwargs)

    async def request(
        self,
        method: str,
        path: str,
        body: bytes = b"",
        json: Any = None,
        headers: Optional[Dict[str, str]] = None,
        query_string: bytes = b"",
    ) -> ASGITestResponse:
        await self._ensure_started()

        content_type = ""
        if json is not None:
            import json as json_module
            body = json_module.dumps(json).encode("utf-8")
            content_type = "application/json"

        raw_headers = []
        if content_type:
            raw_headers.append((b"content-type", content_type.encode()))
        if body:
            raw_headers.append((b"content-length", str(len(body)).encode()))
        if headers:
            for name, value in headers.items():
                raw_headers.append((
                    name.lower().encode("latin-1"),
                    value.encode("latin-1"),
                ))

        scope = make_scope(
            method=method,
            path=path,
            query_string=query_string,
            headers=raw_headers,
        )

        return await call_asgi(self.app, scope, body)
```

## Writing ASGI Tests

```python
# test_asgi_tasks.py
import asyncio
from asgi_tasks import application  # From ASGI chapter


client = ASGITestClient(application)


async def test_list_tasks_empty():
    response = await client.get("/tasks")
    assert response.status_code == 200
    assert response.json() == []


async def test_create_and_get_task():
    # Create
    response = await client.post("/tasks", json={"title": "Async task"})
    assert response.status_code == 201
    task = response.json()
    assert task["title"] == "Async task"
    task_id = task["id"]

    # Get
    response = await client.get(f"/tasks/{task_id}")
    assert response.status_code == 200
    assert response.json()["id"] == task_id


async def test_update_task():
    # Create
    response = await client.post("/tasks", json={"title": "To update"})
    task_id = response.json()["id"]

    # Update
    response = await client.patch(f"/tasks/{task_id}", json={"done": True})
    assert response.status_code == 200
    assert response.json()["done"] is True


async def test_content_type_required():
    response = await client.request(
        "POST", "/tasks",
        body=b'{"title": "oops"}',
        # No content-type header
    )
    assert response.status_code == 415


async def run_all_tests():
    await test_list_tasks_empty()
    print("✓ list tasks empty")
    await test_create_and_get_task()
    print("✓ create and get task")
    await test_update_task()
    print("✓ update task")
    await test_content_type_required()
    print("✓ content type required")
    print("All ASGI tests passed.")


if __name__ == "__main__":
    asyncio.run(run_all_tests())
```

## Using pytest-asyncio

For proper async test suites, use `pytest-asyncio`:

```bash
pip install pytest pytest-asyncio
```

```python
# conftest.py
import pytest
from asgi_tasks import application
from test_helpers import ASGITestClient


@pytest.fixture
def client():
    return ASGITestClient(application)
```

```python
# test_asgi_tasks.py
import pytest


@pytest.mark.asyncio
async def test_create_task(client):
    response = await client.post("/tasks", json={"title": "pytest task"})
    assert response.status_code == 201
    assert response.json()["title"] == "pytest task"


@pytest.mark.asyncio
async def test_delete_task(client):
    create_response = await client.post("/tasks", json={"title": "to delete"})
    task_id = create_response.json()["id"]

    delete_response = await client.delete(f"/tasks/{task_id}")
    assert delete_response.status_code == 200

    get_response = await client.get(f"/tasks/{task_id}")
    assert get_response.status_code == 404
```

```bash
pytest test_asgi_tasks.py -v
```

## Testing WebSocket Handlers

WebSocket testing requires simulating the connect/message/disconnect lifecycle:

```python
async def test_websocket_echo():
    from asgi_websockets import application  # WebSocket echo app

    scope = {
        "type": "websocket",
        "asgi": {"version": "3.0"},
        "path": "/ws",
        "query_string": b"",
        "headers": [],
        "server": ("testserver", 8000),
        "client": ("127.0.0.1", 9999),
        "subprotocols": [],
    }

    # Queue of events the app will receive
    incoming = asyncio.Queue()
    outgoing = []

    async def receive():
        return await incoming.get()

    async def send(event):
        outgoing.append(event)

    # Start the handler
    handler_task = asyncio.create_task(application(scope, receive, send))

    # Simulate WebSocket connect
    await incoming.put({"type": "websocket.connect"})
    await asyncio.sleep(0)  # Let the handler process it

    # Check accept was sent
    assert outgoing[-1]["type"] == "websocket.accept"

    # Send a message
    await incoming.put({"type": "websocket.receive", "text": "hello"})
    await asyncio.sleep(0)

    # Check echo
    echo_event = outgoing[-1]
    assert echo_event["type"] == "websocket.send"
    assert "hello" in echo_event.get("text", "")

    # Disconnect
    await incoming.put({"type": "websocket.disconnect", "code": 1000})
    await handler_task


asyncio.run(test_websocket_echo())
```

## Testing Middleware

Test middleware by composing it with a simple app:

```python
async def test_timing_middleware():
    import time

    timing_logs = []

    class CapturingTimingMiddleware(TimingMiddleware):
        async def __call__(self, scope, receive, send):
            # Override to capture instead of print
            started = time.monotonic()
            status_holder = []

            async def capturing_send(event):
                if event["type"] == "http.response.start":
                    status_holder.append(event["status"])
                elif event["type"] == "http.response.body":
                    if not event.get("more_body", False):
                        elapsed = (time.monotonic() - started) * 1000
                        timing_logs.append({
                            "path": scope.get("path"),
                            "status": status_holder[0] if status_holder else None,
                            "elapsed_ms": elapsed,
                        })
                await send(event)

            await self.app(scope, receive, capturing_send)

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

    app = CapturingTimingMiddleware(simple_app)
    test_client = ASGITestClient(app)

    await test_client.get("/test-path")

    assert len(timing_logs) == 1
    assert timing_logs[0]["path"] == "/test-path"
    assert timing_logs[0]["status"] == 200
    assert timing_logs[0]["elapsed_ms"] >= 0
    print("Timing middleware test passed")


asyncio.run(test_timing_middleware())
```

## The `httpx` Transport Approach

The `httpx` library (an async HTTP client) has a built-in ASGI transport that lets you test ASGI apps with a full HTTP client interface:

```bash
pip install httpx
```

```python
import httpx


async def test_with_httpx():
    from asgi_tasks import application

    async with httpx.AsyncClient(
        transport=httpx.ASGITransport(app=application),
        base_url="http://testserver",
    ) as client:
        response = await client.post(
            "/tasks",
            json={"title": "httpx task"},
        )
        assert response.status_code == 201
        task = response.json()
        assert task["title"] == "httpx task"

        # httpx handles cookies, redirects, etc. automatically
        response = await client.get(f"/tasks/{task['id']}")
        assert response.status_code == 200


asyncio.run(test_with_httpx())
```

`httpx.ASGITransport` calls your ASGI app directly, without a network socket. It's the same mechanism we built manually — it constructs scopes, calls `receive` with request body events, and collects the response from `send` events — but packaged as a transport that the full `httpx` client can use.

This gives you all of httpx's conveniences (cookie handling, redirects, JSON serialization) while testing without a server.

## The Principle

Testing WSGI/ASGI apps without a framework is fast (no server startup), correct (you're calling the exact same code path as production), and instructive (you understand exactly what's happening).

The test client libraries — pytest-django, Starlette's TestClient, FastAPI's TestClient, httpx — are all doing exactly what we've done here: constructing the right environ/scope, calling your app, and wrapping the result.

When a test fails mysteriously, knowing that your test client is just calling `app(scope, receive, send)` gives you a concrete place to start debugging.

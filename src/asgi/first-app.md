# Your First ASGI App

We built a tasks API in the WSGI section. Let's rebuild it as ASGI — same functionality, but async, with better structure. By the end you'll have a working JSON API that demonstrates the full ASGI request/response cycle.

## The Foundation

First, let's build the utilities we'll need. In ASGI, headers are bytes and parsing happens more explicitly:

```python
import json
import uuid
from typing import Any, Dict, List, Optional, Tuple


# Type aliases for clarity
Headers = List[Tuple[bytes, bytes]]


def get_header(scope_headers: Headers, name: str) -> Optional[str]:
    """Get a header value from ASGI scope headers (case-insensitive)."""
    name_bytes = name.lower().encode("latin-1")
    for key, value in scope_headers:
        if key.lower() == name_bytes:
            return value.decode("latin-1")
    return None


def make_headers(*pairs: Tuple[str, str]) -> Headers:
    """Build ASGI headers from string tuples."""
    return [
        (name.lower().encode("latin-1"), value.encode("latin-1"))
        for name, value in pairs
    ]


async def read_body(receive) -> bytes:
    """Read the full request body, handling chunked delivery."""
    body = b""
    while True:
        event = await receive()
        if event["type"] == "http.request":
            body += event.get("body", b"")
            if not event.get("more_body", False):
                break
        elif event["type"] == "http.disconnect":
            break
    return body


async def send_response(send, status: int, body: bytes, headers: Headers) -> None:
    """Send a complete HTTP response."""
    # Always include Content-Length
    all_headers = list(headers) + [
        (b"content-length", str(len(body)).encode())
    ]
    await send({
        "type": "http.response.start",
        "status": status,
        "headers": all_headers,
    })
    await send({
        "type": "http.response.body",
        "body": body,
        "more_body": False,
    })


async def send_json(send, data: Any, status: int = 200) -> None:
    """Send a JSON response."""
    body = json.dumps(data, indent=2, default=str).encode("utf-8")
    await send_response(
        send,
        status,
        body,
        make_headers("content-type", "application/json"),
    )
```

## The Application

```python
# In-memory store
tasks: Dict[str, Dict] = {}


async def application(scope, receive, send):
    """Main ASGI application."""
    if scope["type"] == "lifespan":
        await handle_lifespan(receive, send)
        return

    if scope["type"] != "http":
        return

    await handle_http(scope, receive, send)


async def handle_lifespan(receive, send):
    while True:
        event = await receive()
        if event["type"] == "lifespan.startup":
            # Initialize resources here
            print("Application starting up")
            await send({"type": "lifespan.startup.complete"})
        elif event["type"] == "lifespan.shutdown":
            # Clean up resources here
            print("Application shutting down")
            await send({"type": "lifespan.shutdown.complete"})
            break


async def handle_http(scope, receive, send):
    method = scope["method"]
    path = scope["path"]

    # Route dispatch
    if path == "/tasks":
        if method == "GET":
            await list_tasks(scope, receive, send)
        elif method == "POST":
            await create_task(scope, receive, send)
        else:
            await send_json(send, {"error": "method not allowed"}, 405)

    elif path.startswith("/tasks/"):
        task_id = path[len("/tasks/"):]
        if not task_id:
            await send_json(send, {"error": "not found"}, 404)
            return

        if method == "GET":
            await get_task(task_id, scope, receive, send)
        elif method == "DELETE":
            await delete_task(task_id, scope, receive, send)
        elif method == "PATCH":
            await update_task(task_id, scope, receive, send)
        else:
            await send_json(send, {"error": "method not allowed"}, 405)

    else:
        await send_json(send, {"error": "not found"}, 404)


async def list_tasks(scope, receive, send):
    # Consume the body even if we don't use it (good practice)
    await read_body(receive)
    await send_json(send, list(tasks.values()))


async def create_task(scope, receive, send):
    content_type = get_header(scope["headers"], "content-type") or ""
    if "application/json" not in content_type:
        await send_json(send, {"error": "Content-Type must be application/json"}, 415)
        return

    body = await read_body(receive)
    try:
        data = json.loads(body)
    except (json.JSONDecodeError, UnicodeDecodeError):
        await send_json(send, {"error": "invalid JSON"}, 400)
        return

    if "title" not in data:
        await send_json(send, {"error": "title is required"}, 400)
        return

    task = {
        "id": str(uuid.uuid4()),
        "title": str(data["title"]),
        "done": bool(data.get("done", False)),
    }
    tasks[task["id"]] = task
    await send_json(send, task, 201)


async def get_task(task_id: str, scope, receive, send):
    await read_body(receive)
    task = tasks.get(task_id)
    if task is None:
        await send_json(send, {"error": "not found"}, 404)
        return
    await send_json(send, task)


async def delete_task(task_id: str, scope, receive, send):
    await read_body(receive)
    if task_id not in tasks:
        await send_json(send, {"error": "not found"}, 404)
        return
    deleted = tasks.pop(task_id)
    await send_json(send, deleted)


async def update_task(task_id: str, scope, receive, send):
    task = tasks.get(task_id)
    if task is None:
        await send_json(send, {"error": "not found"}, 404)
        return

    body = await read_body(receive)
    try:
        data = json.loads(body)
    except (json.JSONDecodeError, UnicodeDecodeError):
        await send_json(send, {"error": "invalid JSON"}, 400)
        return

    if "done" in data:
        task["done"] = bool(data["done"])
    if "title" in data:
        task["title"] = str(data["title"])

    await send_json(send, task)
```

Save as `asgi_tasks.py`, run with uvicorn:

```bash
pip install uvicorn
uvicorn asgi_tasks:application --reload
```

Test it:

```bash
# Create a task
curl -X POST http://localhost:8000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Learn ASGI"}'

# List tasks
curl http://localhost:8000/tasks

# Update a task (replace the ID with one from your previous response)
curl -X PATCH http://localhost:8000/tasks/YOUR-ID \
  -H "Content-Type: application/json" \
  -d '{"done": true}'

# Delete a task
curl -X DELETE http://localhost:8000/tasks/YOUR-ID
```

## Streaming Responses

One thing ASGI handles well that WSGI struggles with: streaming large responses. Instead of buffering everything in memory, send body chunks as they're produced:

```python
import asyncio


async def streaming_app(scope, receive, send):
    """Send a large response in chunks."""
    if scope["type"] != "http":
        return

    await read_body(receive)  # Consume request body

    # Start the response — no Content-Length for streaming
    await send({
        "type": "http.response.start",
        "status": 200,
        "headers": [
            (b"content-type", b"text/plain"),
            (b"transfer-encoding", b"chunked"),
        ],
    })

    # Send data in chunks
    for i in range(10):
        await asyncio.sleep(0.1)  # Simulate work
        chunk = f"Line {i}: some data\n".encode()
        await send({
            "type": "http.response.body",
            "body": chunk,
            "more_body": True,  # More is coming
        })

    # Final empty body closes the stream
    await send({
        "type": "http.response.body",
        "body": b"",
        "more_body": False,
    })
```

```bash
curl -N http://localhost:8000/  # -N = no buffering, shows chunks as they arrive
```

You'll see lines appear one at a time, 100ms apart. In WSGI, you'd have to return a generator and cross your fingers that Gunicorn didn't buffer it. In ASGI, the streaming model is first-class.

## Detecting Client Disconnects

ASGI lets you detect when a client disconnects mid-request. Useful for canceling expensive work:

```python
import asyncio


async def long_running_app(scope, receive, send):
    if scope["type"] != "http":
        return

    await read_body(receive)

    # Run work and listen for disconnect concurrently
    disconnect_task = asyncio.ensure_future(wait_for_disconnect(receive))
    work_task = asyncio.ensure_future(do_expensive_work())

    done, pending = await asyncio.wait(
        [disconnect_task, work_task],
        return_when=asyncio.FIRST_COMPLETED,
    )

    for task in pending:
        task.cancel()

    if disconnect_task in done:
        # Client disconnected — don't bother sending a response
        return

    result = work_task.result()
    await send_json(send, result)


async def wait_for_disconnect(receive) -> None:
    """Await the http.disconnect event."""
    while True:
        event = await receive()
        if event["type"] == "http.disconnect":
            return


async def do_expensive_work() -> dict:
    await asyncio.sleep(5)  # Simulate 5 seconds of work
    return {"result": "done"}
```

In WSGI, you can't do this — you have no way to detect a client disconnect mid-processing. Either you do the work and try to send, or you set a timeout and hope. ASGI gives you a proper event for it.

## Query Parameters

ASGI passes `query_string` as bytes. Parse it:

```python
import urllib.parse


def parse_query(scope) -> Dict[str, List[str]]:
    """Parse query string from ASGI scope."""
    qs = scope.get("query_string", b"").decode("latin-1")
    return urllib.parse.parse_qs(qs, keep_blank_values=True)


# In a handler:
async def search_tasks(scope, receive, send):
    await read_body(receive)
    params = parse_query(scope)
    query = params.get("q", [""])[0]
    done_filter = params.get("done", [None])[0]

    results = list(tasks.values())
    if query:
        results = [t for t in results if query.lower() in t["title"].lower()]
    if done_filter is not None:
        done = done_filter.lower() == "true"
        results = [t for t in results if t["done"] == done]

    await send_json(send, results)
```

## The `__main__` Runner

For development, uvicorn can be run programmatically:

```python
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(
        "asgi_tasks:application",
        host="127.0.0.1",
        port=8000,
        reload=True,  # Auto-reload on file changes
        log_level="info",
    )
```

Or as a module:

```bash
python -m uvicorn asgi_tasks:application --reload
```

## What This Reveals

Looking at the application we just built, notice what's verbose:
- The routing is still manual `if/elif`
- Reading the body requires the `read_body` helper
- Header access requires `get_header`
- Every handler needs to `await read_body` even if it doesn't use the body

This is exactly what Starlette (and by extension FastAPI) solves. Starlette's `Request` object wraps the ASGI scope with `request.method`, `request.url`, `request.headers`, `await request.body()`, `await request.json()`. Starlette's routing matches paths and dispatches to async view functions. Starlette's `Response` classes build the correct `http.response.start` and `http.response.body` events.

FastAPI adds type-annotated dependency injection on top of Starlette.

But now you know what they're building on. Every Starlette route handler ultimately calls `await send({"type": "http.response.start", ...})`. Every FastAPI request processes an ASGI scope dict. The wrappers are real, the underlying reality is what we've been working with.

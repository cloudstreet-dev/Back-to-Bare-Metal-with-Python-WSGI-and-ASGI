# The ASGI Spec (scope, receive, send — that's literally it)

The ASGI specification lives at [asgi.readthedocs.io](https://asgi.readthedocs.io). Like WSGI, the actual interface is simpler than the documentation makes it sound. Unlike WSGI, there are three connection types to understand: HTTP, WebSocket, and Lifespan.

Let's read the spec.

## The Interface

An ASGI application is an async callable with this signature:

```python
async def application(scope: dict, receive: callable, send: callable) -> None:
    ...
```

Three arguments:
- **`scope`**: a dict describing the connection (like `environ`, but for the connection type)
- **`receive`**: an async callable — call it to receive the next event from the client
- **`send`**: an async callable — call it to send an event to the client

No return value. All communication happens through `receive` and `send`.

## The `scope` Dictionary

`scope` contains connection metadata. The most important key is `type`, which tells you what kind of connection you're handling.

### HTTP scope

```python
scope = {
    "type": "http",
    "asgi": {"version": "3.0"},
    "http_version": "1.1",        # or "2"
    "method": "GET",              # uppercase
    "path": "/users/42",
    "raw_path": b"/users/42",
    "query_string": b"active=true",
    "root_path": "",
    "scheme": "http",             # or "https"
    "headers": [                  # list of (name, value) byte-string tuples
        (b"host", b"example.com"),
        (b"accept", b"application/json"),
        (b"content-type", b"application/json"),
        (b"content-length", b"42"),
    ],
    "server": ("127.0.0.1", 8000),  # (host, port) tuple
    "client": ("127.0.0.1", 54321), # client address
}
```

Notice: `headers` are bytes, not strings. ASGI works closer to the wire than WSGI — the header names and values are byte strings, and frameworks convert them to strings when they wrap the scope.

### WebSocket scope

```python
scope = {
    "type": "websocket",
    "asgi": {"version": "3.0"},
    "path": "/ws/chat",
    "query_string": b"room=general",
    "headers": [...],  # same format as HTTP
    "server": ("127.0.0.1", 8000),
    "client": ("127.0.0.1", 54322),
    "subprotocols": [],  # requested WebSocket subprotocols
}
```

### Lifespan scope

```python
scope = {
    "type": "lifespan",
    "asgi": {"version": "3.0"},
}
```

We'll cover lifespan in detail in its own chapter.

## The Events

`receive` and `send` deal in *events* — dicts with a `type` key.

### HTTP events

**Received from client:**

```python
# http.request — the body of the HTTP request
{
    "type": "http.request",
    "body": b"...",          # bytes (possibly empty)
    "more_body": False,      # True if more chunks are coming
}

# http.disconnect — client disconnected
{
    "type": "http.disconnect",
}
```

**Sent to client:**

```python
# http.response.start — sends status and headers
# Must be sent before http.response.body
{
    "type": "http.response.start",
    "status": 200,           # integer, not string
    "headers": [
        (b"content-type", b"text/plain"),
        (b"content-length", b"13"),
    ],
}

# http.response.body — sends body data
{
    "type": "http.response.body",
    "body": b"Hello, world!",
    "more_body": False,      # True = more chunks coming; False = done
}
```

### WebSocket events

**Received:**

```python
# websocket.connect — client initiated WebSocket handshake
{"type": "websocket.connect"}

# websocket.receive — client sent a message
{
    "type": "websocket.receive",
    "bytes": None,           # bytes message (or None)
    "text": "hello",         # text message (or None)
}

# websocket.disconnect — client disconnected
{
    "type": "websocket.disconnect",
    "code": 1000,            # WebSocket close code
}
```

**Sent:**

```python
# websocket.accept — accept the WebSocket handshake
{
    "type": "websocket.accept",
    "subprotocol": None,     # optional agreed subprotocol
    "headers": [],           # extra headers in the handshake response
}

# websocket.send — send a message to the client
{
    "type": "websocket.send",
    "bytes": None,           # bytes message (or None)
    "text": "hello",         # text message (or None)
}

# websocket.close — close the connection
{
    "type": "websocket.close",
    "code": 1000,            # WebSocket close code
}
```

## The Simplest Possible ASGI App

```python
async def application(scope, receive, send):
    """A complete, working ASGI application."""
    if scope["type"] != "http":
        return  # Ignore non-HTTP connections for now

    # Read the request (we don't use it, but we should consume it)
    event = await receive()
    assert event["type"] == "http.request"

    # Send the response
    await send({
        "type": "http.response.start",
        "status": 200,
        "headers": [
            (b"content-type", b"text/plain; charset=utf-8"),
            (b"content-length", b"13"),
        ],
    })
    await send({
        "type": "http.response.body",
        "body": b"Hello, world!",
        "more_body": False,
    })
```

Run it with Uvicorn:

```bash
pip install uvicorn
uvicorn app:application
```

That's a fully functional ASGI application. No framework, no dependencies beyond uvicorn.

## Reading the Full Request Body

HTTP bodies can arrive in chunks. The `more_body` flag tells you if more is coming:

```python
async def read_body(receive: callable) -> bytes:
    """Read the complete request body, handling chunks."""
    body = b""
    while True:
        event = await receive()
        if event["type"] == "http.request":
            body += event.get("body", b"")
            if not event.get("more_body", False):
                break
        elif event["type"] == "http.disconnect":
            break  # Client disconnected before sending full body
    return body
```

For small bodies, the server typically sends everything in one `http.request` event with `more_body=False`. For large bodies or streaming uploads, you'll see multiple events.

## A More Complete Example

```python
import json


async def application(scope, receive, send):
    if scope["type"] == "lifespan":
        await handle_lifespan(scope, receive, send)
    elif scope["type"] == "http":
        await handle_http(scope, receive, send)


async def handle_lifespan(scope, receive, send):
    while True:
        event = await receive()
        if event["type"] == "lifespan.startup":
            await send({"type": "lifespan.startup.complete"})
        elif event["type"] == "lifespan.shutdown":
            await send({"type": "lifespan.shutdown.complete"})
            break


async def handle_http(scope, receive, send):
    method = scope["method"]
    path = scope["path"]

    # Read the full body
    body = await read_body(receive)

    # Route
    if path == "/" and method == "GET":
        response_body = b"Hello from ASGI!"
        status = 200
    elif path == "/echo" and method == "POST":
        # Echo the request body back
        data = json.loads(body) if body else {}
        response_body = json.dumps({"echo": data}).encode()
        status = 200
    else:
        response_body = b"Not Found"
        status = 404

    await send({
        "type": "http.response.start",
        "status": status,
        "headers": [
            (b"content-type", b"application/json"),
            (b"content-length", str(len(response_body)).encode()),
        ],
    })
    await send({
        "type": "http.response.body",
        "body": response_body,
        "more_body": False,
    })


async def read_body(receive) -> bytes:
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
```

## Comparing WSGI and ASGI Side by Side

```python
# WSGI
def wsgi_app(environ, start_response):
    method = environ['REQUEST_METHOD']
    path = environ['PATH_INFO']
    body = environ['wsgi.input'].read(int(environ.get('CONTENT_LENGTH') or 0))

    start_response("200 OK", [("Content-Type", "text/plain")])
    return [b"Hello"]


# ASGI equivalent
async def asgi_app(scope, receive, send):
    method = scope['method']
    path = scope['path']
    body = await read_body(receive)

    await send({
        "type": "http.response.start",
        "status": 200,
        "headers": [(b"content-type", b"text/plain")],
    })
    await send({
        "type": "http.response.body",
        "body": b"Hello",
        "more_body": False,
    })
```

The ASGI version is more verbose for simple HTTP. That's the cost of the generality — the same interface handles HTTP, WebSockets, and lifespan, so HTTP alone requires a bit more ceremony.

## Headers: Bytes or Strings?

ASGI headers are byte-string tuples: `(b"content-type", b"text/plain")`. This is closer to the wire format — HTTP headers are bytes.

WSGI headers are string tuples: `("Content-Type", "text/plain")`. WSGI is at a higher abstraction level, hiding the byte/string conversion.

Frameworks handle the bytes-to-strings conversion for you. When you access `request.headers["content-type"]` in Starlette or FastAPI, the framework has already decoded the byte strings from `scope["headers"]`. But at the raw ASGI level, it's bytes.

## The `asgi` Key

Every scope dict includes an `"asgi"` key with version information:

```python
scope["asgi"] == {"version": "3.0"}
```

ASGI 3.0 is the current version (the one we're describing). Earlier versions had a different interface; you'll rarely encounter them today. If you're writing ASGI apps in 2024+, you're on 3.0.

## What Uvicorn Does

Uvicorn is to ASGI what Gunicorn is to WSGI — it handles the transport and calls your application.

When Uvicorn receives an HTTP request:
1. Reads bytes from the socket
2. Parses the HTTP request
3. Builds the `scope` dict
4. Creates `receive` and `send` callables backed by the socket
5. Calls `await app(scope, receive, send)`

When your app calls `await receive()`, Uvicorn either returns buffered data or reads more from the socket (awaiting the I/O without blocking other connections).

When your app calls `await send(event)`, Uvicorn serializes the event to HTTP bytes and writes them to the socket.

The event loop is asyncio. Multiple connections share a single thread via cooperative multitasking — each `await` point is an opportunity for the event loop to switch to another connection.

This is the key difference from WSGI: one thread can handle many concurrent connections, because `await` yields control voluntarily rather than blocking.

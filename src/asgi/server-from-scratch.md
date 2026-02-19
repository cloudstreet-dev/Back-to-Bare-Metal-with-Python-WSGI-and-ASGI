# Build an ASGI Server from Scratch

Building a WSGI server required sockets and HTTP parsing. Building an ASGI server requires all of that plus `asyncio`. The concepts are the same; the mechanics shift from blocking I/O to coroutines.

This chapter builds a working async HTTP server that calls ASGI applications correctly. Not production-ready — we'll leave chunked encoding and HTTP/2 for Uvicorn — but correct enough to understand what Uvicorn is actually doing.

## The Architecture

An asyncio-based server has a different shape than a threaded one:

```
asyncio event loop
    ↓
asyncio.start_server()
    ↓
handle_connection() — called as a coroutine for each connection
    ├── reads from asyncio.StreamReader
    ├── parses HTTP request
    ├── builds scope dict
    ├── creates receive/send coroutines
    ├── awaits app(scope, receive, send)
    └── writes to asyncio.StreamWriter
```

Instead of spawning a thread per connection, we create a coroutine per connection. The event loop switches between coroutines at `await` points, so hundreds of concurrent connections can be handled by a single thread.

## The HTTP Parser

Same logic as the WSGI server, adapted for async reading:

```python
import asyncio
from dataclasses import dataclass
from typing import Dict, List, Optional, Tuple


@dataclass
class ParsedRequest:
    method: str
    path: str
    query_string: bytes
    http_version: str
    headers: List[Tuple[bytes, bytes]]
    body: bytes


async def read_http_request(reader: asyncio.StreamReader) -> Optional[bytes]:
    """
    Read a complete HTTP request from an async stream.
    Returns raw bytes or None if the connection closed.
    """
    data = b""

    # Read until we have the complete headers
    try:
        while b"\r\n\r\n" not in data:
            chunk = await asyncio.wait_for(reader.read(4096), timeout=5.0)
            if not chunk:
                return None
            data += chunk
    except asyncio.TimeoutError:
        return None

    # Parse Content-Length to read the body
    header_end = data.find(b"\r\n\r\n") + 4
    content_length = 0

    for line in data[:header_end].decode("latin-1").split("\r\n"):
        if line.lower().startswith("content-length:"):
            try:
                content_length = int(line.split(":", 1)[1].strip())
            except ValueError:
                pass
            break

    # Read body if needed
    body_received = len(data) - header_end
    while body_received < content_length:
        try:
            chunk = await asyncio.wait_for(reader.read(4096), timeout=5.0)
        except asyncio.TimeoutError:
            break
        if not chunk:
            break
        data += chunk
        body_received += len(chunk)

    return data


def parse_request(raw: bytes) -> Optional[ParsedRequest]:
    """Parse raw HTTP bytes into a ParsedRequest."""
    header_end = raw.find(b"\r\n\r\n")
    if header_end == -1:
        return None

    header_section = raw[:header_end].decode("latin-1")
    body = raw[header_end + 4:]

    lines = header_section.split("\r\n")
    try:
        method, raw_path, http_version = lines[0].split(" ", 2)
    except ValueError:
        return None

    # Split path and query string
    if b"?" in raw_path.encode():
        path_bytes, query_string = raw_path.encode().split(b"?", 1)
    else:
        path_bytes = raw_path.encode()
        query_string = b""

    # Parse headers as byte tuples (ASGI format)
    headers: List[Tuple[bytes, bytes]] = []
    for line in lines[1:]:
        if ": " in line:
            name, _, value = line.partition(": ")
            headers.append((name.lower().encode("latin-1"),
                            value.encode("latin-1")))

    # Trim body
    content_length = 0
    for name, value in headers:
        if name == b"content-length":
            try:
                content_length = int(value)
            except ValueError:
                pass
            break

    return ParsedRequest(
        method=method.upper(),
        path=path_bytes.decode("latin-1"),
        query_string=query_string,
        http_version=http_version,
        headers=headers,
        body=body[:content_length],
    )
```

## The ASGI Bridge

The key piece: create `receive` and `send` coroutines that bridge between the ASGI protocol and the TCP connection.

```python
async def make_receive_send(
    request: ParsedRequest,
    writer: asyncio.StreamWriter,
) -> tuple:
    """
    Create the receive and send callables for an ASGI HTTP connection.
    Returns (receive, send, get_response_started).
    """
    # Track whether we've sent the body yet
    body_sent = False
    request_consumed = False
    response_started = False
    disconnect_event = asyncio.Event()

    async def receive():
        nonlocal request_consumed
        if not request_consumed:
            request_consumed = True
            return {
                "type": "http.request",
                "body": request.body,
                "more_body": False,
            }
        # Wait for disconnect (in a real server, we'd detect this from the socket)
        await disconnect_event.wait()
        return {"type": "http.disconnect"}

    response_headers = []
    response_status = None

    async def send(event):
        nonlocal response_started, body_sent, response_status

        if event["type"] == "http.response.start":
            response_status = event["status"]
            response_headers.extend(event.get("headers", []))
            response_started = True

        elif event["type"] == "http.response.body":
            if not response_started:
                raise RuntimeError("Must send http.response.start before body")

            body = event.get("body", b"")
            more_body = event.get("more_body", False)

            if not body_sent:
                # Write the HTTP response headers first
                status_line = f"HTTP/1.1 {response_status} {get_reason(response_status)}\r\n"
                writer.write(status_line.encode("latin-1"))

                for name, value in response_headers:
                    if isinstance(name, bytes):
                        name = name.decode("latin-1")
                    if isinstance(value, bytes):
                        value = value.decode("latin-1")
                    writer.write(f"{name}: {value}\r\n".encode("latin-1"))
                writer.write(b"\r\n")
                body_sent = True

            writer.write(body)

            if not more_body:
                await writer.drain()
                disconnect_event.set()

    return receive, send


def get_reason(status_code: int) -> str:
    reasons = {
        200: "OK", 201: "Created", 204: "No Content",
        301: "Moved Permanently", 302: "Found", 304: "Not Modified",
        400: "Bad Request", 401: "Unauthorized", 403: "Forbidden",
        404: "Not Found", 405: "Method Not Allowed",
        422: "Unprocessable Entity",
        500: "Internal Server Error",
    }
    return reasons.get(status_code, "Unknown")
```

## The Server Loop

```python
import sys
from typing import Callable


async def handle_connection(
    reader: asyncio.StreamReader,
    writer: asyncio.StreamWriter,
    app: Callable,
    server_host: str,
    server_port: int,
) -> None:
    """Handle one HTTP connection."""
    try:
        # Read the request
        raw = await read_http_request(reader)
        if not raw:
            return

        # Parse it
        request = parse_request(raw)
        if request is None:
            writer.write(b"HTTP/1.1 400 Bad Request\r\n\r\n")
            await writer.drain()
            return

        # Build the ASGI scope
        scope = {
            "type": "http",
            "asgi": {"version": "3.0"},
            "http_version": request.http_version.replace("HTTP/", ""),
            "method": request.method,
            "path": request.path,
            "raw_path": request.path.encode("latin-1"),
            "query_string": request.query_string,
            "root_path": "",
            "scheme": "http",
            "headers": request.headers,
            "server": (server_host, server_port),
        }

        # Get client address
        peername = writer.get_extra_info("peername")
        if peername:
            scope["client"] = peername

        # Create receive/send
        receive, send = await make_receive_send(request, writer)

        # Call the ASGI app
        await app(scope, receive, send)

    except Exception as e:
        print(f"Error handling connection: {e}", file=sys.stderr)
        try:
            writer.write(b"HTTP/1.1 500 Internal Server Error\r\n\r\n")
            await writer.drain()
        except Exception:
            pass
    finally:
        try:
            writer.close()
            await writer.wait_closed()
        except Exception:
            pass


async def serve(
    app: Callable,
    host: str = "127.0.0.1",
    port: int = 8000,
) -> None:
    """Start the ASGI server."""
    # Send the lifespan startup event
    await send_lifespan_startup(app)

    server = await asyncio.start_server(
        lambda r, w: handle_connection(r, w, app, host, port),
        host,
        port,
    )

    async with server:
        addr = server.sockets[0].getsockname()
        print(f"Serving on http://{addr[0]}:{addr[1]}", file=sys.stderr)
        try:
            await server.serve_forever()
        except (KeyboardInterrupt, asyncio.CancelledError):
            pass

    await send_lifespan_shutdown(app)


async def send_lifespan_startup(app: Callable) -> None:
    """Send the lifespan.startup event if the app handles it."""
    scope = {"type": "lifespan", "asgi": {"version": "3.0"}}
    startup_complete = asyncio.Event()
    events = asyncio.Queue()

    await events.put({"type": "lifespan.startup"})

    async def receive():
        return await events.get()

    async def send(event):
        if event["type"] == "lifespan.startup.complete":
            startup_complete.set()
        elif event["type"] == "lifespan.startup.failed":
            raise RuntimeError(f"Startup failed: {event.get('message', '')}")

    try:
        task = asyncio.create_task(app(scope, receive, send))
        await asyncio.wait_for(startup_complete.wait(), timeout=10.0)
        # Don't await the task — it runs for the app's lifetime
    except asyncio.TimeoutError:
        print("Warning: lifespan startup timed out", file=sys.stderr)
    except Exception as e:
        print(f"Warning: lifespan startup failed: {e}", file=sys.stderr)


async def send_lifespan_shutdown(app: Callable) -> None:
    """Send the lifespan.shutdown event."""
    # In a full implementation, we'd track the lifespan task
    # and send shutdown. Simplified here.
    pass
```

## Running the Server

```python
# asgi_server.py

# [paste all the code above, then:]

if __name__ == "__main__":
    from asgi_tasks import application  # The app from the previous chapter
    asyncio.run(serve(application))
```

```bash
python asgi_server.py &
curl -X POST http://127.0.0.1:8000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Works on my ASGI server"}'
```

## Testing Concurrency

The real test: can our server handle multiple requests concurrently? Since we're using asyncio, the answer should be yes — as long as the handlers don't block.

```python
# concurrent_test.py
import asyncio
import time


async def make_request(session_num: int) -> float:
    """Make an HTTP request and return how long it took."""
    start = time.monotonic()
    reader, writer = await asyncio.open_connection("127.0.0.1", 8000)

    request = (
        f"GET /tasks HTTP/1.1\r\n"
        f"Host: localhost\r\n"
        f"Connection: close\r\n"
        f"\r\n"
    )
    writer.write(request.encode())
    await writer.drain()

    response = b""
    while chunk := await reader.read(4096):
        response += chunk

    writer.close()
    elapsed = time.monotonic() - start
    print(f"Request {session_num}: {elapsed*1000:.1f}ms")
    return elapsed


async def main():
    # Make 10 concurrent requests
    tasks = [make_request(i) for i in range(10)]
    times = await asyncio.gather(*tasks)
    print(f"Total elapsed: {max(times)*1000:.1f}ms")
    print(f"Average: {sum(times)/len(times)*1000:.1f}ms")


asyncio.run(main())
```

With a synchronous server (our WSGI server from earlier), 10 sequential requests take 10x as long as one. With our async server, 10 concurrent requests should take roughly as long as one — because the event loop handles them all simultaneously.

## What Uvicorn Does Differently

Our server is correct but incomplete. Uvicorn adds:

**HTTP/1.1 keep-alive**: reuse the connection for multiple requests. We close after each one. This matters for performance — TLS handshakes and TCP connection setup are expensive.

**HTTP/2**: multiplexed streams over a single connection. This is substantially more complex — h2 (the Python HTTP/2 library) handles the framing; Uvicorn maps the streams to ASGI scopes.

**Proper streaming**: chunked transfer encoding for responses without Content-Length, streaming request body parsing.

**SSL/TLS**: pass `--ssl-keyfile` and `--ssl-certfile` to Uvicorn and it handles TLS. Our server is plain HTTP.

**Worker processes**: `uvicorn --workers 4` forks four worker processes, each running the asyncio event loop. More workers = more CPU cores utilized.

**Graceful shutdown**: when you send SIGTERM, Uvicorn stops accepting new connections, finishes in-flight requests, then exits. Our `KeyboardInterrupt` handling is abrupt.

But the core logic — accept connection, read request, build scope, call `await app(scope, receive, send)`, write response — is exactly what we've implemented.

## The asyncio Mental Model

One thing worth making explicit: `asyncio` is cooperative multitasking. Tasks run until they explicitly yield control with `await`. When a task awaits network I/O (`reader.read()`, `writer.drain()`), the event loop can run other tasks.

This means:
- **CPU-bound work blocks everything**: if a handler does heavy computation without awaiting, no other connection can run. Use `asyncio.run_in_executor` for CPU-bound work.
- **Blocking I/O blocks everything**: `time.sleep(1)` in a handler blocks the entire server for 1 second. Use `await asyncio.sleep(1)` instead.
- **One thread, many connections**: unlike threaded servers, there's no race condition between connections (within one process). The event loop is single-threaded.

This model is why async Python is fast for I/O-bound workloads (web servers, API proxies) but not for CPU-bound ones (image processing, ML inference). Know which one you have.

# Build a WSGI Server from Scratch

We've been running our WSGI applications with `wsgiref.simple_server`. It works, but it's a black box. In this chapter, we'll build our own WSGI server using Python's `socket` module — the same primitives that Gunicorn uses, just without fifteen years of production hardening.

By the end of this chapter, you'll understand exactly what Gunicorn's workers are doing on every request.

## What a WSGI Server Must Do

A WSGI server has one job: accept TCP connections, parse HTTP requests, call WSGI applications, and send HTTP responses. In order:

1. Bind to a port and listen for connections
2. Accept a connection
3. Read bytes from the socket
4. Parse the bytes into an HTTP request
5. Build the `environ` dictionary
6. Define a `start_response` callable
7. Call the WSGI application
8. Serialize the response (status + headers + body)
9. Write the response bytes to the socket
10. Close the connection (or keep it alive for HTTP/1.1)

We'll implement all of these, handling one request at a time (no threading — that's a refinement for later).

## The HTTP Parser

First, let's build a proper request parser:

```python
from dataclasses import dataclass, field
from typing import Dict, List, Tuple, Optional
import io


@dataclass
class ParsedRequest:
    method: str
    path: str
    query_string: str
    http_version: str
    headers: Dict[str, str]
    body: bytes


def parse_request(raw: bytes) -> Optional[ParsedRequest]:
    """
    Parse raw HTTP request bytes into a ParsedRequest.
    Returns None if the request is malformed.
    """
    # Split at the blank line separating headers from body
    header_end = raw.find(b"\r\n\r\n")
    if header_end == -1:
        return None  # Incomplete request

    header_section = raw[:header_end].decode("latin-1")
    body = raw[header_end + 4:]

    lines = header_section.split("\r\n")
    if not lines:
        return None

    # Parse the request line
    try:
        method, raw_path, http_version = lines[0].split(" ", 2)
    except ValueError:
        return None

    # Split path from query string
    if "?" in raw_path:
        path, query_string = raw_path.split("?", 1)
    else:
        path, query_string = raw_path, ""

    # Parse headers
    headers: Dict[str, str] = {}
    for line in lines[1:]:
        if ": " in line:
            name, _, value = line.partition(": ")
            headers[name.lower()] = value

    # Trim body to Content-Length if present
    content_length = int(headers.get("content-length", len(body)))
    body = body[:content_length]

    return ParsedRequest(
        method=method,
        path=path,
        query_string=query_string,
        http_version=http_version,
        headers=headers,
        body=body,
    )
```

## Building the `environ` Dictionary

The spec defines exactly what keys `environ` must contain. Here's the conversion from our parsed request:

```python
import os
import sys


def build_environ(request: ParsedRequest, server_name: str, server_port: int) -> dict:
    """
    Build a WSGI environ dict from a parsed HTTP request.
    See PEP 3333 for the complete specification.
    """
    environ = {
        # CGI variables
        "REQUEST_METHOD": request.method,
        "SCRIPT_NAME": "",
        "PATH_INFO": request.path,
        "QUERY_STRING": request.query_string,
        "SERVER_NAME": server_name,
        "SERVER_PORT": str(server_port),
        "SERVER_PROTOCOL": request.http_version,
        "GATEWAY_INTERFACE": "CGI/1.1",

        # WSGI variables
        "wsgi.version": (1, 0),
        "wsgi.url_scheme": "http",
        "wsgi.input": io.BytesIO(request.body),
        "wsgi.errors": sys.stderr,
        "wsgi.multithread": False,
        "wsgi.multiprocess": False,
        "wsgi.run_once": False,
    }

    # Content-Type and Content-Length: no HTTP_ prefix (CGI convention)
    if "content-type" in request.headers:
        environ["CONTENT_TYPE"] = request.headers["content-type"]
    else:
        environ["CONTENT_TYPE"] = ""

    if "content-length" in request.headers:
        environ["CONTENT_LENGTH"] = request.headers["content-length"]
    else:
        environ["CONTENT_LENGTH"] = ""

    # All other headers: HTTP_ prefix, uppercased, hyphens → underscores
    for name, value in request.headers.items():
        if name in ("content-type", "content-length"):
            continue
        key = "HTTP_" + name.upper().replace("-", "_")
        environ[key] = value

    return environ
```

## The Response Builder

```python
def build_response(status: str, headers: List[Tuple[str, str]], body: bytes) -> bytes:
    """Serialize a WSGI response to HTTP bytes."""
    lines = [f"HTTP/1.1 {status}\r\n"]
    for name, value in headers:
        lines.append(f"{name}: {value}\r\n")
    lines.append("\r\n")
    return "".join(lines).encode("latin-1") + body
```

## The Server

Now we put it all together:

```python
import socket
import sys
from typing import Callable


def serve(app: Callable, host: str = "127.0.0.1", port: int = 8000) -> None:
    """
    A minimal single-threaded WSGI server.
    Handles one request at a time. Not suitable for production,
    but suitable for understanding what production servers do.
    """
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    # Allow reusing the address immediately after restart
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

    server_socket.bind((host, port))
    server_socket.listen(5)

    print(f"Serving on http://{host}:{port}", file=sys.stderr)

    while True:
        try:
            conn, addr = server_socket.accept()
            handle_connection(conn, addr, app, host, port)
        except KeyboardInterrupt:
            print("\nShutting down.", file=sys.stderr)
            break

    server_socket.close()


def handle_connection(
    conn: socket.socket,
    addr: tuple,
    app: Callable,
    server_name: str,
    server_port: int,
) -> None:
    """Handle a single HTTP connection."""
    try:
        # Read the full request
        raw = receive_request(conn)
        if not raw:
            return

        # Parse it
        request = parse_request(raw)
        if request is None:
            conn.sendall(b"HTTP/1.1 400 Bad Request\r\n\r\n")
            return

        # Build environ and call the app
        environ = build_environ(request, server_name, server_port)
        status, headers, body = call_app(app, environ)

        # Send the response
        response = build_response(status, headers, body)
        conn.sendall(response)

    except Exception as e:
        print(f"Error handling request: {e}", file=sys.stderr)
        try:
            conn.sendall(b"HTTP/1.1 500 Internal Server Error\r\n\r\n")
        except Exception:
            pass
    finally:
        conn.close()


def receive_request(conn: socket.socket) -> bytes:
    """
    Read bytes from a socket until we have a complete HTTP request.
    Handles the header/body split correctly.
    """
    data = b""
    conn.settimeout(5.0)

    try:
        # Read until we have the full headers
        while b"\r\n\r\n" not in data:
            chunk = conn.recv(4096)
            if not chunk:
                return data
            data += chunk

        # Find the end of headers
        header_end = data.find(b"\r\n\r\n") + 4

        # Determine Content-Length from headers
        header_section = data[:header_end].decode("latin-1")
        content_length = 0
        for line in header_section.split("\r\n"):
            if line.lower().startswith("content-length:"):
                try:
                    content_length = int(line.split(":", 1)[1].strip())
                except ValueError:
                    pass
                break

        # Read the body if needed
        body_received = len(data) - header_end
        while body_received < content_length:
            chunk = conn.recv(4096)
            if not chunk:
                break
            data += chunk
            body_received += len(chunk)

    except socket.timeout:
        pass  # Return whatever we have

    return data


def call_app(app: Callable, environ: dict) -> tuple:
    """
    Call a WSGI application and collect the response.
    Returns (status, headers, body_bytes).
    """
    response_args = []

    def start_response(status, headers, exc_info=None):
        if exc_info:
            try:
                if response_args:
                    raise exc_info[1].with_traceback(exc_info[2])
            finally:
                exc_info = None
        response_args.clear()
        response_args.append((status, headers))

    result = app(environ, start_response)

    try:
        body = b"".join(result)
    finally:
        if hasattr(result, "close"):
            result.close()

    if not response_args:
        raise RuntimeError("WSGI app did not call start_response")

    status, headers = response_args[0]
    return status, headers, body
```

## Putting It All Together

```python
# wsgi_server.py — the complete server in one file

import io
import os
import socket
import sys
from dataclasses import dataclass
from typing import Callable, Dict, List, Optional, Tuple


@dataclass
class ParsedRequest:
    method: str
    path: str
    query_string: str
    http_version: str
    headers: Dict[str, str]
    body: bytes


def parse_request(raw: bytes) -> Optional[ParsedRequest]:
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
    path, query_string = (raw_path.split("?", 1) if "?" in raw_path
                          else (raw_path, ""))
    headers: Dict[str, str] = {}
    for line in lines[1:]:
        if ": " in line:
            name, _, value = line.partition(": ")
            headers[name.lower()] = value
    content_length = int(headers.get("content-length", len(body)))
    return ParsedRequest(method, path, query_string, http_version,
                         headers, body[:content_length])


def build_environ(req: ParsedRequest, host: str, port: int) -> dict:
    env = {
        "REQUEST_METHOD": req.method,
        "SCRIPT_NAME": "",
        "PATH_INFO": req.path,
        "QUERY_STRING": req.query_string,
        "SERVER_NAME": host,
        "SERVER_PORT": str(port),
        "SERVER_PROTOCOL": req.http_version,
        "GATEWAY_INTERFACE": "CGI/1.1",
        "wsgi.version": (1, 0),
        "wsgi.url_scheme": "http",
        "wsgi.input": io.BytesIO(req.body),
        "wsgi.errors": sys.stderr,
        "wsgi.multithread": False,
        "wsgi.multiprocess": False,
        "wsgi.run_once": False,
        "CONTENT_TYPE": req.headers.get("content-type", ""),
        "CONTENT_LENGTH": req.headers.get("content-length", ""),
    }
    for name, value in req.headers.items():
        if name not in ("content-type", "content-length"):
            env["HTTP_" + name.upper().replace("-", "_")] = value
    return env


def build_response(status: str, headers: List[Tuple[str, str]],
                   body: bytes) -> bytes:
    lines = [f"HTTP/1.1 {status}\r\n"]
    for name, value in headers:
        lines.append(f"{name}: {value}\r\n")
    lines.append("\r\n")
    return "".join(lines).encode("latin-1") + body


def receive_request(conn: socket.socket) -> bytes:
    data = b""
    conn.settimeout(5.0)
    try:
        while b"\r\n\r\n" not in data:
            chunk = conn.recv(4096)
            if not chunk:
                return data
            data += chunk
        header_end = data.find(b"\r\n\r\n") + 4
        content_length = 0
        for line in data[:header_end].decode("latin-1").split("\r\n"):
            if line.lower().startswith("content-length:"):
                try:
                    content_length = int(line.split(":", 1)[1].strip())
                except ValueError:
                    pass
                break
        while len(data) - header_end < content_length:
            chunk = conn.recv(4096)
            if not chunk:
                break
            data += chunk
    except socket.timeout:
        pass
    return data


def call_app(app: Callable, environ: dict) -> tuple:
    response_args = []

    def start_response(status, headers, exc_info=None):
        if exc_info:
            try:
                if response_args:
                    raise exc_info[1].with_traceback(exc_info[2])
            finally:
                exc_info = None
        response_args.clear()
        response_args.append((status, headers))

    result = app(environ, start_response)
    try:
        body = b"".join(result)
    finally:
        if hasattr(result, "close"):
            result.close()
    if not response_args:
        raise RuntimeError("App did not call start_response")
    status, headers = response_args[0]
    return status, headers, body


def serve(app: Callable, host: str = "127.0.0.1", port: int = 8000) -> None:
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.bind((host, port))
    sock.listen(5)
    print(f"Serving on http://{host}:{port}", file=sys.stderr)
    try:
        while True:
            conn, addr = sock.accept()
            try:
                raw = receive_request(conn)
                if not raw:
                    continue
                req = parse_request(raw)
                if req is None:
                    conn.sendall(b"HTTP/1.1 400 Bad Request\r\n\r\n")
                    continue
                environ = build_environ(req, host, port)
                status, headers, body = call_app(app, environ)
                conn.sendall(build_response(status, headers, body))
            except Exception as e:
                print(f"Error: {e}", file=sys.stderr)
                try:
                    conn.sendall(b"HTTP/1.1 500 Internal Server Error\r\n\r\n")
                except Exception:
                    pass
            finally:
                conn.close()
    except KeyboardInterrupt:
        print("\nShutting down.", file=sys.stderr)
    finally:
        sock.close()


# --- Test application ---

def hello_app(environ, start_response):
    name = environ.get("QUERY_STRING", "").split("=")[-1] or "world"
    body = f"Hello, {name}!\n".encode("utf-8")
    start_response("200 OK", [
        ("Content-Type", "text/plain"),
        ("Content-Length", str(len(body))),
    ])
    return [body]


if __name__ == "__main__":
    serve(hello_app)
```

Save as `wsgi_server.py` and test it:

```bash
python wsgi_server.py &
curl "http://127.0.0.1:8000/?name=WSGI"
# Hello, WSGI!
```

Now point it at the tasks app from the previous chapter:

```python
# at the bottom of wsgi_server.py, replace the __main__ block:
if __name__ == "__main__":
    from tasks_app import application
    serve(application)
```

```bash
curl -X POST http://127.0.0.1:8000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Works on my server"}'
```

## What Gunicorn Does That We Don't

Our server handles one request at a time. Gunicorn has workers — multiple processes or threads that each run the request loop above simultaneously. The `sync` worker is essentially what we've built, replicated across N processes. The `gthread` worker uses threads within each process. The `gevent` worker uses greenlets.

Beyond concurrency, Gunicorn adds:

- **HTTP/1.1 keep-alive**: reusing connections for multiple requests
- **Chunked transfer encoding**: streaming responses without known `Content-Length`
- **Request size limits**: protecting against large payloads
- **Graceful worker restarts**: new workers start before old ones die
- **Worker timeout killing**: killing workers that hang

But the core loop — accept, parse, environ, call, serialize, send — is exactly what we've built.

## The Latin-1 Encoding Detail

You may have noticed we decode HTTP headers as `latin-1` rather than `utf-8`. This is intentional.

The HTTP spec (RFC 7230) says header field names and values are ISO-8859-1 (latin-1) by default. This is a historical artifact from HTTP/1.0. Headers containing non-ASCII characters are technically not valid in HTTP/1.1 (they should be encoded, e.g., RFC 5987 encoding for `Content-Disposition` filenames).

In practice, Python's WSGI servers use `latin-1` for headers to preserve byte-for-byte fidelity. If you round-trip a header through `encode('latin-1').decode('latin-1')`, you get the original bytes back — which matters for the WSGI spec's requirement that `environ` values be native strings.

This is one of those details that doesn't matter until it does, at which point you'll spend a day debugging mysterious encoding errors.

## The `SO_REUSEADDR` Detail

```python
server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
```

Without this, if you restart the server quickly after stopping it, you'll get `[Errno 98] Address already in use`. This happens because the OS keeps the socket in `TIME_WAIT` state for a while after close, waiting for any stray packets that might still be in transit. `SO_REUSEADDR` tells the OS to let us reuse the address anyway.

Gunicorn sets this too. So does every other production server. It's the first thing you add after "it works once" breaks down.

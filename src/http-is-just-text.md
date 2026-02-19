# HTTP Is Just Text

Before we talk about WSGI or ASGI, we need to talk about what they're abstracting over. And what they're abstracting over is HTTP. And HTTP is just text.

This is not a simplification. Open a TCP connection to port 80 of any web server in the world, type the right bytes, and you'll get an HTTP response. You don't need a library. You need a socket and the knowledge of what to type.

Let's actually do it.

## Talking to a Web Server with a Raw Socket

```python
import socket

def raw_http_request(host: str, path: str = "/") -> str:
    """Make an HTTP/1.1 GET request using nothing but a socket."""
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.connect((host, 80))

    # This is a complete, valid HTTP/1.1 request.
    request = (
        f"GET {path} HTTP/1.1\r\n"
        f"Host: {host}\r\n"
        f"Connection: close\r\n"
        f"\r\n"
    )

    sock.sendall(request.encode("utf-8"))

    # Read the response in chunks
    response = b""
    while chunk := sock.recv(4096):
        response += chunk

    sock.close()
    return response.decode("utf-8", errors="replace")


if __name__ == "__main__":
    response = raw_http_request("example.com")
    print(response[:500])  # Just the beginning
```

Run this and you'll see something like:

```
HTTP/1.1 200 OK
Content-Encoding: gzip
Accept-Ranges: bytes
Age: 123456
Cache-Control: max-age=604800
Content-Type: text/html; charset=UTF-8
Date: Thu, 01 Jan 2026 00:00:00 GMT
...

<!doctype html>
<html>
...
```

That's it. That's HTTP. A text request, a text response. The format is specified in RFC 9110 (and historically RFC 2616, RFC 7230, etc.) but the format itself is not complicated.

## The Structure of an HTTP Request

An HTTP request has this shape:

```
METHOD /path HTTP/version\r\n
Header-Name: header-value\r\n
Another-Header: another-value\r\n
\r\n
[optional body]
```

A minimal GET request:

```
GET /index.html HTTP/1.1\r\n
Host: example.com\r\n
\r\n
```

A POST request with a body:

```
POST /api/users HTTP/1.1\r\n
Host: api.example.com\r\n
Content-Type: application/json\r\n
Content-Length: 27\r\n
\r\n
{"name": "Alice", "age": 30}
```

Three parts: the request line, the headers, and the body. Each header is on its own line. Headers are separated from the body by a blank line (`\r\n\r\n`). The `\r\n` is a carriage return followed by a newline — HTTP requires both, not just `\n`.

## The Structure of an HTTP Response

An HTTP response:

```
HTTP/version STATUS_CODE Reason Phrase\r\n
Header-Name: header-value\r\n
Another-Header: another-value\r\n
\r\n
[body]
```

A minimal response:

```
HTTP/1.1 200 OK\r\n
Content-Type: text/plain\r\n
Content-Length: 13\r\n
\r\n
Hello, world!
```

The status line, headers, blank line, body. Same structure, mirrored.

## Parsing HTTP Requests

Let's write a basic HTTP request parser. Not to use in production — for understanding what Gunicorn and Uvicorn do on every single request before they ever touch your application code.

```python
from dataclasses import dataclass, field
from typing import Dict, Optional


@dataclass
class HTTPRequest:
    method: str
    path: str
    query_string: str
    http_version: str
    headers: Dict[str, str]
    body: bytes

    @property
    def content_type(self) -> Optional[str]:
        return self.headers.get("content-type")

    @property
    def content_length(self) -> int:
        return int(self.headers.get("content-length", 0))


def parse_request(raw: bytes) -> HTTPRequest:
    """
    Parse a raw HTTP request into an HTTPRequest object.
    Handles the header/body split and basic header parsing.
    """
    # Split headers from body at the blank line
    header_section, _, body = raw.partition(b"\r\n\r\n")

    # Split header section into individual lines
    lines = header_section.decode("utf-8", errors="replace").split("\r\n")

    # First line is the request line
    request_line = lines[0]
    method, raw_path, http_version = request_line.split(" ", 2)

    # Split path from query string
    if "?" in raw_path:
        path, query_string = raw_path.split("?", 1)
    else:
        path, query_string = raw_path, ""

    # Parse headers (everything after the request line)
    headers = {}
    for line in lines[1:]:
        if ": " in line:
            name, _, value = line.partition(": ")
            headers[name.lower()] = value

    return HTTPRequest(
        method=method,
        path=path,
        query_string=query_string,
        http_version=http_version,
        headers=headers,
        body=body,
    )


# Test it
raw_request = (
    b"POST /api/users?active=true HTTP/1.1\r\n"
    b"Host: localhost\r\n"
    b"Content-Type: application/json\r\n"
    b"Content-Length: 27\r\n"
    b"\r\n"
    b'{"name": "Alice", "age": 30}'
)

req = parse_request(raw_request)
print(f"Method: {req.method}")
print(f"Path: {req.path}")
print(f"Query: {req.query_string}")
print(f"Content-Type: {req.content_type}")
print(f"Body: {req.body}")
```

Output:
```
Method: POST
Path: /api/users
Query: active=true
Content-Type: application/json
Body: b'{"name": "Alice", "age": 30}'
```

This is, roughly, what every web server does before handing control to your application. Gunicorn's HTTP parser is more robust (it handles edge cases, malformed requests, chunked transfer encoding, etc.), but conceptually it's doing exactly this.

## Building an HTTP Response

The other direction: given what you want to send back, construct valid HTTP bytes.

```python
def build_response(
    status_code: int,
    reason: str,
    headers: Dict[str, str],
    body: bytes,
) -> bytes:
    """Build a raw HTTP/1.1 response."""
    status_line = f"HTTP/1.1 {status_code} {reason}\r\n"

    # Always include Content-Length
    headers["Content-Length"] = str(len(body))

    header_lines = "".join(
        f"{name}: {value}\r\n"
        for name, value in headers.items()
    )

    return (
        status_line.encode("utf-8")
        + header_lines.encode("utf-8")
        + b"\r\n"
        + body
    )


response = build_response(
    200,
    "OK",
    {"Content-Type": "text/plain"},
    b"Hello, world!",
)
print(response.decode("utf-8"))
```

Output:
```
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 13

Hello, world!
```

## What WSGI Does With All This

Now think about this: when a request comes in to a Gunicorn worker, the worker:

1. Reads bytes from the socket
2. Parses them into method, path, headers, body (as above)
3. Packs all of that into a dictionary called `environ`
4. Calls your WSGI application with `environ` and `start_response`
5. Takes whatever your application returns and writes it back to the socket as HTTP response bytes

The `environ` dictionary is just a structured version of the parsed HTTP request. `REQUEST_METHOD` is the method. `PATH_INFO` is the path. `HTTP_CONTENT_TYPE` is the Content-Type header. `wsgi.input` is a file-like object wrapping the body bytes.

When you call `start_response("200 OK", [("Content-Type", "text/plain")])` in your WSGI app, you're providing the status line and headers that the server will write back. When you return `[b"Hello, world!"]`, you're providing the response body.

The server just... sends it.

```
[raw bytes in] -> [parse] -> [your callable] -> [serialize] -> [raw bytes out]
```

That's the entire pipeline. WSGI is just the contract for the middle part.

## Keepalive, Chunked Encoding, and Things We're Ignoring

Real HTTP has some complexity we've glossed over:

**Connection: keep-alive** — HTTP/1.1 defaults to keeping the connection open for multiple requests. The server needs to know when one request ends and the next begins, which it does via `Content-Length` or chunked transfer encoding.

**Chunked transfer encoding** — instead of specifying `Content-Length` upfront, you can stream the response in chunks, each prefixed with its size in hex. This is how streaming responses work.

**HTTP/2** — multiplexed streams over a single connection, binary framing, header compression. Same semantics, very different wire format.

**TLS** — everything above happens over an encrypted connection. Same protocol, but the bytes going over the wire are ciphertext.

WSGI abstracts all of this. You don't handle keep-alive or chunked encoding directly. The server does. You write your callable; the server handles the transport.

ASGI handles more of these edge cases natively — particularly streaming — which is part of why it exists. We'll get there.

## The Thing to Hold Onto

HTTP is a text protocol. Requests are lines of text: a request line, headers, body. Responses are lines of text: a status line, headers, body. The blank line between headers and body is significant. The `\r\n` line endings are required.

Everything your framework does is ultimately:

1. Parse the incoming text into a convenient Python object
2. Call your handler function
3. Serialize the result back into text

That's the whole transaction. Hold that mental model as we build the WSGI layer on top of it.

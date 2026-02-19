# The WSGI Spec (It Fits on a Napkin)

PEP 3333 is the WSGI specification. It is 2,500 words long. For context, this chapter is longer. The actual interface it defines is expressible in fewer than ten lines of Python. This is either a sign of elegant design or a sign that we've been dramatically over-complicating web development for twenty years. Possibly both.

Let's read the spec together — not the full PEP, but the essential contract it defines.

## The Interface in Full

The complete WSGI interface, distilled:

```python
def application(environ: dict, start_response: callable) -> Iterable[bytes]:
    """
    A WSGI application is any callable that:
    1. Accepts environ (dict) and start_response (callable)
    2. Calls start_response(status, response_headers) exactly once before returning
    3. Returns an iterable of byte strings (the response body)
    """
    status = "200 OK"
    response_headers = [("Content-Type", "text/plain; charset=utf-8")]
    start_response(status, response_headers)
    return [b"Hello, world!\n"]
```

That's it. That's WSGI. Everything else is detail.

## The `environ` Dictionary

`environ` is a Python dictionary containing CGI-style environment variables plus some WSGI-specific additions. When Gunicorn receives an HTTP request, it parses it and packs the results into this dict.

Here are the keys your application will actually use:

```python
# Request method
environ['REQUEST_METHOD']    # "GET", "POST", "PUT", "DELETE", etc.

# URL components
environ['PATH_INFO']         # "/users/42" — the URL path
environ['QUERY_STRING']      # "active=true&page=2" — without the "?"
environ['SERVER_NAME']       # "example.com"
environ['SERVER_PORT']       # "80" (note: string, not int)

# HTTP headers (prefixed with HTTP_, hyphens become underscores, uppercased)
environ['HTTP_HOST']                # "example.com"
environ['HTTP_ACCEPT']              # "text/html,application/xhtml+xml,..."
environ['HTTP_AUTHORIZATION']       # "Bearer abc123"
environ['HTTP_CONTENT_TYPE']        # Note: also available as CONTENT_TYPE (no HTTP_ prefix)
environ['CONTENT_TYPE']             # "application/json"
environ['CONTENT_LENGTH']           # "42" (string, or empty string if unknown)

# Request body
environ['wsgi.input']        # file-like object, read() to get the body bytes

# WSGI metadata
environ['wsgi.version']      # (1, 0)
environ['wsgi.url_scheme']   # "http" or "https"
environ['wsgi.multithread']  # True if server may run multiple threads
environ['wsgi.multiprocess'] # True if server may fork multiple processes
environ['wsgi.run_once']     # True if application will only be invoked once
environ['wsgi.errors']       # file-like object for error output (stderr)
```

Let's write a small app that dumps the environ so you can see it for yourself:

```python
import json


def environ_dumper(environ, start_response):
    """Dump the environ dict as JSON for debugging."""
    # Some values aren't JSON-serializable; convert them
    safe_environ = {}
    for key, value in sorted(environ.items()):
        if isinstance(value, (str, int, float, bool, type(None))):
            safe_environ[key] = value
        else:
            safe_environ[key] = f"<{type(value).__name__}>"

    body = json.dumps(safe_environ, indent=2).encode("utf-8")

    start_response("200 OK", [
        ("Content-Type", "application/json"),
        ("Content-Length", str(len(body))),
    ])
    return [body]


if __name__ == "__main__":
    from wsgiref.simple_server import make_server
    server = make_server("127.0.0.1", 8000, environ_dumper)
    print("Serving on http://127.0.0.1:8000")
    server.serve_forever()
```

Run this, hit `http://127.0.0.1:8000/some/path?foo=bar` in your browser, and you'll see everything Gunicorn (or `wsgiref`) passes to your application. It demystifies a lot.

## The `start_response` Callable

`start_response` is a callable provided by the server. Your application calls it to set the response status and headers. Its signature:

```python
def start_response(
    status: str,              # e.g. "200 OK", "404 Not Found"
    response_headers: list,   # list of (name, value) tuples
    exc_info=None,            # for error handling, discussed below
) -> write_callable:          # legacy write callable, don't use this
```

The `status` string must be a valid HTTP status: three digits, a space, and a reason phrase. The reason phrase can be anything — the spec doesn't require it to be the canonical one — but convention is to use the standard phrases.

The `response_headers` is a list of `(name, value)` tuples. Names are case-insensitive in HTTP; convention is Title-Case. Values must be strings.

```python
# Valid calls to start_response
start_response("200 OK", [
    ("Content-Type", "text/html; charset=utf-8"),
    ("X-Custom-Header", "my-value"),
])

start_response("404 Not Found", [
    ("Content-Type", "text/plain"),
])

start_response("302 Found", [
    ("Location", "https://example.com/new-url"),
    ("Content-Type", "text/plain"),
])
```

**The `write` callable** that `start_response` returns is a legacy escape hatch for applications that need to write response data before returning from the callable. Don't use it in new code. The spec includes it for backward compatibility with pre-WSGI CGI-style code.

## The Return Value

Your application must return an iterable of byte strings. Each item in the iterable is a chunk of the response body. The server will concatenate and send them.

```python
# All of these are valid return values:

return [b"Hello, world!"]                         # Single chunk
return [b"Hello, ", b"world!"]                    # Multiple chunks
return iter([b"chunk 1", b"chunk 2"])             # Iterator
return (b"x" for x in range(3))                  # Generator

# For streaming responses, a generator is useful:
def streaming_app(environ, start_response):
    start_response("200 OK", [("Content-Type", "text/plain")])
    def generate():
        for i in range(100):
            yield f"Line {i}\n".encode("utf-8")
    return generate()
```

One important constraint: **you must call `start_response` before (or while) the server is consuming your return iterable**. In practice, call it before you return. The server will call `next()` on your iterable to get chunks, and by that point it needs to know the status and headers.

## The `close()` Method

If your return iterable has a `close()` method, the server will call it when it's done — even if an exception occurred during iteration. This is how you ensure cleanup (open file handles, database connections, etc.) happens even when the response is only partially sent.

```python
class FileResponse:
    def __init__(self, filepath):
        self.f = open(filepath, "rb")

    def __iter__(self):
        while chunk := self.f.read(8192):
            yield chunk

    def close(self):
        self.f.close()  # Server will call this


def file_serving_app(environ, start_response):
    path = environ['PATH_INFO'].lstrip('/')
    response = FileResponse(path)
    start_response("200 OK", [("Content-Type", "application/octet-stream")])
    return response
```

## Error Handling with `exc_info`

If an error occurs after `start_response` has been called (and headers may have been sent), you can call `start_response` again with `exc_info` set. This is how middleware propagates exceptions:

```python
import sys

def application(environ, start_response):
    try:
        # ... do work ...
        start_response("200 OK", [("Content-Type", "text/plain")])
        return [b"OK"]
    except Exception:
        start_response("500 Internal Server Error",
                       [("Content-Type", "text/plain")],
                       sys.exc_info())  # Pass exception info
        return [b"Internal Server Error"]
```

If headers haven't been sent yet, the server will use the new status/headers. If headers have already been sent (which can happen with streaming responses), the server will re-raise the exception — there's nothing else it can do at that point.

## What the Server Side Looks Like

To fully understand the contract, it helps to see what the server-side caller looks like. Here's a minimal version:

```python
def call_wsgi_app(app, environ):
    """
    Call a WSGI app and collect the response.
    Returns (status, headers, body_bytes).
    """
    response_started = []

    def start_response(status, headers, exc_info=None):
        if exc_info:
            try:
                if response_started:
                    raise exc_info[1].with_traceback(exc_info[2])
            finally:
                exc_info = None
        response_started.append((status, headers))

    result = app(environ, start_response)

    try:
        body = b"".join(result)
    finally:
        if hasattr(result, "close"):
            result.close()

    status, headers = response_started[0]
    return status, headers, body
```

This is what Gunicorn's worker essentially does — before sending it back over the socket as HTTP bytes. Note how `start_response` just stores the status and headers; the actual sending happens after `app()` returns.

## The Complete Spec, Annotated

Here's the one-page summary of everything WSGI requires:

```
APPLICATION:
- Must be callable
- Takes two arguments: environ (dict), start_response (callable)
- Must call start_response(status, headers) exactly once
  (or on error, may call it again with exc_info)
- Must return an iterable of byte strings
- The iterable may have a close() method; if so, server must call it

ENVIRON:
- Must contain CGI/1.1 variables (REQUEST_METHOD, PATH_INFO, etc.)
- Must contain wsgi.input (readable file-like object for body)
- Must contain wsgi.errors (writable file-like for errors)
- Must contain wsgi.version (1, 0)
- Must contain wsgi.url_scheme ("http" or "https")
- Must contain wsgi.multithread, wsgi.multiprocess, wsgi.run_once (bools)
- HTTP headers: prefixed with HTTP_, hyphens→underscores, uppercased
  Exception: Content-Type and Content-Length have no HTTP_ prefix

START_RESPONSE:
- Takes status (str), response_headers (list of 2-tuples), exc_info (optional)
- Status format: "NNN Reason Phrase"
- Headers: list of (name, value) tuples, strings only
- Returns a write() callable (legacy; don't use)
- May be called again only if exc_info is provided

SERVER:
- Must call app(environ, start_response) to get response
- Must send status and headers before body
- Must call result.close() if it exists, even on error
- Must handle chunked responses (iterate over return value)
```

That's the contract. Two functions talking to each other with a well-defined interface. The server provides `environ` and `start_response`; your app provides the response.

In the next chapter, we'll write a real WSGI application — no `wsgiref`, no framework, just the spec and a server call.

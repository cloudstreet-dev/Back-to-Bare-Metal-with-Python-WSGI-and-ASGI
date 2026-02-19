# Request and Response Objects (DIY Edition)

Working directly with `environ` and `start_response` gets old. Not because they're wrong — they're a fine low-level interface — but because nobody wants to write `environ.get('HTTP_AUTHORIZATION', '').split(' ')[-1]` every time they need a Bearer token.

Request and Response objects are wrappers. They take the raw WSGI interface and present it through a more convenient API. In this chapter, we'll build both.

## The Request Object

What do we actually want from a request object? Looking at how we've been using `environ`:

```python
# Things we access constantly:
method = environ['REQUEST_METHOD']
path = environ['PATH_INFO']
query_string = environ['QUERY_STRING']
content_type = environ.get('CONTENT_TYPE', '')
body = environ['wsgi.input'].read(int(environ.get('CONTENT_LENGTH') or 0))
host = environ.get('HTTP_HOST', '')
auth = environ.get('HTTP_AUTHORIZATION', '')

# Things we compute repeatedly:
params = urllib.parse.parse_qs(query_string)
json_body = json.loads(body)
```

Let's wrap all of this:

```python
import json
import urllib.parse
from typing import Any, Dict, List, Optional


class Request:
    """
    A wrapper around a WSGI environ dict.
    Provides convenient access to request data.
    """

    def __init__(self, environ: dict):
        self._environ = environ
        self._body: Optional[bytes] = None  # Cached, body can only be read once

    # ── Basic properties ──────────────────────────────────────────────────

    @property
    def method(self) -> str:
        return self._environ['REQUEST_METHOD'].upper()

    @property
    def path(self) -> str:
        return self._environ.get('PATH_INFO', '/')

    @property
    def query_string(self) -> str:
        return self._environ.get('QUERY_STRING', '')

    @property
    def scheme(self) -> str:
        return self._environ.get('wsgi.url_scheme', 'http')

    @property
    def host(self) -> str:
        return self._environ.get('HTTP_HOST', self._environ.get('SERVER_NAME', ''))

    @property
    def url(self) -> str:
        """The full request URL."""
        url = f"{self.scheme}://{self.host}{self.path}"
        if self.query_string:
            url += f"?{self.query_string}"
        return url

    # ── Headers ───────────────────────────────────────────────────────────

    def get_header(self, name: str, default: Optional[str] = None) -> Optional[str]:
        """
        Get a request header by name (case-insensitive).
        Handles the HTTP_ prefix and Content-Type/Content-Length exceptions.
        """
        name_lower = name.lower()
        if name_lower in ('content-type', 'content-length'):
            key = name_lower.replace('-', '_').upper()
        else:
            key = 'HTTP_' + name_lower.replace('-', '_').upper()
        return self._environ.get(key, default)

    @property
    def content_type(self) -> str:
        return self._environ.get('CONTENT_TYPE', '')

    @property
    def content_length(self) -> int:
        try:
            return int(self._environ.get('CONTENT_LENGTH') or 0)
        except ValueError:
            return 0

    @property
    def authorization(self) -> Optional[str]:
        return self.get_header('Authorization')

    @property
    def bearer_token(self) -> Optional[str]:
        """Extract Bearer token from Authorization header."""
        auth = self.authorization
        if auth and auth.startswith('Bearer '):
            return auth[7:]
        return None

    # ── Query parameters ──────────────────────────────────────────────────

    @property
    def query_params(self) -> Dict[str, List[str]]:
        """Parsed query string as dict of name → list of values."""
        return urllib.parse.parse_qs(self.query_string, keep_blank_values=True)

    def query(self, name: str, default: Optional[str] = None) -> Optional[str]:
        """Get the first value of a query parameter."""
        values = self.query_params.get(name)
        return values[0] if values else default

    def query_list(self, name: str) -> List[str]:
        """Get all values of a query parameter."""
        return self.query_params.get(name, [])

    # ── Body ──────────────────────────────────────────────────────────────

    @property
    def body(self) -> bytes:
        """Read and cache the request body."""
        if self._body is None:
            if self.content_length > 0:
                self._body = self._environ['wsgi.input'].read(self.content_length)
            else:
                self._body = b''
        return self._body

    @property
    def text(self) -> str:
        """Request body decoded as UTF-8 text."""
        return self.body.decode('utf-8')

    def json(self) -> Any:
        """Parse request body as JSON. Raises ValueError on failure."""
        return json.loads(self.body)

    def form(self) -> Dict[str, List[str]]:
        """Parse application/x-www-form-urlencoded body."""
        return urllib.parse.parse_qs(self.body.decode('utf-8'), keep_blank_values=True)

    # ── Path parameters (set by router) ───────────────────────────────────

    @property
    def path_params(self) -> Dict[str, str]:
        """Path parameters extracted by the router."""
        return self._environ.get('route.params', {})

    def path_param(self, name: str, default: Optional[str] = None) -> Optional[str]:
        return self.path_params.get(name, default)

    # ── Convenience ───────────────────────────────────────────────────────

    @property
    def is_json(self) -> bool:
        return 'application/json' in self.content_type

    @property
    def is_form(self) -> bool:
        return 'application/x-www-form-urlencoded' in self.content_type

    def __repr__(self) -> str:
        return f"<Request {self.method} {self.path}>"
```

## The Response Object

The response side is about building the thing we return from WSGI handlers. The raw interface requires:

1. Calling `start_response(status, headers)`
2. Returning an iterable of bytes

A Response object collects status, headers, and body, then handles the WSGI mechanics:

```python
from typing import Callable, Dict, Iterable, List, Optional, Tuple, Union


class Response:
    """
    A WSGI response builder.

    Usage:
        def handler(environ, start_response):
            response = Response("Hello, world!", content_type="text/plain")
            return response(environ, start_response)
    """

    def __init__(
        self,
        body: Union[str, bytes, None] = None,
        status: int = 200,
        content_type: str = "text/plain; charset=utf-8",
        headers: Optional[Dict[str, str]] = None,
    ):
        self.status_code = status
        self.headers: Dict[str, str] = {"Content-Type": content_type}
        if headers:
            self.headers.update(headers)

        if body is None:
            self._body = b""
        elif isinstance(body, str):
            self._body = body.encode("utf-8")
        else:
            self._body = body

    @property
    def status_line(self) -> str:
        phrases = {
            200: "OK", 201: "Created", 204: "No Content",
            301: "Moved Permanently", 302: "Found", 304: "Not Modified",
            400: "Bad Request", 401: "Unauthorized", 403: "Forbidden",
            404: "Not Found", 405: "Method Not Allowed",
            409: "Conflict", 415: "Unsupported Media Type",
            422: "Unprocessable Entity",
            500: "Internal Server Error", 503: "Service Unavailable",
        }
        phrase = phrases.get(self.status_code, "Unknown")
        return f"{self.status_code} {phrase}"

    def set_header(self, name: str, value: str) -> "Response":
        self.headers[name] = value
        return self

    def set_cookie(
        self,
        name: str,
        value: str,
        max_age: Optional[int] = None,
        path: str = "/",
        http_only: bool = True,
        secure: bool = False,
        same_site: str = "Lax",
    ) -> "Response":
        cookie = f"{name}={value}; Path={path}; SameSite={same_site}"
        if max_age is not None:
            cookie += f"; Max-Age={max_age}"
        if http_only:
            cookie += "; HttpOnly"
        if secure:
            cookie += "; Secure"
        # Multiple Set-Cookie headers — requires special handling
        self.headers.setdefault("_cookies", "")
        self._cookies = getattr(self, "_cookies", [])
        self._cookies.append(cookie)
        return self

    def __call__(self, environ: dict, start_response: Callable) -> List[bytes]:
        """Make Response a WSGI-compatible callable."""
        body = self._body
        headers = list(self.headers.items())

        # Add Content-Length if not present
        if "Content-Length" not in self.headers:
            headers.append(("Content-Length", str(len(body))))

        # Handle multiple Set-Cookie headers
        for cookie in getattr(self, "_cookies", []):
            headers.append(("Set-Cookie", cookie))

        start_response(self.status_line, headers)
        return [body] if body else []


# ── Factory functions ─────────────────────────────────────────────────────────

class JSONResponse(Response):
    def __init__(self, data: Any, status: int = 200, **kwargs):
        body = json.dumps(data, default=str).encode("utf-8")
        super().__init__(
            body=body,
            status=status,
            content_type="application/json",
            **kwargs,
        )


class HTMLResponse(Response):
    def __init__(self, html: str, status: int = 200, **kwargs):
        super().__init__(
            body=html,
            status=status,
            content_type="text/html; charset=utf-8",
            **kwargs,
        )


class RedirectResponse(Response):
    def __init__(self, location: str, permanent: bool = False):
        super().__init__(
            status=301 if permanent else 302,
            headers={"Location": location},
        )


class EmptyResponse(Response):
    def __init__(self, status: int = 204):
        super().__init__(status=status)
```

## Adapting the Router to Use Request/Response

Now let's update the router from the previous chapter to work with our new objects. We'll add a thin adapter that converts between the WSGI interface and Request/Response objects:

```python
from typing import Callable


HandlerFunc = Callable[[Request], Response]


def wsgi_handler(func: HandlerFunc) -> Callable:
    """
    Decorator: wraps a Request→Response function into a WSGI handler.
    Injects a Request object and calls the Response.
    """
    def wrapper(environ: dict, start_response: Callable) -> list:
        request = Request(environ)
        response = func(request)
        return response(environ, start_response)
    return wrapper


# Now our route handlers look like:

router = Router()

@router.get("/tasks/{task_id}")
@wsgi_handler
def get_task(request: Request) -> Response:
    task_id = request.path_param("task_id")
    task = tasks.get(task_id)
    if task is None:
        return JSONResponse({"error": "not found"}, status=404)
    return JSONResponse(task)


@router.post("/tasks")
@wsgi_handler
def create_task(request: Request) -> Response:
    if not request.is_json:
        return JSONResponse({"error": "content-type must be application/json"}, 415)
    try:
        data = request.json()
    except ValueError:
        return JSONResponse({"error": "invalid JSON"}, 400)
    if "title" not in data:
        return JSONResponse({"error": "title is required"}, 400)
    task = {"id": str(uuid.uuid4()), "title": data["title"], "done": False}
    tasks[task["id"]] = task
    return JSONResponse(task, status=201)
```

Look at how readable this is compared to the raw WSGI version. Same underlying mechanism, much cleaner interface.

## Testing With Request and Response Objects

One of the biggest benefits of wrapping the WSGI interface: testability. We can construct Request objects with arbitrary environ dicts and inspect Response objects directly.

```python
# test_handlers.py
import io
import json


def make_request(
    method: str = "GET",
    path: str = "/",
    body: bytes = b"",
    content_type: str = "",
    headers: dict = None,
) -> Request:
    """Build a Request object for testing."""
    environ = {
        "REQUEST_METHOD": method,
        "PATH_INFO": path,
        "QUERY_STRING": "",
        "CONTENT_TYPE": content_type,
        "CONTENT_LENGTH": str(len(body)),
        "wsgi.input": io.BytesIO(body),
        "wsgi.errors": io.StringIO(),
        "wsgi.url_scheme": "http",
        "wsgi.version": (1, 0),
        "wsgi.multithread": False,
        "wsgi.multiprocess": False,
        "wsgi.run_once": False,
        "SERVER_NAME": "testserver",
        "SERVER_PORT": "80",
        "HTTP_HOST": "testserver",
    }
    if headers:
        for name, value in headers.items():
            key = "HTTP_" + name.upper().replace("-", "_")
            environ[key] = value
    return Request(environ)


def call_handler(handler, request: Request) -> Response:
    """Call a WSGI handler and return a Response-like object."""
    responses = []

    def start_response(status, headers, exc_info=None):
        responses.append((status, headers))

    body_chunks = handler(request._environ, start_response)
    body = b"".join(body_chunks)

    status_str, headers_list = responses[0]
    status_code = int(status_str.split(" ")[0])
    response = Response(body=body, status=status_code)
    for name, value in headers_list:
        response.set_header(name, value)
    return response


# Tests
def test_create_task():
    payload = json.dumps({"title": "test task"}).encode()
    request = make_request("POST", "/tasks", body=payload,
                           content_type="application/json")
    request._environ["route.params"] = {}

    response = call_handler(create_task.__wrapped__, request)
    # Note: __wrapped__ gets past our @wsgi_handler decorator

    assert response.status_code == 201
    data = json.loads(response._body)
    assert data["title"] == "test task"
    assert "id" in data
    assert data["done"] is False


def test_get_nonexistent_task():
    request = make_request("GET", "/tasks/nonexistent")
    request._environ["route.params"] = {"task_id": "nonexistent"}

    response = call_handler(get_task.__wrapped__, request)
    assert response.status_code == 404


if __name__ == "__main__":
    test_create_task()
    test_get_nonexistent_task()
    print("All tests passed.")
```

## What Frameworks Add on Top

Django's `HttpRequest` object does exactly what we've done, plus:

- Multipart form parsing (file uploads)
- Cookie parsing
- Session integration (request.session)
- User authentication (request.user)
- Per-request caching

Flask's `Request` (from Werkzeug) adds:

- `request.files` for uploads
- `request.cookies`
- `request.args` (query params)
- `request.form` with multi-dict behavior
- JSON parsing with error handling

FastAPI's request handling goes further: it automatically converts JSON bodies to Pydantic models based on your function's type annotations.

But all of these are the same thing we built, with more edge cases handled and more convenience methods added. The foundation is identical: wrap `environ` in a nicer API, provide a `Response` class that speaks WSGI.

## Closing the Loop

We now have all the pieces of a working web framework:

- **Server** (previous chapter): accepts connections, calls WSGI apps
- **Router** (previous chapter): dispatches to handlers based on method + path
- **Middleware** (two chapters ago): composable wrappers for cross-cutting concerns
- **Request/Response** (this chapter): a clean interface for handlers

In the patterns section, we'll assemble these into a small but complete framework. First, we need to understand why WSGI has limits — and what ASGI does about them.

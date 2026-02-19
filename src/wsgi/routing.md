# Routing Without a Framework (It's Just String Matching)

URL routing sounds like one of those problems that requires a framework. It doesn't. Routing is pattern matching on a string. The string is `PATH_INFO`. The patterns are either exact matches, prefix matches, or regular expressions. Everything else is sugar.

Let's build a router that would be genuinely usable in a small project.

## What a Router Does

A router maps incoming requests (method + path) to handler functions. Given:

```
GET /users/42/posts
```

It should find the handler registered for `GET /users/{user_id}/posts` and call it with `user_id="42"`.

The three parts of routing:
1. **Registration**: associate a pattern with a handler
2. **Matching**: find the pattern that matches the incoming path
3. **Extraction**: pull path parameters out of the matched URL

## Naive Routing: Just `if` Statements

We already saw this in the tasks app. It scales to about 5 routes before it becomes unpleasant:

```python
def application(environ, start_response):
    method = environ['REQUEST_METHOD']
    path = environ['PATH_INFO']

    if path == '/' and method == 'GET':
        return index(environ, start_response)
    if path == '/users' and method == 'GET':
        return list_users(environ, start_response)
    # ... this gets old fast
```

Let's do better.

## A Dict-Based Exact Router

For applications that only need exact path matching (no parameters), a dict is fast and readable:

```python
from typing import Callable, Dict, Optional, Tuple


# Type: maps (method, path) to handler
RouteTable = Dict[Tuple[str, str], Callable]


def make_exact_router(routes: RouteTable) -> Callable:
    """
    Build a WSGI app from a dict of (method, path) → handler mappings.
    Returns 404 for unmatched paths, 405 for wrong method on matched path.
    """
    # Build a set of known paths for 404 vs 405 distinction
    known_paths = {path for (_, path) in routes}

    def router(environ, start_response):
        method = environ['REQUEST_METHOD']
        path = environ['PATH_INFO']
        key = (method, path)

        if key in routes:
            return routes[key](environ, start_response)

        if path in known_paths:
            # Path exists but method is wrong
            allowed = [m for (m, p) in routes if p == path]
            start_response("405 Method Not Allowed", [
                ("Allow", ", ".join(allowed)),
                ("Content-Type", "text/plain"),
            ])
            return [b"Method Not Allowed"]

        start_response("404 Not Found", [("Content-Type", "text/plain")])
        return [b"Not Found"]

    return router


# Usage
def index(environ, start_response):
    start_response("200 OK", [("Content-Type", "text/plain")])
    return [b"Home"]

def about(environ, start_response):
    start_response("200 OK", [("Content-Type", "text/plain")])
    return [b"About"]

app = make_exact_router({
    ("GET", "/"): index,
    ("GET", "/about"): about,
})
```

## A Pattern Router with Path Parameters

For path parameters like `/users/{user_id}`, we need pattern matching. We'll convert path templates to regular expressions:

```python
import re
from dataclasses import dataclass
from typing import Any, Callable, Dict, List, Optional, Tuple


@dataclass
class Route:
    method: str
    pattern: re.Pattern
    handler: Callable
    param_names: List[str]


def compile_path(path_template: str) -> Tuple[re.Pattern, List[str]]:
    """
    Convert a path template like '/users/{user_id}/posts/{post_id}'
    to a compiled regex and a list of parameter names.

    Supported converters:
        {name}      → matches any non-slash characters
        {name:int}  → matches one or more digits
        {name:slug} → matches URL-safe characters (letters, digits, hyphens)
    """
    converters = {
        "str": r"[^/]+",
        "int": r"[0-9]+",
        "slug": r"[a-zA-Z0-9-]+",
    }

    param_names = []
    pattern = ""
    remaining = path_template

    for match in re.finditer(r"\{(\w+)(?::(\w+))?\}", path_template):
        # Add the literal part before this parameter
        start = match.start()
        pattern += re.escape(remaining[:start - (len(path_template) - len(remaining))])

        name = match.group(1)
        converter = match.group(2) or "str"
        param_names.append(name)
        pattern += f"(?P<{name}>{converters.get(converter, converters['str'])})"

        remaining = path_template[match.end():]

    # A cleaner approach using re.sub:
    param_names = []

    def replace_param(m):
        name = m.group(1)
        converter = m.group(2) or "str"
        param_names.append(name)
        return f"(?P<{name}>{converters.get(converter, converters['str'])})"

    regex_str = re.sub(r"\{(\w+)(?::(\w+))?\}", replace_param, path_template)
    regex_str = "^" + regex_str + "$"

    return re.compile(regex_str), param_names


class Router:
    """
    A WSGI router with path parameter support.

    Usage:
        router = Router()

        @router.route("GET", "/users/{user_id:int}")
        def get_user(environ, start_response):
            user_id = int(environ['route.params']['user_id'])
            ...
    """

    def __init__(self):
        self.routes: List[Route] = []

    def route(self, method: str, path: str):
        """Decorator to register a route handler."""
        def decorator(func: Callable) -> Callable:
            pattern, param_names = compile_path(path)
            self.routes.append(Route(
                method=method.upper(),
                pattern=pattern,
                handler=func,
                param_names=param_names,
            ))
            return func
        return decorator

    # Convenience methods
    def get(self, path: str):
        return self.route("GET", path)

    def post(self, path: str):
        return self.route("POST", path)

    def put(self, path: str):
        return self.route("PUT", path)

    def delete(self, path: str):
        return self.route("DELETE", path)

    def __call__(self, environ, start_response):
        """The router itself is a WSGI app."""
        method = environ['REQUEST_METHOD']
        path = environ['PATH_INFO']

        matched_routes = []

        for route in self.routes:
            match = route.pattern.match(path)
            if match:
                matched_routes.append((route, match))

        if not matched_routes:
            start_response("404 Not Found", [("Content-Type", "text/plain")])
            return [b"Not Found"]

        # Check if any match the method
        for route, match in matched_routes:
            if route.method == method:
                # Inject path params into environ
                environ['route.params'] = match.groupdict()
                return route.handler(environ, start_response)

        # Path matched but method didn't
        allowed = [r.method for r, _ in matched_routes]
        start_response("405 Method Not Allowed", [
            ("Allow", ", ".join(sorted(set(allowed)))),
            ("Content-Type", "text/plain"),
        ])
        return [b"Method Not Allowed"]
```

## Using the Router

```python
import json
import uuid

router = Router()
tasks = {}  # In-memory store


def json_response(start_response, data, status=200):
    phrases = {200: "OK", 201: "Created", 400: "Bad Request",
               404: "Not Found", 405: "Method Not Allowed"}
    body = json.dumps(data).encode("utf-8")
    start_response(f"{status} {phrases.get(status, 'Unknown')}", [
        ("Content-Type", "application/json"),
        ("Content-Length", str(len(body))),
    ])
    return [body]


def read_json_body(environ):
    try:
        length = int(environ.get("CONTENT_LENGTH") or 0)
    except ValueError:
        return None
    if not length:
        return None
    try:
        return json.loads(environ["wsgi.input"].read(length))
    except (json.JSONDecodeError, KeyError):
        return None


@router.get("/tasks")
def list_tasks(environ, start_response):
    return json_response(start_response, list(tasks.values()))


@router.post("/tasks")
def create_task(environ, start_response):
    data = read_json_body(environ)
    if not data or "title" not in data:
        return json_response(start_response, {"error": "title required"}, 400)
    task = {"id": str(uuid.uuid4()), "title": data["title"], "done": False}
    tasks[task["id"]] = task
    return json_response(start_response, task, 201)


@router.get("/tasks/{task_id}")
def get_task(environ, start_response):
    task_id = environ["route.params"]["task_id"]
    task = tasks.get(task_id)
    if task is None:
        return json_response(start_response, {"error": "not found"}, 404)
    return json_response(start_response, task)


@router.delete("/tasks/{task_id}")
def delete_task(environ, start_response):
    task_id = environ["route.params"]["task_id"]
    if task_id not in tasks:
        return json_response(start_response, {"error": "not found"}, 404)
    return json_response(start_response, tasks.pop(task_id))


@router.get("/users/{user_id:int}/tasks")
def user_tasks(environ, start_response):
    user_id = environ["route.params"]["user_id"]  # Already matched as digits
    # In a real app, filter by user_id
    return json_response(start_response, {
        "user_id": int(user_id),
        "tasks": list(tasks.values()),
    })


if __name__ == "__main__":
    from wsgiref.simple_server import make_server
    with make_server("127.0.0.1", 8000, router) as server:
        print("http://127.0.0.1:8000")
        server.serve_forever()
```

Test it:
```bash
curl -X POST http://localhost:8000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "understand routing"}'

curl http://localhost:8000/users/42/tasks
```

## Sub-Applications and Mounting

A more advanced pattern: route by path prefix and delegate to sub-applications.

```python
class Mount:
    """
    Mount WSGI sub-applications at path prefixes.

        app = Mount({
            "/api": api_app,
            "/admin": admin_app,
            "/": fallback_app,
        })
    """

    def __init__(self, mounts: Dict[str, Callable]):
        # Sort by prefix length descending so more specific paths match first
        self.mounts = sorted(
            mounts.items(),
            key=lambda item: len(item[0]),
            reverse=True,
        )

    def __call__(self, environ, start_response):
        path = environ.get("PATH_INFO", "/")

        for prefix, app in self.mounts:
            if path.startswith(prefix):
                # Strip the prefix and adjust PATH_INFO/SCRIPT_NAME
                environ = dict(environ)
                environ["SCRIPT_NAME"] = environ.get("SCRIPT_NAME", "") + prefix
                environ["PATH_INFO"] = path[len(prefix):] or "/"
                return app(environ, start_response)

        start_response("404 Not Found", [("Content-Type", "text/plain")])
        return [b"Not Found"]


# Example: mount API and docs under different prefixes
from wsgiref.simple_server import demo_app

app = Mount({
    "/api": router,        # Our task API
    "/": demo_app,         # Fallback
})
```

## What Flask's Router Actually Does

Flask uses Werkzeug's routing system, which is more sophisticated than ours — it handles URL encoding, trailing slash redirects, weighted route matching (so `/users/me` takes precedence over `/users/{id}`), and URL generation (reversing a route name to a URL).

But the core mechanism is the same: convert URL templates to regular expressions, match `PATH_INFO` against them, extract named groups as parameters. Werkzeug just handles more edge cases and has a more refined API for it.

FastAPI's router is Starlette's, which is built on the same principle. The `@app.get("/items/{item_id}")` decorator registers a pattern and a handler. When a request comes in, Starlette walks the route table looking for a match.

## The Trailing Slash Question

Should `/users/` and `/users` match the same route? Opinions vary. Flask defaults to treating them as different routes (and optionally redirecting one to the other). Django's `path()` function matches literally.

Our router matches literally too — `{path_template}` must exactly match, no trailing slash normalization. If you want both to work, register both:

```python
@router.get("/tasks")
@router.get("/tasks/")
def list_tasks(environ, start_response):
    ...
```

Or add a middleware that strips trailing slashes:

```python
def strip_trailing_slash(app):
    def wrapper(environ, start_response):
        path = environ.get("PATH_INFO", "/")
        if path != "/" and path.endswith("/"):
            environ = dict(environ)
            environ["PATH_INFO"] = path.rstrip("/")
        return app(environ, start_response)
    return wrapper
```

Routing is opinionated. Now you understand the opinions well enough to choose your own.

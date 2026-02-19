# Your First WSGI App (No Training Wheels)

Let's build something real. Not "hello world" (we did that in the introduction) — a genuinely useful WSGI application with multiple routes, request parsing, and proper response handling. All without a framework.

By the end of this chapter you'll have a working JSON API that you can run with Gunicorn. It won't be pretty. That's the point.

## The Problem

We're building a simple in-memory task management API. It will support:

- `GET /tasks` — list all tasks
- `POST /tasks` — create a task
- `GET /tasks/{id}` — get a specific task
- `DELETE /tasks/{id}` — delete a task

That's enough to demonstrate routing, request body parsing, path parameter extraction, and proper HTTP response semantics without drowning in incidental complexity.

## Reading the Request Body

The first thing most tutorials skip over: how do you read the request body in WSGI?

```python
def read_body(environ) -> bytes:
    """Read the request body from environ['wsgi.input']."""
    try:
        content_length = int(environ.get('CONTENT_LENGTH', 0) or 0)
    except (ValueError, TypeError):
        content_length = 0

    if content_length > 0:
        return environ['wsgi.input'].read(content_length)
    return b''
```

Why `or 0`? Because `CONTENT_LENGTH` might be an empty string (`""`), which `int()` can't convert. The `or 0` handles that case. Why the try/except? Because you can't fully trust incoming headers.

Why `read(content_length)` rather than just `read()`? The spec says `wsgi.input` may be a socket-backed stream. Calling `read()` without a limit might block indefinitely waiting for a connection that never closes. Always read exactly `Content-Length` bytes.

## Parsing JSON Bodies

```python
import json
from typing import Any, Optional


def parse_json_body(environ) -> Optional[Any]:
    """Parse a JSON request body, returning None if absent or invalid."""
    content_type = environ.get('CONTENT_TYPE', '')
    if 'application/json' not in content_type:
        return None

    body = read_body(environ)
    if not body:
        return None

    try:
        return json.loads(body)
    except json.JSONDecodeError:
        return None
```

## Building Responses

Rather than calling `start_response` directly everywhere, let's build a small helper:

```python
def json_response(start_response, data: Any, status: int = 200) -> list[bytes]:
    """Send a JSON response."""
    STATUS_PHRASES = {
        200: "OK",
        201: "Created",
        400: "Bad Request",
        404: "Not Found",
        405: "Method Not Allowed",
        500: "Internal Server Error",
    }
    body = json.dumps(data, indent=2).encode("utf-8")
    phrase = STATUS_PHRASES.get(status, "Unknown")
    start_response(
        f"{status} {phrase}",
        [
            ("Content-Type", "application/json"),
            ("Content-Length", str(len(body))),
        ]
    )
    return [body]
```

## The Application

Now the actual application. Notice the structure: it's just a function that dispatches based on method and path.

```python
import json
import re
import uuid
from typing import Any, Optional


# In-memory store
tasks: dict[str, dict] = {}


def read_body(environ) -> bytes:
    try:
        content_length = int(environ.get('CONTENT_LENGTH', 0) or 0)
    except (ValueError, TypeError):
        content_length = 0
    if content_length > 0:
        return environ['wsgi.input'].read(content_length)
    return b''


def parse_json_body(environ) -> Optional[Any]:
    content_type = environ.get('CONTENT_TYPE', '')
    if 'application/json' not in content_type:
        return None
    body = read_body(environ)
    if not body:
        return None
    try:
        return json.loads(body)
    except json.JSONDecodeError:
        return None


STATUS_PHRASES = {
    200: "OK", 201: "Created", 400: "Bad Request",
    404: "Not Found", 405: "Method Not Allowed",
}


def json_response(start_response, data: Any, status: int = 200) -> list[bytes]:
    body = json.dumps(data, indent=2).encode("utf-8")
    phrase = STATUS_PHRASES.get(status, "Unknown")
    start_response(
        f"{status} {phrase}",
        [("Content-Type", "application/json"),
         ("Content-Length", str(len(body)))]
    )
    return [body]


def application(environ, start_response):
    method = environ['REQUEST_METHOD']
    path = environ['PATH_INFO']

    # Route: GET /tasks
    if path == '/tasks' and method == 'GET':
        return json_response(start_response, list(tasks.values()))

    # Route: POST /tasks
    if path == '/tasks' and method == 'POST':
        data = parse_json_body(environ)
        if not data or 'title' not in data:
            return json_response(start_response,
                                 {"error": "title is required"}, 400)
        task = {
            "id": str(uuid.uuid4()),
            "title": data['title'],
            "done": False,
        }
        tasks[task['id']] = task
        return json_response(start_response, task, 201)

    # Route: /tasks/{id}
    match = re.fullmatch(r'/tasks/([^/]+)', path)
    if match:
        task_id = match.group(1)

        if method == 'GET':
            task = tasks.get(task_id)
            if task is None:
                return json_response(start_response,
                                     {"error": "not found"}, 404)
            return json_response(start_response, task)

        if method == 'DELETE':
            if task_id not in tasks:
                return json_response(start_response,
                                     {"error": "not found"}, 404)
            deleted = tasks.pop(task_id)
            return json_response(start_response, deleted)

        return json_response(start_response,
                             {"error": "method not allowed"}, 405)

    # 404 for everything else
    return json_response(start_response, {"error": "not found"}, 404)


if __name__ == '__main__':
    from wsgiref.simple_server import make_server
    print("Serving on http://127.0.0.1:8000")
    with make_server('127.0.0.1', 8000, application) as server:
        server.serve_forever()
```

Save this as `tasks_app.py` and run it:

```bash
python tasks_app.py
```

Or with Gunicorn:

```bash
pip install gunicorn
gunicorn tasks_app:application
```

## Trying It Out

```bash
# Create a task
curl -X POST http://localhost:8000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Learn WSGI"}'
# {"id": "abc-123", "title": "Learn WSGI", "done": false}

# List tasks
curl http://localhost:8000/tasks
# [{"id": "abc-123", "title": "Learn WSGI", "done": false}]

# Get a specific task
curl http://localhost:8000/tasks/abc-123
# {"id": "abc-123", "title": "Learn WSGI", "done": false}

# Delete a task
curl -X DELETE http://localhost:8000/tasks/abc-123
# {"id": "abc-123", "title": "Learn WSGI", "done": false}

# Missing title
curl -X POST http://localhost:8000/tasks \
  -H "Content-Type: application/json" \
  -d '{"description": "oops"}'
# {"error": "title is required"}

# Not found
curl http://localhost:8000/tasks/nonexistent
# {"error": "not found"}
```

It works. No framework involved. The routing is regex matching on `PATH_INFO`. The request parsing is reading from `wsgi.input`. The responses are byte strings with proper headers.

## What This Reveals About Frameworks

Look at our `application` function and notice what's getting tedious:

1. **Routing**: `if path == '/tasks' and method == 'GET'` — this will get unreadable fast
2. **Response construction**: `json_response(start_response, data, status)` — we're passing `start_response` everywhere
3. **Request parsing**: `parse_json_body(environ)` — repeated on every endpoint that accepts a body
4. **Error handling**: every route independently returns error responses

A framework solves these problems. Flask gives you `@app.route('/tasks', methods=['GET'])`. Django gives you URL patterns and view functions. FastAPI gives you type annotations and automatic parsing.

But now you know what they're solving. The `@app.route` decorator is just adding your function to a routing table and wrapping it so it conforms to the WSGI interface. The request object (`flask.request`, `django.request`) is just a wrapper around `environ`. The response class is a wrapper around `start_response` and the return value.

It's convenience all the way down.

## The `wsgiref` Module

Python's standard library includes `wsgiref`, a reference implementation of a WSGI server:

```python
from wsgiref.simple_server import make_server
from wsgiref.validate import validator

# Wrap your app with the validator to catch WSGI spec violations
validated_app = validator(application)

with make_server('127.0.0.1', 8000, validated_app) as server:
    server.serve_forever()
```

`wsgiref.validate.validator` wraps your application and checks that it correctly implements the WSGI spec — proper return types, calling `start_response` at the right time, etc. Use it during development; remove it for production.

`wsgiref.simple_server` is not production-ready (it's single-threaded and handles one request at a time), but it's useful for local development when you want zero dependencies.

## On State and Concurrency

The `tasks` dictionary in our app is module-level state. This works fine for a single-process server, but Gunicorn's default is to use multiple worker processes. Each worker has its own copy of the module, its own `tasks` dict. Changes in worker 1 are invisible to worker 2.

For production, you'd use a database or shared cache (Redis) instead of in-memory state. This isn't a WSGI limitation — it's just how multi-process architectures work. But it's worth being explicit about: WSGI does not share state between requests. Each call to `application()` is independent.

This is actually a feature. It makes WSGI applications easy to reason about and easy to scale horizontally. Stateless by default.

## Exercises

Before moving on, try these modifications to the app:

1. Add a `PATCH /tasks/{id}` endpoint that updates the `done` field
2. Add proper `Content-Type` validation — return 415 if it's not `application/json` on POST
3. Add a request logger: print method, path, and status code for every request
4. Handle `PATCH /tasks/{id}` — change the `done` field

Exercise 3 is a preview of the next chapter. The logging belongs in middleware.

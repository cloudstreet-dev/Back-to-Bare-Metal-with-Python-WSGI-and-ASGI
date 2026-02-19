# The Lie You've Been Living

You've been writing Python web applications for — let's say — a while. You know how to define routes. You know how to write views. You know that `request.method` gives you `"GET"` or `"POST"`, and you know that returning a `Response` object makes things appear in someone's browser.

What you may not know is what any of that actually *is*.

Here is the thing that your framework would prefer you not examine too closely: it's a function. The entire web application you've been building — the routing, the middleware, the template rendering, the session handling, the authentication system, the REST API — all of it ultimately compiles down to a Python callable that takes some arguments and returns something.

That's it. That's the whole trick.

## Let's Prove It Right Now

Here is a complete, functional web application that will run in production:

```python
def application(environ, start_response):
    status = "200 OK"
    headers = [("Content-Type", "text/plain")]
    start_response(status, headers)
    return [b"Hello, world"]
```

Save this as `app.py`. Install gunicorn (`pip install gunicorn`). Run:

```bash
gunicorn app:application
```

You now have a production web server serving HTTP requests. No framework. No dependencies beyond gunicorn. No magic.

If you point your browser at `http://localhost:8000`, you'll see "Hello, world".

That function — `application` — is a WSGI application. Everything Django and Flask have ever done starts from exactly this interface.

## The Moment of Recognition

Now look at Django. From `django/core/handlers/wsgi.py`:

```python
class WSGIHandler(base.BaseHandler):
    request_class = WSGIRequest

    def __call__(self, environ, start_response):
        set_script_prefix(get_script_name(environ))
        signals.request_started.send(sender=self.__class__, environ=environ)
        request = self.request_class(environ)
        response = self.get_response(request)
        # ... headers, status ...
        start_response(status, response_headers)
        # ...
        return response
```

It's `__call__`. Django's entire web framework is a class with a `__call__` method that takes `environ` and `start_response`. It is, by definition, a callable that implements the WSGI interface.

Every piece of Django — the ORM, the admin, the URL dispatcher, the template engine — exists to produce that callable. The callable is the product.

Flask? Same thing:

```python
class Flask(App):
    def __call__(self, environ, start_response):
        return self.wsgi_app(environ, start_response)
```

FastAPI runs on top of Starlette, which is ASGI (we'll get to that). But strip it down and you find the same idea: a callable with a defined interface.

## Why This Matters

If you understand that your framework is a callable with a specific signature, several things become clear:

**Testing becomes obvious.** Your app is a function. Call it with the right arguments and inspect the result. No magic test client needed — though those are useful too.

**Middleware makes sense.** Middleware is a callable that takes a callable and returns a callable. It's function composition. It's wrappers. Once you see this, the middleware stack is just a chain of decorators with extra steps.

**The framework is not special.** It's solving real problems — routing, request parsing, response serialization — but it's doing so with the same Python you write every day. There's no privileged access, no hidden C extensions doing the real work (well, sometimes there are C extensions, but not for routing). It's just code.

**Debugging gets easier.** When something goes wrong at the framework level, you now have a mental model of where to look. The request came in. It hit the WSGI callable. Something happened between `environ` and `start_response`. You can trace it.

## The Interfaces, Briefly

There are two specs we care about in this book.

**WSGI** (Web Server Gateway Interface, PEP 3333) is the synchronous interface. It's been around since 2003. Every line of Python web code written before async became mainstream runs on top of it. The entire spec is essentially:

```
application(environ, start_response) -> iterable of bytes
```

**ASGI** (Asynchronous Server Gateway Interface) is the async successor. It was designed to handle things WSGI can't — WebSockets, long-polling, HTTP/2 push — by making the entire interface async. The spec is:

```
application(scope, receive, send) -> None  # but async
```

Both specs define a contract between a *web server* (Gunicorn, Uvicorn, Hypercorn) and a *web application* (your code, or Django, or FastAPI). The server handles the TCP connection, parses the HTTP request, and calls your callable. Your callable decides what to return. The server sends it back.

The framework just makes it easier to write that callable. That's the whole job.

## What This Book Will Do

We're going to start at the bottom.

In Part I, we'll look at what HTTP actually is (text over a socket), what the frameworks are doing, and why understanding this matters for your day-to-day work.

In Part II, we'll implement WSGI from first principles: a server, middleware, routing, and request/response abstractions — all from scratch.

In Part III, we'll do the same for ASGI: the async model, WebSockets, lifespan events, and building an async server.

In Part IV, we'll look at patterns — testing, middleware composition, and building a small framework — to solidify everything.

By the end, you'll be able to read the Gunicorn source code and understand what it's doing. You'll know what Uvicorn's `main()` actually does. You'll be able to debug framework-level issues because you'll have written the framework-level code yourself.

More importantly, you'll look at your next Django application and see it for what it is: a callable. A very sophisticated, well-tested, production-hardened callable — but a callable nonetheless.

The magic was just Python with good variable names.

Let's start with the protocol.

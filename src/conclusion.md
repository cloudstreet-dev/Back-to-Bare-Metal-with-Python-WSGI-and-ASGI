# You Know Too Much Now (What to Do With It)

You started this book believing — or at least accepting on faith — that web frameworks were doing something fundamentally complex. Something that required expertise and abstraction to approach safely. Something you didn't need to understand to use.

That belief, it turns out, was based on a lie of omission. Not a malicious one. Frameworks genuinely are complex in the engineering sense — they handle thousands of edge cases, they're battle-tested against adversarial input, they've accumulated years of hard-won knowledge about what goes wrong in production. None of that complexity is wasted.

But the *interface* they expose to the world is just a callable. A function. The whole thing.

## What You've Built

Over the course of this book, you built:

- **A raw HTTP parser** that turns TCP bytes into structured request data
- **A WSGI server** that accepts connections, builds `environ`, and calls applications
- **WSGI middleware** for logging, authentication, CORS, timing, and request IDs
- **A URL router** with path parameter extraction using compiled regular expressions
- **Request and Response classes** that wrap the WSGI interface with a clean API
- **An ASGI server** using `asyncio.start_server` with proper event-based flow
- **ASGI middleware** with request/response interception and lifespan support
- **A WebSocket chat server** using the full ASGI WebSocket protocol
- **A test harness** for both WSGI and ASGI applications
- **A complete mini-framework** with routing, lifespan, WebSockets, and middleware

Each of these was built from first principles, consulting the spec rather than copying from an existing library. None of them are production-ready in the sense that Gunicorn or Uvicorn is production-ready — that would take more than a book — but all of them are *correct*, which is what matters for understanding.

## What This Changes

The practical impact of understanding your tools at this level is subtle but real.

**Debugging gets easier.** When Django throws an error in `WSGIHandler.__call__`, you know what that is. When Uvicorn logs a warning about a malformed request, you understand the parsing step it's complaining about. When FastAPI returns a 422 on a request you think is valid, you can trace it through the parameter extraction code you now understand.

**Performance becomes legible.** WSGI workers handle one request at a time per thread — you know why, because you built a synchronous server. ASGI handles many requests per event loop iteration — you know why, because you built an async server. When someone says "add more Gunicorn workers," you know what that means at the socket level.

**Middleware composition is obvious.** You'll never again be confused about the order your middleware runs in, because you've implemented `build_middleware_stack` yourself and seen how `reversed(middleware)` produces the right wrapping order.

**Testing is straightforward.** Your app is a callable. Call it with a fake environ or scope. Inspect the result. This is all your test client is doing, and now you know it.

**Choosing between WSGI and ASGI is rational.** You know what WSGI can't do (hold connections open, do async I/O efficiently) and why ASGI exists to address those limitations. The choice isn't "what's the new thing" — it's a decision based on your actual requirements.

## What You Should Still Use Frameworks For

Knowing how something works doesn't mean you should build it yourself for every project. The wheel is understood; you still don't build a new one for every car.

Use Django when you want the batteries: ORM, admin, migrations, auth. It's a well-engineered solution to a set of common problems, and "well-engineered" includes fifteen years of security patches.

Use FastAPI when you want async performance, type-annotated APIs, and automatic OpenAPI docs with minimal boilerplate. The type coercion and documentation generation are genuinely valuable and tedious to implement correctly.

Use Flask when you want something small and explicit, where you add only what you need.

Use the mini-framework from the last chapter when you want something you understand completely and can modify freely — for personal projects, for microservices with unusual requirements, for fun.

The right answer depends on context. Now you have enough context to make the decision rationally.

## What the Spec Documents Are For

You now have a reason to read them:

- **PEP 3333** (WSGI): [python.org/dev/peps/pep-3333](https://peps.python.org/pep-3333/) — the canonical reference for everything `environ`, `start_response`, and the response iterable contract
- **ASGI Spec**: [asgi.readthedocs.io](https://asgi.readthedocs.io) — scope types, event names, and the lifespan protocol
- **RFC 9110** (HTTP Semantics): the definitive reference for HTTP methods, status codes, and header semantics
- **RFC 6455** (WebSocket): the WebSocket protocol spec, including the handshake, frame format, and close codes

These aren't bedtime reading. They're reference documents. Now that you have a mental model of what they're specifying, they become useful rather than impenetrable.

## The Deeper Point

There's a broader principle at work here beyond Python web development.

Most of the complexity in software is accidental — it comes from accumulated decisions, backward compatibility constraints, and the need to handle edge cases that most applications never encounter. The essential complexity (the hard part that can't be simplified away) is usually much smaller.

The essential complexity of a web server is: read bytes, parse HTTP, call a callable, write bytes. Everything else is handling the cases where that simple description fails.

When you encounter a system that seems impossibly complex — a message broker, a container runtime, a database engine — the same approach applies: find the interface, build something that implements it, and watch the complexity become accidental rather than essential.

This book was about Python web protocols. The method is general.

## A Note on `bare`

The mini-framework we built in the last chapter is a teaching tool, not a production framework. If you found the exercise genuinely useful and want to continue building, consider:

- Adding proper error handling with custom exception classes
- Implementing dependency injection (FastAPI's killer feature is more tractable than it looks)
- Adding background task scheduling
- Implementing WebSocket rooms properly with asyncio queues
- Writing comprehensive tests for the framework itself

Or, more likely, go back to FastAPI or Starlette with a much better understanding of what they're doing and why.

## Thank You

This book exists because Georgiy Treyvus asked the right question at the right time. Building software is easier when someone asks "but what's *actually* happening?"

Go build something. You know too much to be mystified by it now.

---

*Back to Bare Metal: WSGI & ASGI for Python Developers*
*Published by CloudStreet — [github.com/cloudstreet-dev](https://github.com/cloudstreet-dev)*
*CC0 1.0 Universal — no rights reserved*

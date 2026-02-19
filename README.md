# Back to Bare Metal: WSGI & ASGI for Python Developers

> Django and FastAPI are just fancy callables. This book strips away the magic, shows you exactly what's happening under the hood, and teaches you to build Python web apps from first principles — synchronous with WSGI, async with ASGI. Know your tools. Really know them.

**Part of the [CloudStreet](https://github.com/cloudstreet-dev) series.**

*With special thanks to Georgiy Treyvus, whose idea inspired this book.*

---

## What This Book Is

Most Python web developers have a mental model of their framework that goes something like: _request goes in, magic happens, response comes out_. This book is an intervention.

WSGI is a Python callable with a specific signature. ASGI is an async Python callable with a specific signature. Django, Flask, FastAPI, Starlette — they're all just implementations of these specs. Once you see that, you can't unsee it. This book makes sure you see it.

By the end, you'll have written a WSGI server, an ASGI server, middleware from scratch, a router, request/response objects, and a small framework. You'll also understand why WebSockets broke WSGI and what ASGI actually solved.

---

## Table of Contents

### Part I: What You Think You Know

| Chapter | Description |
|---------|-------------|
| [The Lie You've Been Living](src/introduction.md) | Frameworks are just callables. Here's proof. |
| [HTTP Is Just Text](src/http-is-just-text.md) | You've been treating it like magic. It isn't. |
| [What Django and FastAPI Are Actually Doing](src/what-frameworks-do.md) | The curtain, pulled back. |

### Part II: WSGI

| Chapter | Description |
|---------|-------------|
| [The WSGI Spec](src/wsgi/the-spec.md) | It fits on a napkin. We'll use a whole chapter anyway. |
| [Your First WSGI App](src/wsgi/first-app.md) | No training wheels. |
| [Build a WSGI Server from Scratch](src/wsgi/server-from-scratch.md) | So you understand what Gunicorn is actually doing. |
| [Middleware: Turtles All the Way Down](src/wsgi/middleware.md) | Callables wrapping callables wrapping callables. |
| [Routing Without a Framework](src/wsgi/routing.md) | It's just string matching. |
| [Request and Response Objects (DIY Edition)](src/wsgi/request-response.md) | Build the abstractions yourself, once. |

### Part III: ASGI

| Chapter | Description |
|---------|-------------|
| [Why WSGI Can't Have Nice Things](src/asgi/why-not-wsgi.md) | WebSockets, async, and the fundamental problem. |
| [The ASGI Spec](src/asgi/the-spec.md) | scope, receive, send — that's literally it. |
| [Your First ASGI App](src/asgi/first-app.md) | Async from the ground up. |
| [Build an ASGI Server from Scratch](src/asgi/server-from-scratch.md) | asyncio, sockets, and HTTP/1.1. |
| [Lifespan Events](src/asgi/lifespan.md) | Startup, shutdown, and existential dread. |
| [WebSockets Over ASGI](src/asgi/websockets.md) | Finally, a reason to care. |

### Part IV: Patterns

| Chapter | Description |
|---------|-------------|
| [ASGI Middleware Deep Dive](src/patterns/middleware-asgi.md) | The async version of turtles all the way down. |
| [Testing Without a Framework](src/patterns/testing.md) | Your app is a callable. Test it like one. |
| [Roll Your Own Mini-Framework](src/patterns/roll-your-own.md) | For fun and genuine understanding. |

---

[You Know Too Much Now](src/conclusion.md)

---

## Building Locally

### Prerequisites

Install mdBook:

```bash
cargo install mdbook
```

Or download a binary from the [mdBook releases page](https://github.com/rust-lang/mdBook/releases).

### Build

```bash
mdbook build
```

The output will be in `book/`.

### Serve with live reload

```bash
mdbook serve
```

Then open [http://localhost:3000](http://localhost:3000).

---

## License

This book is released under [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/) — public domain. Use it however you like.

All code examples are also CC0. Copy them, modify them, ship them. No attribution required (though a nod is always appreciated).

---

## CloudStreet

This book is part of the CloudStreet series — technical books for developers who want to understand their tools at the level where the interesting things happen.

More at [github.com/cloudstreet-dev](https://github.com/cloudstreet-dev).

# Why WSGI Can't Have Nice Things

WSGI has been reliable for twenty years. It's simple, well-understood, and supported by every Python web server ever written. So why does ASGI exist?

The short answer is: WebSockets. The longer answer involves a fundamental mismatch between the WSGI request/response model and how modern web applications actually behave.

## The WSGI Model

WSGI assumes a specific shape for web communication:

```
Client sends request → Server parses it → Your app runs → App returns response → Done
```

This is the HTTP request/response cycle. One request in, one response out. The entire interaction fits in a single function call:

```python
response_iterable = application(environ, start_response)
```

When `application` returns, the transaction is complete. The connection can be closed. The worker is free to handle the next request.

This model works perfectly for 95% of HTTP traffic. GET a page, POST a form, fetch some JSON. One request, one response, move on.

The remaining 5% is where things fall apart.

## WebSockets: The Problem

WebSockets are a protocol for bidirectional communication over a persistent connection. Once a WebSocket handshake completes, *both* client and server can send messages at any time, for as long as the connection stays open. There's no concept of "one request, one response."

How would you model this in WSGI? You can't. Let's try anyway to see why:

```python
def websocket_app(environ, start_response):
    # At this point we want to:
    # 1. Complete the WebSocket upgrade handshake
    # 2. Read messages from the client
    # 3. Send messages to the client
    # 4. Keep the connection open indefinitely
    # 5. React to messages as they arrive
    #
    # But start_response must be called synchronously.
    # And we must return an iterable.
    # And when we return, the connection closes.
    #
    # There is no way to model "keep connection open, react to events"
    # within this interface.

    start_response("101 Switching Protocols", [...])
    return ???  # What do we return? How do we keep reading?
```

The WSGI model is inherently synchronous and single-turn. WebSockets are inherently asynchronous and multi-turn.

Some WSGI servers tried to hack around this with extensions. None of them were portable. The WebSocket support in Flask before ASGI required either `gevent` (monkey-patching) or a separate WebSocket server running alongside the WSGI app, with a reverse proxy in front. It was not elegant.

## Long Polling and Server-Sent Events

The same problem, milder:

**Long polling**: client sends request, server holds it open for up to 30 seconds until there's data to send, then responds. Works in WSGI (barely), but ties up a worker thread for the entire wait time.

**Server-Sent Events (SSE)**: server sends a stream of events to the client over a single HTTP connection. WSGI can technically do this by returning a generator, but the worker is tied up until the stream ends.

Both of these require *holding a connection open* while doing work asynchronously. WSGI's synchronous model handles this with blocking — tie up a thread per connection. This works but doesn't scale: 1000 concurrent SSE connections = 1000 blocked threads.

## The Async Problem

Python 3.4 introduced `asyncio`. By 2016, `async`/`await` was mainstream Python. Web frameworks wanted to write async handlers:

```python
async def get_user(request):
    user = await db.fetch_one("SELECT * FROM users WHERE id = ?", user_id)
    return JSONResponse(user)
```

This is genuinely better. The worker isn't blocked while waiting for the database — it can handle other requests. But you can't call an async function from a synchronous context without `asyncio.run()` or equivalent. And WSGI servers call your app synchronously:

```python
# This is what Gunicorn does. It's synchronous.
result = application(environ, start_response)
```

You can't stick `await` in there. A WSGI server cannot efficiently run async applications because the interface itself is synchronous.

Some solutions emerged:
- Run each WSGI request in a thread pool and bridge to asyncio
- Use gevent to make blocking calls look async
- Accept that you can't use `async/await` in WSGI handlers

None of these are satisfying. The synchronous interface was a genuine constraint.

## The HTTP/2 Problem

HTTP/2 multiplexes multiple requests over a single TCP connection. The server can push resources to the client before they're requested. Requests can be prioritized. All of this happens over a long-lived connection with multiple concurrent streams.

WSGI models each request as a separate function call with its own `environ`. It has no concept of a "connection" that persists across requests, no way to push data to the client, no way to handle multiple concurrent streams over a single connection.

HTTP/2 server push never really went anywhere (HTTP/3 dropped it), but the point stands: WSGI's model of "one function call = one request" is a fundamental constraint that HTTP/2's connection-level features can't fit into.

## What ASGI Changes

ASGI (Asynchronous Server Gateway Interface) solves these problems with a different model:

Instead of:
```python
# WSGI: one call, one response
response = app(environ, start_response)
```

ASGI uses:
```python
# ASGI: async, event-based, connection-aware
await app(scope, receive, send)
```

The differences:
- **`async`**: the application is an async callable, so Python's event loop can interleave multiple concurrent requests on a single thread
- **`scope`**: connection metadata (similar to `environ`) but includes the *type* of connection — HTTP, WebSocket, or lifespan
- **`receive`**: an async callable that your app calls to receive events (incoming messages, request body chunks, WebSocket messages)
- **`send`**: an async callable that your app calls to send events (response start, body chunks, WebSocket messages)

The event-based model means:
- WebSockets work natively: receive a message event, send a message event, repeat
- Server-sent events work without blocking: send body chunk events as data arrives
- HTTP/2 streams could work: each stream is a separate scope
- Lifespan events work: `startup` and `shutdown` events around the app's lifecycle

ASGI is more complex than WSGI. That's not an accident — it's handling more complex scenarios. But it's still just a callable. A more sophisticated callable, with a more sophisticated interface, but a callable.

## The Cost

Nothing is free. ASGI's complexity has real costs:

**Debugging is harder.** Async tracebacks are longer and less obvious. An exception in an async generator might surface somewhere unexpected.

**The mental model is different.** WSGI is easy to reason about: function in, function out. ASGI requires understanding coroutines, event loops, and the `receive`/`send` event model.

**Not everything is async.** Most database drivers, file I/O, and third-party libraries were written for synchronous use. In ASGI, calling a blocking function in a coroutine blocks the entire event loop. You need `asyncio.run_in_executor` or async libraries.

**You don't need it for standard HTTP.** If your application is 100% request/response with no WebSockets, no SSE, and no HTTP/2 push, WSGI is fine. Gunicorn on a few WSGI workers will serve you well.

But if you do need WebSockets, or you want to write genuinely async handlers that don't block on I/O, or you're building something that needs to hold many connections open simultaneously — ASGI is the right tool.

Let's look at the spec.

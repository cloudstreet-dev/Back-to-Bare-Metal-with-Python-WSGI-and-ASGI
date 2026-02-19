# WebSockets Over ASGI (Finally, a Reason to Care)

We've spent several chapters saying "WSGI can't do WebSockets." Now let's actually use them.

WebSockets are a persistent, bidirectional communication channel between client and server. The browser opens a connection that stays open. Either side can send messages at any time. This enables chat applications, real-time dashboards, live collaboration, games — any scenario where you need low-latency communication without the overhead of polling.

ASGI handles WebSockets natively. Let's see how.

## The WebSocket Handshake

A WebSocket connection starts as an HTTP request with special headers:

```
GET /ws HTTP/1.1
Host: localhost:8000
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

The server responds with:

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

After this exchange, the connection is no longer HTTP — it's a WebSocket stream. The `Sec-WebSocket-Accept` value is a specific hash of the client's key; browsers verify it to prevent cross-origin attacks.

In ASGI, the server handles this handshake. Your application just receives a `websocket` scope and sends a `websocket.accept` event.

## The Simplest WebSocket App

```python
async def application(scope, receive, send):
    if scope["type"] == "lifespan":
        await handle_lifespan(receive, send)
    elif scope["type"] == "http":
        await handle_http(scope, receive, send)
    elif scope["type"] == "websocket":
        await handle_websocket(scope, receive, send)


async def handle_websocket(scope, receive, send):
    """Echo server: sends back whatever the client sends."""

    # Wait for the connection event
    event = await receive()
    assert event["type"] == "websocket.connect"

    # Accept the connection
    await send({"type": "websocket.accept"})

    # Echo loop
    while True:
        event = await receive()

        if event["type"] == "websocket.receive":
            # Echo the message back
            if event.get("text"):
                await send({
                    "type": "websocket.send",
                    "text": f"Echo: {event['text']}",
                })
            elif event.get("bytes"):
                await send({
                    "type": "websocket.send",
                    "bytes": event["bytes"],
                })

        elif event["type"] == "websocket.disconnect":
            # Client disconnected — exit the handler
            break
```

Run it:

```bash
uvicorn ws_app:application
```

Test with a WebSocket client. Python's `websockets` library works well:

```python
# test_ws.py
import asyncio
import websockets


async def test():
    async with websockets.connect("ws://localhost:8000/ws") as ws:
        await ws.send("Hello, WebSocket!")
        response = await ws.recv()
        print(response)  # Echo: Hello, WebSocket!


asyncio.run(test())
```

Or from the browser console:

```javascript
const ws = new WebSocket("ws://localhost:8000/ws");
ws.onmessage = (e) => console.log(e.data);
ws.send("Hello from browser");
// Echo: Hello from browser
```

## A Real Example: Chat Room

An echo server demonstrates the protocol. A chat room demonstrates why you'd actually use it.

```python
import asyncio
import json
from typing import Dict, Set


# Connected clients: room_name → set of send callables
rooms: Dict[str, Set] = {}


async def handle_websocket(scope, receive, send):
    """Multi-room chat server."""
    # Get room from query string
    query = scope.get("query_string", b"").decode()
    room = "general"
    for param in query.split("&"):
        if param.startswith("room="):
            room = param[5:]
            break

    # Join the room
    if room not in rooms:
        rooms[room] = set()
    rooms[room].add(send)

    try:
        # Accept the WebSocket connection
        event = await receive()
        if event["type"] != "websocket.connect":
            return
        await send({"type": "websocket.accept"})

        # Announce arrival
        await broadcast(room, {
            "type": "system",
            "message": f"A user joined #{room}",
        }, exclude=send)

        # Message loop
        while True:
            event = await receive()

            if event["type"] == "websocket.receive":
                text = event.get("text", "")
                if not text:
                    continue

                try:
                    data = json.loads(text)
                except json.JSONDecodeError:
                    data = {"text": text}

                # Broadcast to everyone in the room
                await broadcast(room, {
                    "type": "message",
                    "text": data.get("text", text),
                    "room": room,
                })

            elif event["type"] == "websocket.disconnect":
                break

    finally:
        # Remove from room on disconnect
        rooms.get(room, set()).discard(send)
        await broadcast(room, {
            "type": "system",
            "message": f"A user left #{room}",
        })


async def broadcast(room: str, data: dict, exclude=None) -> None:
    """Send a message to all clients in a room."""
    message = json.dumps(data)
    dead_clients = set()

    for client_send in list(rooms.get(room, set())):
        if client_send is exclude:
            continue
        try:
            await client_send({
                "type": "websocket.send",
                "text": message,
            })
        except Exception:
            dead_clients.add(client_send)

    # Clean up dead clients
    rooms.get(room, set()).difference_update(dead_clients)
```

Test with multiple connections:

```python
# multi_client_test.py
import asyncio
import json
import websockets


async def client(name: str, room: str = "general"):
    async with websockets.connect(f"ws://localhost:8000/ws?room={room}") as ws:
        print(f"{name} connected")

        # Send a message
        await ws.send(json.dumps({"text": f"Hello from {name}"}))

        # Receive messages for 2 seconds
        async def receive_loop():
            while True:
                msg = await ws.recv()
                data = json.loads(msg)
                print(f"{name} received: {data}")

        try:
            await asyncio.wait_for(receive_loop(), timeout=2.0)
        except asyncio.TimeoutError:
            pass


async def main():
    # Connect three clients concurrently
    await asyncio.gather(
        client("Alice"),
        client("Bob"),
        client("Carol"),
    )


asyncio.run(main())
```

## Rejecting WebSocket Connections

Sometimes you need to reject a WebSocket connection — bad authentication, rate limit exceeded, room full:

```python
async def handle_websocket(scope, receive, send):
    # Wait for connect event
    event = await receive()
    assert event["type"] == "websocket.connect"

    # Check authentication
    token = get_token_from_scope(scope)
    if not await is_valid_token(token):
        # Reject the connection with a close code
        await send({
            "type": "websocket.close",
            "code": 4001,  # Application-defined code (4000-4999 are custom)
        })
        return

    # Accept and proceed
    await send({"type": "websocket.accept"})
    # ...


def get_token_from_scope(scope) -> str:
    """Extract Bearer token from WebSocket upgrade headers."""
    for name, value in scope.get("headers", []):
        if name == b"authorization":
            auth = value.decode("latin-1")
            if auth.startswith("Bearer "):
                return auth[7:]
    # Also check query string
    query = scope.get("query_string", b"").decode()
    for param in query.split("&"):
        if param.startswith("token="):
            return param[6:]
    return ""


async def is_valid_token(token: str) -> bool:
    # Check against database, JWT validation, etc.
    return token == "valid-token"  # Simplified
```

WebSocket close codes follow the [RFC 6455 spec](https://www.rfc-editor.org/rfc/rfc6455#section-7.4):
- `1000` — Normal closure
- `1001` — Going away (server shutdown)
- `1002` — Protocol error
- `1003` — Unsupported data type
- `4000-4999` — Application-defined (use these for your own status codes)

## Sending Binary Data

WebSockets support both text frames (UTF-8 encoded) and binary frames. Use binary for protocol buffers, binary file transfers, or any non-text data:

```python
async def binary_echo(scope, receive, send):
    event = await receive()
    assert event["type"] == "websocket.connect"
    await send({"type": "websocket.accept"})

    while True:
        event = await receive()
        if event["type"] == "websocket.receive":
            if event.get("bytes") is not None:
                # Echo binary data back
                await send({
                    "type": "websocket.send",
                    "bytes": event["bytes"],
                })
            elif event.get("text") is not None:
                # Convert text to binary (example)
                await send({
                    "type": "websocket.send",
                    "bytes": event["text"].encode("utf-8"),
                })
        elif event["type"] == "websocket.disconnect":
            break
```

## Handling Both HTTP and WebSocket

Real applications serve regular HTTP endpoints and WebSocket endpoints. Wire them together:

```python
async def application(scope, receive, send):
    if scope["type"] == "lifespan":
        await handle_lifespan(receive, send)
    elif scope["type"] == "http":
        await handle_http(scope, receive, send)
    elif scope["type"] == "websocket":
        path = scope["path"]
        if path == "/ws/chat":
            await handle_websocket(scope, receive, send)
        elif path == "/ws/echo":
            await echo_websocket(scope, receive, send)
        else:
            # Reject unknown WebSocket paths
            event = await receive()
            if event["type"] == "websocket.connect":
                await send({"type": "websocket.close", "code": 4004})
    else:
        pass  # Unknown scope type, ignore


async def handle_http(scope, receive, send):
    await read_body(receive)
    path = scope["path"]

    if path == "/":
        # Serve a simple HTML page with a WebSocket client
        html = """
        <!DOCTYPE html>
        <html>
        <head><title>WebSocket Chat</title></head>
        <body>
            <input id="msg" placeholder="Message..." />
            <button onclick="send()">Send</button>
            <ul id="log"></ul>
            <script>
                const ws = new WebSocket("ws://" + location.host + "/ws/chat");
                ws.onmessage = e => {
                    const li = document.createElement("li");
                    li.textContent = e.data;
                    document.getElementById("log").appendChild(li);
                };
                function send() {
                    const input = document.getElementById("msg");
                    ws.send(JSON.stringify({text: input.value}));
                    input.value = "";
                }
            </script>
        </body>
        </html>
        """.encode("utf-8")

        await send({
            "type": "http.response.start",
            "status": 200,
            "headers": [
                (b"content-type", b"text/html; charset=utf-8"),
                (b"content-length", str(len(html)).encode()),
            ],
        })
        await send({
            "type": "http.response.body",
            "body": html,
            "more_body": False,
        })
    else:
        body = b"Not Found"
        await send({
            "type": "http.response.start",
            "status": 404,
            "headers": [(b"content-type", b"text/plain"),
                        (b"content-length", str(len(body)).encode())],
        })
        await send({"type": "http.response.body", "body": body, "more_body": False})
```

Save, run with `uvicorn`, open `http://localhost:8000` in a browser. Open multiple tabs — they're all in the same chat room.

## The Concurrency Model

When two WebSocket clients are connected, their handler coroutines run concurrently on the event loop:

```
event loop:
    coroutine A (client 1): waiting at "await receive()"
    coroutine B (client 2): waiting at "await receive()"

Client 1 sends a message:
    → coroutine A wakes up
    → calls broadcast()
    → await client_2_send(...)  # sends to client 2
    → coroutine A goes back to "await receive()"

Client 2 sends a message:
    → coroutine B wakes up
    → calls broadcast()
    → await client_1_send(...)  # sends to client 1
    → coroutine B goes back to "await receive()"
```

Each handler runs independently, cooperatively yielding at `await` points. This is why our broadcast works without locks or synchronization: the event loop is single-threaded, so only one coroutine runs at a time. There's no race condition.

This breaks down if you have CPU-bound work in a handler (use `run_in_executor`) or if you use actual threads (use `asyncio.Lock`). But for pure async I/O, the cooperative model is both safe and efficient.

## What Frameworks Add

Starlette provides a `WebSocket` class that wraps the ASGI scope and adds convenience methods:

```python
from starlette.websockets import WebSocket

async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    while True:
        text = await websocket.receive_text()  # or receive_bytes(), receive_json()
        await websocket.send_text(f"Echo: {text}")
```

FastAPI builds on Starlette's WebSocket with dependency injection:

```python
from fastapi import FastAPI, WebSocket

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    await websocket.send_text("Connected!")
    # ...
```

Under the hood, `websocket.accept()` sends `{"type": "websocket.accept"}`, `websocket.receive_text()` awaits `receive()` and extracts the text field, and `websocket.send_text()` sends `{"type": "websocket.send", "text": ...}`.

The raw events are still there. The framework just provides a nicer interface to them.

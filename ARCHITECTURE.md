# Architecture

## The shape of the problem

Chrome, launched with `--remote-debugging-port=N`, exposes HTTP endpoints that list
its targets and hand out a WebSocket URL for each one. Over that socket it speaks a
JSON-RPC dialect: the Chrome DevTools Protocol.

Two kinds of message arrive on the same socket, interleaved:

```jsonc
// a reply — answers a request you made, carries its id
{"id": 7, "result": {"frameId": "..."}}

// an event — unsolicited, arrives whenever, no id
{"method": "Page.loadEventFired", "params": {"timestamp": 8931.4}}
```

That distinction drives the whole design. A reply must be routed back to the caller
that is blocked waiting for it. An event has no caller — something may be waiting for
it, or nothing may be. Two different mechanisms, one socket.

## Layers

```
┌─────────────────────────────────────────────────────┐
│ nanotest.py   collect tests, run them, report        │
├─────────────────────────────────────────────────────┤
│ page.py       goto · click · type · wait · screenshot│
│ browser.py    launch Chrome · create targets · clean │
├─────────────────────────────────────────────────────┤
│ cdp.py        id correlation · event waiters · thread│
├─────────────────────────────────────────────────────┤
│ ws.py         handshake · framing · masking          │
├─────────────────────────────────────────────────────┤
│ socket        TCP                                    │
└─────────────────────────────────────────────────────┘
```

Each layer knows only the one below it.

| Layer | Knows | Must not know |
|---|---|---|
| `ws.py` | sockets, bytes | that CDP exists |
| `cdp.py` | `ws.py`, JSON | pages, browsers, selectors |
| `browser.py` | `cdp.py`, OS processes | frame layout |
| `page.py` | `cdp.py` | that a WebSocket is involved |

The test: `struct.unpack` in `page.py` means the seam has broken.

## Layer 1 — `ws.py`

A WebSocket connection begins as an ordinary HTTP GET carrying `Upgrade: websocket`.
The server answers `101 Switching Protocols` and the *same* TCP connection becomes a
WebSocket. No second connection is opened.

The handshake carries a proof. The client sends 16 random bytes, base64-encoded, as
`Sec-WebSocket-Key`. The server appends a fixed GUID, takes SHA-1, base64-encodes the
result and returns it as `Sec-WebSocket-Accept`. The client recomputes and compares —
which is what distinguishes a real WebSocket server from an HTTP server that happened
to answer 101.

After that, every message is a frame:

```
byte 1   FIN (1) · RSV 1-3 (3) · opcode (4)
byte 2   MASK (1) · payload length (7)
         length == 126 → next 2 bytes are the real length
         length == 127 → next 8 bytes are the real length
next 4   masking key, present only when MASK is set
rest     payload
```

**Every frame a client sends must be masked** — each payload byte XORed with
`key[i % 4]`, using a fresh random 4-byte key. Frames from the server are not masked.
An unmasked client frame gets the connection closed with no explanation.

Three things are easy to skip and all three eventually bite:

- **Fragmentation.** A large message arrives as `FIN=0` followed by continuation
  frames. Screenshots routinely trigger this.
- **Ping.** The server sends ping frames; a client that does not reply with pong gets
  disconnected.
- **The 64-bit length path.** Rarely exercised in small tests, always exercised by a
  full-page screenshot.

## Layer 2 — `cdp.py`

One background reader thread owns the socket. Everything else blocks on events.

```
send(method, params)                    reader thread
  ├── id = next(counter)                  ├── msg = json.loads(ws.recv_text())
  ├── pending[id] = Event()               ├── "id" in msg?
  ├── ws.send_text(json)                  │     └── pending[id].set(result)
  └── wait on that Event  ◄───────────────┘
                                          └── else it is an event
                                                └── wake matching waiters
```

**Register the waiter before triggering the action.** `goto()` must register interest
in `Page.loadEventFired` *first*, then send `Page.navigate`. Reversed, the event can
arrive before anyone is listening and the call hangs forever. This is the bug this
design exists to prevent.

Two listener kinds:

- **one-shot waiters** — `goto` waiting for a load event, removed once fired
- **persistent subscribers** — console message collection, which runs for the life of
  the page

## Layer 3 — `browser.py`

Owns the OS process. Picks a free port by binding to port 0, creates a throwaway
profile directory, launches Chrome, then polls `/json/version` until the debugger
answers — Chrome is not ready the instant `Popen` returns.

Cleanup must survive exceptions: the process killed, the temp profile removed, targets
closed. A driver that leaks Chrome processes across a failing test suite is worse than
no driver.

Targets are created over the browser-level connection (`Target.createTarget`), then
looked up in `/json/list` to get their page-level WebSocket URL. Browser-level and
page-level are different sockets with different capabilities: the browser connection
cannot evaluate JavaScript in a page.

## Layer 4 — `page.py`

CDP has no `click`. A click is composed:

1. `Runtime.evaluate` — one call that finds the element, scrolls it into view, and
   returns the centre of its bounding rect
2. `Input.dispatchMouseEvent` — `mousePressed`, then `mouseReleased`, at that point

Three round trips. Counting round trips rather than milliseconds is the right unit
here; it is what the benchmark work later measures.

Typing dispatches per-character key events rather than assigning to `value`, so
`keydown`/`input` handlers and framework bindings behave as they would for a user.

Waiting never sleeps. Each wait evaluates a DOM predicate against a deadline, or waits
on a protocol event, and raises a descriptive timeout when it expires.

## Planned: the Action / Observation seam

Everything an automated caller needs — page state, available actions, the result of an
action — passes through one serializable boundary:

```python
Action(type="click", target="#submit", value=None)
Observation(url=..., title=..., visible=[...], console_errors=[...])
```

The human-facing API (`page.click(...)`) sits above it; the structured layer sits
below. The test for whether the seam is real: *can this action be written to JSON and
replayed tomorrow?*

This exists so that recording, deterministic replay, and measurement can be added
later without rewriting `page.py`.

## Planned: a second transport

Chrome also speaks CDP over `--remote-debugging-pipe` — file descriptors 3 and 4, with
no TCP, no HTTP upgrade and no frame encoding. Same protocol, different wire.

Because `cdp.py` depends on a transport interface rather than on `ws.py` directly,
adding it is a new file rather than a rewrite. Having both is what makes the two
measurable against each other.

## Decisions

| Decision | Why |
|---|---|
| Hand-written WebSocket instead of a library | The protocol layer is the point of the project |
| Reader thread rather than async | Replies and events are multiplexed on one socket; a blocking API over a background reader is the smaller design |
| Waiter registered before action | Otherwise events fire before anyone listens |
| Click as evaluate + input events | Native focus, pointer events and overlay blocking must behave as they do for a user |
| Transport behind an interface | CDP is being superseded by WebDriver BiDi |
| Poll DOM predicates, never sleep | A sleep is either flaky or slow, usually both |

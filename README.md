# nanodriver

A browser driver written from scratch: an RFC 6455 WebSocket client on a raw TCP
socket, and a Chrome DevTools Protocol client on top of it.

Python standard library only — no Selenium, no Playwright, no dependencies.

> **Status: in progress.** The handshake layer is being built. Nothing to install yet.

## Why

Chrome already exposes a remote-control interface. Playwright and Puppeteer did not
build it — they connect to it. It is JSON-RPC over a WebSocket, and it offers exactly
two primitives that matter:

- run this JavaScript in the page
- deliver this input event at these coordinates

There is no `click` command in the protocol. `click` is something a driver composes
from both. This project composes it by hand, one layer at a time, to see what the
tools above are actually doing.

## Design

Each layer knows only the one beneath it.

```
page.py        goto / click / type / wait / screenshot
cdp.py         JSON-RPC: correlate replies by id, dispatch events
ws.py          RFC 6455: handshake, framing, masking
socket         TCP
```

`ws.py` does not know what CDP is. `page.py` does not know what a frame is. If
`struct.unpack` ever appears in `page.py`, the seam has broken.

That separation is not tidiness. Chrome is migrating from CDP toward WebDriver BiDi;
a clean transport seam means one file changes rather than all of them.

See [ARCHITECTURE.md](ARCHITECTURE.md) for the protocol details and the reasoning
behind each layer.

## Intended API

Not yet implemented — this is the target shape.

```python
from nanodriver import Browser

with Browser.launch(headless=True) as browser:
    page = browser.new_page()
    page.goto("http://localhost:8000/login")

    page.type("#username", "ritik")     # real key events, per character
    page.click("#submit")               # real mouse press/release at the element centre

    page.wait_for_text("#greeting", "Welcome")
    page.screenshot("out.png", full_page=True)

    assert page.console_errors() == []
```

Two rules the implementation holds to:

- **No `sleep()`.** Every wait polls a real DOM predicate against a deadline, or waits
  on a protocol event.
- **Real input events.** `click` resolves the element's centre, scrolls it into view,
  and dispatches `Input.dispatchMouseEvent` — so native focus, pointer events and
  overlay blocking behave the way a user's click does. Typing dispatches key events
  rather than assigning to `value`.

## Not goals

Cross-browser support, a rendering engine, or feature parity with Playwright. This is
a driver built to be understood and measured, not to replace anything.

## License

Apache-2.0

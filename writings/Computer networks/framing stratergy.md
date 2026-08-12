---
date: 2026-07-29
Title:
tags: []
---
Here's the disconnect: `socket.write()` doesn't send bytes onto the network the instant you call it. It hands the bytes to the OS's send buffer. The OS then decides when to actually push a packet out — and it batches. If you call `write()` twice within a few microseconds (like in the demo — two `socket.write()` calls back-to-back with no `await` or delay between them), the OS hasn't even gotten around to sending the first one yet before the second `write()` call happens. So it just bundles both into the same outgoing packet(s).

On the receiving end, the OS handed all those bytes to your process in one go, so you get one `data()` call with both messages stuck together.

"One after another" only guarantees separation if you wait for the round trip:

```js
socket.write(request1);
// wait for response1's data() to actually fire...
socket.write(request2);
```

Here you genuinely can't glue request1 and request2, because request2 doesn't even exist yet when request1's packet goes out there's a real network round-trip in between.

But in the demo, the server wrote both responses without waiting for anything:

```js
socket.write(JSON.stringify({ result: 1 }));
socket.write(JSON.stringify({ result: 2 })); // this ran a few microseconds later, not after a round trip
```

Nothing paused between those two lines. From the CPU's perspective they're basically simultaneous. That's the "one after another" that's fast enough to still get glued.

So the real rule isn't "sequential vs simultaneous" it's "was there a network round-trip gap in between, or not." Anything that fires in a tight loop, a `Promise.all`, back-to-back writes without awaiting an ack, etc. is not safe from gluing, even though your code visually shows them on separate lines "one after another."

Try modifying the demo: add `await Bun.sleep(50)` between the two `socket.write()` calls on the server and re-run it. You should now reliably get two separate `data()` calls — because you've now forced an actual time gap larger than the OS's batching window.

Yes identical problem, because TCP is TCP regardless of language. Go's `net.Conn` is just as much a raw byte stream as Bun's socket. A `conn.Read()` call gives you *whatever bytes happened to arrive*, with zero guarantee it lines up with one client request or one complete message.

```go
buf := make([]byte, 4096)
n, err := conn.Read(buf)
// buf[:n] could be: half a request, one full request,
// or one and a half requests — no guarantee either way
```

The difference: HTTP already solved this for you. When you build an HTTP server "from scratch," you're not inventing your own framing you're implementing HTTP's existing framing rules, which are part of the spec:

1. Read the request line + headers, up to `\r\n\r\n`. That blank line marks the end of headers — this is basically your delimiter/framing boundary for the header section.
2. Then look at the `Content-Length` header. It tells you exactly how many more bytes to read for the body — this is length-prefixed framing, just with the length in a text header instead of 4 raw bytes.
3. Or, if `Transfer-Encoding: chunked` the body itself is self-framed: each chunk is preceded by its own length in hex, followed by `\r\n`, ending with a `0\r\n\r\n` terminator.

So structurally it's the exact same buffering loop you just wrote for the Unix socket:

```go
for {
    n, err := conn.Read(buf)
    if err != nil { break }
    accumulated = append(accumulated, buf[:n]...)

    if !gotHeaders && bytes.Contains(accumulated, []byte("\r\n\r\n")) {
        // parse headers, extract Content-Length
        gotHeaders = true
    }
    if gotHeaders && len(bodySoFar) >= contentLength {
        // full request assembled — handle it
    }
}
```

Same "keep a buffer, don't assume one `Read` = one message" logic just governed by HTTP's own rules (`\r\n\r\n`, `Content-Length`, chunked encoding) instead of a custom 4-byte length prefix you'd invent for your own RPC protocol.

In practice almost nobody hand-rolls this in Go `net/http` already implements exactly this framing logic for you. But if the exercise is "build an HTTP server using only `net.Listen`/`net.Conn`," yes, you'd be re-implementing this buffering-and-boundary-detection dance by hand, same as what you just did with the Unix socket demo.

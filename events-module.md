# EventEmitter, Buffer & Streams — Interview & Programming Reference

Covers: **Module overview · How they work together · EventEmitter · Buffer · Streams basics · Streams deep dive**

```js
const EventEmitter = require("node:events");
const { Buffer }   = require("node:buffer"); // also global
const stream       = require("node:stream");
```

---

# 1. Module Overview

These three are the **foundational trio** of Node's I/O model:

- **EventEmitter** — the *eventing mechanism*. "When X happens, run these listeners."
- **Buffer** — the *binary data container*. Fixed-length raw bytes outside the V8 heap.
- **Streams** — *data moving over time, in chunks*. Built **on top of** the other two.

Almost everything async in Node sits on these: `http` req/res, `fs` read/write streams,
`net` sockets, `process.stdin/stdout`, `zlib`, `crypto` — all are streams, all are EventEmitters,
all move Buffers.

---

# 2. How They Work Together (the key mental model)

A **Stream IS an EventEmitter that carries Buffers**.

```
┌─────────────────────────────────────────────┐
│  STREAM                                       │
│   • extends EventEmitter  → emits "data",     │
│                             "end", "error"…   │
│   • each chunk it emits is a BUFFER           │
└─────────────────────────────────────────────┘
```

Concretely:
```js
const rs = fs.createReadStream("big.log");

rs.on("data", (chunk) => {        // ← EventEmitter API (.on)
  // chunk is a Buffer            // ← Buffer is the payload
  console.log(chunk.length, "bytes");
});
rs.on("end",   () => {});          // EventEmitter event
rs.on("error", (err) => {});       // EventEmitter event
```

So the layering is: **Buffer** holds the bytes → **Stream** chunks them and **emits events** as
each piece arrives → **EventEmitter** is the machinery delivering those events. Understanding
this is the whole section: streams aren't a separate concept, they're *EventEmitter + Buffer
+ backpressure*.

---

# 3. EventEmitter

The pub/sub core of Node. Objects emit named events; listeners react.

## Core API
```js
const ee = new EventEmitter();

ee.on("greet", (name) => console.log("hi", name)); // subscribe (repeating)
ee.once("init", () => console.log("runs once"));    // auto-removed after 1 call
ee.emit("greet", "Sarkhan");                         // fire → calls listeners NOW
ee.off("greet", fn);          // = removeListener
ee.removeAllListeners("greet");
ee.listenerCount("greet");
ee.eventNames();              // ["greet", ...]
ee.prependListener("greet", fn); // add to FRONT of listener array
```

## Emission is SYNCHRONOUS and ordered (interview point)
When you call `emit`, **all listeners run synchronously, in registration order**, before
`emit` returns. EventEmitter is not async by itself — the asynchrony in Node comes from
*when* events are emitted (by I/O), not from the emitter.

```js
ee.on("x", () => console.log("1"));
ee.on("x", () => console.log("2"));
ee.emit("x");
console.log("3");        // → 1, 2, 3   (NOT 3,1,2)
```

## The `"error"` event is special ⚠️
If an `"error"` event is emitted and there is **no listener** for it, Node **throws** and
crashes the process.
```js
ee.on("error", (err) => log(err));   // ALWAYS attach this on real emitters
ee.emit("error", new Error("boom")); // without the listener above → process crash
```

## Memory-leak warning
Default max **10 listeners per event**. Exceeding it logs a warning (a likely leak —
listeners added but never removed).
```js
ee.setMaxListeners(20);              // raise if you genuinely need more
```

## Meta events
`"newListener"` / `"removeListener"` fire when listeners are added/removed — useful for
introspection/lazy setup.

## Modern promise helpers
```js
const { once, on } = require("node:events");

await once(ee, "ready");             // resolve when "ready" fires (or reject on "error")

for await (const [data] of on(ee, "data")) {  // async-iterate events
  // ...
}
```

## Custom emitter (the standard pattern)
```js
class Order extends EventEmitter {
  place() { /* ... */ this.emit("placed", { id: 1 }); }
}
const o = new Order();
o.on("placed", (order) => ship(order));
o.place();
```

---

# 4. Buffer

A **fixed-length sequence of raw bytes**, stored **outside the V8 heap**. Node's way to handle
binary data (files, TCP, images, crypto) — things JS strings can't represent cleanly.

> Buffer is a subclass of `Uint8Array`, so TypedArray methods work too.

## Why it exists
JavaScript historically had no binary type. I/O is inherently binary (bytes on disk/wire).
Buffer gives a memory region for those bytes that isn't subject to JS string encoding or GC
pressure the same way.

## Creating buffers
```js
Buffer.from("hello");              // from string (utf8 by default)
Buffer.from("48656c6c6f", "hex");  // from hex
Buffer.from([72, 73]);             // from byte array → "HI"

Buffer.alloc(10);                  // 10 zero-filled bytes  (SAFE)
Buffer.allocUnsafe(10);            // 10 bytes, NOT cleared (FAST, may hold old memory)
```
> ⚠️ **`allocUnsafe` security note:** it may contain leftover data from previously freed
> memory. Only use it when you immediately overwrite every byte. Default to `alloc`.

## Reading / converting
```js
const buf = Buffer.from("héllo");
buf.toString();          // "héllo"  (utf8)
buf.toString("hex");
buf.toString("base64");
buf.length;              // BYTE length, not character count ⚠️
Buffer.byteLength("héllo"); // bytes a string would occupy
```
> ⚠️ `.length` is **bytes**. `"é"` is 2 bytes in UTF-8 → `Buffer.from("é").length === 2`
> while `"é".length === 1`. Classic off-by-one source.

## Encodings
`utf8` (default) · `hex` · `base64` · `base64url` · `ascii` · `latin1` · `utf16le`.

## Common operations
```js
Buffer.concat([a, b]);             // join buffers
buf.slice(0, 4);                   // ⚠️ shares memory with original (mutations leak!)
buf.subarray(0, 4);                // same shared-memory behavior
Buffer.compare(a, b);              // sort/compare
buf.equals(other);
buf.write("data", offset);
buf.copy(target);
```
> ⚠️ `slice`/`subarray` return a **view into the same memory**, not a copy. Writing to the
> slice changes the original. Use `Buffer.from(buf.subarray(...))` to copy.

## In streams
Stream chunks are Buffers by default. If you call `stream.setEncoding("utf8")` or read in
object mode, you get strings/objects instead.

---

# 5. Streams Basics

A stream processes data **in chunks over time** instead of loading it all into memory.

## Why streams (two wins)
1. **Memory** — handle data larger than RAM (a 10GB file in constant memory).
2. **Time** — start processing the first chunk before the last one arrives (lower latency).
   Plus **composability** via `pipe`.

```js
// ❌ loads entire file into memory
const data = await fs.promises.readFile("10gb.log");

// ✅ constant memory, starts immediately
fs.createReadStream("10gb.log").pipe(process.stdout);
```

## The four stream types
| Type | Direction | Example |
|---|---|---|
| **Readable** | source you read from | `fs.createReadStream`, `http` request, `process.stdin` |
| **Writable** | sink you write to | `fs.createWriteStream`, `http` response, `process.stdout` |
| **Duplex** | both, independent | TCP socket (`net.Socket`) |
| **Transform** | Duplex that modifies | `zlib.createGzip`, `crypto` cipher |

## Readable modes: flowing vs paused
- **Paused** (default) — you pull data explicitly with `.read()`.
- **Flowing** — data is pushed to you via `"data"` listeners; attaching a `"data"` handler
  or calling `.pipe()` switches the stream into flowing mode.

```js
// flowing (push)
rs.on("data", chunk => {});

// paused (pull)
rs.on("readable", () => { let c; while ((c = rs.read()) !== null) {} });
```

## Object mode
By default streams carry Buffers/strings. `{ objectMode: true }` lets a stream carry arbitrary
JS objects — useful for data pipelines (e.g. CSV rows).

---

# 6. Streams Deep Dive

## Backpressure (THE most important stream concept) ⚠️
**Backpressure** = a slow Writable can't keep up with a fast Readable. Without handling it,
data piles up in memory until the process dies.

`pipe`/`pipeline` handle it **automatically**: they pause the readable when the writable's
internal buffer is full, and resume on `"drain"`.

```js
readable.pipe(writable);   // ← backpressure managed for you
```

Manual control (what `pipe` does under the hood):
```js
const ok = writable.write(chunk);
if (!ok) {
  readable.pause();                       // stop reading
  writable.once("drain", () => readable.resume()); // resume when buffer empties
}
```
> `.write()` returns **`false`** when the internal buffer has passed `highWaterMark` — that's
> the signal to stop and wait for `"drain"`.

## `highWaterMark`
The buffer threshold (default 16KB for byte streams, 16 objects in object mode). It's a hint
controlling *when* backpressure kicks in — not a hard limit.

## `pipe` vs `pipeline` (use pipeline)
```js
// pipe: chainable but error handling is awkward (must wire .on('error') on every stream,
// and a mid-pipeline failure can leak file descriptors)
a.pipe(b).pipe(c);

// pipeline: proper error propagation + automatic cleanup. PREFERRED.
const { pipeline } = require("node:stream/promises");
await pipeline(
  fs.createReadStream("in.txt"),
  zlib.createGzip(),
  fs.createWriteStream("out.gz")
);
```
> Interview point: **always prefer `pipeline`** over chained `pipe` for real code — it
> propagates errors from any stage and destroys all streams on failure (no fd leaks).

## Key events by stream type
**Readable:** `data` (chunk arrived) · `end` (no more data) · `error` · `close`
**Writable:** `drain` (buffer emptied, safe to write again) · `finish` (all writes done) ·
`error` · `close`

> `end` vs `finish`: `end` = readable finished **producing**; `finish` = writable finished
> **consuming** (after `.end()` + flush). Don't mix them up.

## Custom streams (implement the underscore methods)
```js
const { Readable, Writable, Transform } = require("node:stream");

// Readable: implement _read, push chunks, push(null) to end
class Counter extends Readable {
  #n = 0;
  _read() { this.#n < 3 ? this.push(String(this.#n++)) : this.push(null); }
}

// Writable: implement _write(chunk, encoding, callback)
class Logger extends Writable {
  _write(chunk, enc, cb) { console.log(chunk.toString()); cb(); }
}

// Transform: implement _transform(chunk, encoding, callback)
class Upper extends Transform {
  _transform(chunk, enc, cb) { cb(null, chunk.toString().toUpperCase()); }
}

// usage
fs.createReadStream("in.txt").pipe(new Upper()).pipe(process.stdout);
```
> Call the `callback` (`cb`) to signal you're done with a chunk — **forgetting `cb()` stalls
> the stream** (a common bug). Pass an error as the first arg to fail the stream.

## Async iteration (modern, clean reading)
```js
for await (const chunk of fs.createReadStream("big.log")) {
  process(chunk);   // backpressure handled by the loop
}
```

## How it all reconnects
A Transform stream is a **Duplex** (EventEmitter) that receives **Buffer** chunks, transforms
them, and pushes Buffers out — emitting `data`/`end`/`error` along the way, with `highWaterMark`
driving backpressure. That single sentence ties the whole section (EventEmitter + Buffer +
backpressure) together.

---

## Cross-cutting interview one-liners

- **What is a stream?** An EventEmitter that moves Buffer chunks over time, with backpressure.
- **Why streams over readFile?** Constant memory + start processing before all data arrives.
- **Four stream types?** Readable, Writable, Duplex, Transform.
- **What is backpressure?** Slow consumer vs fast producer; `pipe`/`pipeline` pause/resume to
  bound memory. `.write()` returning `false` + the `drain` event are the signals.
- **`pipe` vs `pipeline`?** `pipeline` adds error propagation + auto-cleanup → always prefer it.
- **`end` vs `finish`?** Readable done producing vs Writable done consuming.
- **Why does the `error` event matter on EventEmitter?** No `error` listener → Node throws/crashes.
- **Is `emit` sync or async?** Synchronous — listeners run in order before `emit` returns.
- **`Buffer.alloc` vs `allocUnsafe`?** `alloc` zero-fills (safe); `allocUnsafe` is faster but
  may expose old memory — overwrite fully.
- **`Buffer.length`?** Bytes, not characters — multibyte chars (`é`) count as >1.
- **Does `slice` copy?** No — it's a view sharing memory with the original.

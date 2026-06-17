# Node.js Standard Library — Interview & Programming Reference

Covers: **Module system · Process · Errors · File System · Path · Crypto · Child Process · Cluster · Worker Threads**

> Convention used throughout: `node:` prefix on builtins (`require("node:fs")`) — modern, explicit, avoids shadowing by npm packages.

---

# 1. Module Overview

## Three module types
- **Local** — files you write; loaded by relative path (`./utils`). Reusability + separation of concerns.
- **Built-in (core)** — ship with Node (`fs`, `path`, …). No install.
- **Third-party** — from npm, resolved out of `node_modules`.

## CommonJS (CJS) vs ES Modules (ESM)

| | CommonJS | ESM |
|---|---|---|
| Syntax | `require` / `module.exports` | `import` / `export` |
| Loading | Synchronous, runtime | Static, resolved before execution |
| Bindings | Copy of value at import time | **Live bindings** (reflect later changes) |
| `this` at top level | `module.exports` | `undefined` |
| Wrapper | Yes (function wrapper) | No |
| `__dirname`/`__filename` | Injected | Don't exist → use `import.meta` |
| Top-level `await` | No | Yes |

## The CommonJS module wrapper (interview favorite)
Node wraps every CJS file before running it:

```js
(function (exports, require, module, __filename, __dirname) {
  // your code
});
```

Those 5 identifiers are **function parameters Node injects** — not globals, not from any module.
This is also *why* each module has its own scope (function scope) and doesn't pollute global.
It is **not** an IIFE — Node defines the function and calls it, passing the 5 values.

> ESM cannot use this wrapper: `import`/`export` are only legal at the top level, are
> statically analyzed before execution (enabling live bindings + cyclic handling), and
> support top-level `await` — none of which a runtime function call provides.

## `module.exports` vs `exports`
`exports` is just a **reference** to `module.exports`. Reassigning `exports` breaks the link:

```js
exports.foo = 1;            // ✅ works (mutating shared object)
exports = { foo: 1 };       // ❌ no effect — only what's on module.exports is returned
module.exports = { foo: 1 }; // ✅ replaces the export object
```

## require resolution algorithm (order)
1. Core module? (`fs`, `path`) → return it.
2. Starts with `./`, `../`, `/`? → load as file, then as directory (`index.js` / `package.json` `main`).
3. Otherwise → walk up `node_modules` folders from current dir to root.
4. File resolution tries exact, then `.js`, `.json`, `.node`.

## Module caching
Modules are **cached after first load** — `require` returns the same instance every time.
Side effects run once. (Cache key = resolved absolute filename, visible in `require.cache`.)

## Circular dependencies
If A requires B and B requires A, the one required *second* gets a **partial (incomplete) export**
of the first, as it was at that moment. Node doesn't crash — it returns whatever has been
exported so far. Structure code to avoid relying on not-yet-defined exports during load.

---

# 2. Process

`process` is a **global** — no `require` needed. Represents the running Node process.

## Most-used properties
```js
process.argv;        // ["node", "/path/script.js", ...userArgs]  → CLI args
process.env;         // environment variables (object of strings)
process.cwd();       // current working directory (where node was launched)
process.pid;         // process id
process.platform;    // "linux" | "darwin" | "win32" | ...
process.arch;        // "x64" | "arm64" | ...
process.version;     // Node version, e.g. "v22.3.0"
process.versions;    // versions of node, v8, libuv, openssl, ...
process.memoryUsage(); // { rss, heapTotal, heapUsed, external, arrayBuffers }
process.uptime();    // seconds since process start
```

> `process.argv` always has the real user args starting at **index 2** (0 = node binary, 1 = script).

## Standard streams
```js
process.stdout.write("no newline\n");  // stdout (a Writable stream)
process.stderr.write("errors here\n"); // stderr (Writable)
process.stdin.on("data", chunk => {}); // stdin (Readable)
```
`console.log` → `stdout`; `console.error` → `stderr`. Keeping logs vs errors separate matters
for piping/redirection.

## Exiting
```js
process.exit(0);   // 0 = success, non-zero = failure. Forces exit (flush risk!)
process.exitCode = 1; // preferred — set code, let event loop drain naturally
```
> `process.exit()` can truncate pending async writes. Prefer setting `exitCode` and letting
> the process end on its own.

## Signals (graceful shutdown — common interview topic)
```js
process.on("SIGINT",  () => { /* Ctrl+C */ shutdown(); });
process.on("SIGTERM", () => { /* kill / orchestrators */ shutdown(); });
```
Graceful shutdown = stop accepting new work, finish in-flight requests, close DB/server, then exit.

## Global error safety nets
```js
process.on("uncaughtException", (err) => { log(err); process.exit(1); });
process.on("unhandledRejection", (reason) => { log(reason); });
```
> These are **last resorts**, not normal flow control. After `uncaughtException` the process
> is in an undefined state — log and exit, don't try to continue.

## `process.nextTick()` vs microtasks vs `setImmediate` (HIGH-yield interview)
```js
process.nextTick(cb);  // runs BEFORE the event loop continues — highest priority
Promise.resolve().then(cb); // microtask — after nextTick queue, before next phase
setImmediate(cb);      // runs in the "check" phase of the NEXT loop iteration
setTimeout(cb, 0);     // "timers" phase — generally after... but ordering vs setImmediate
                       // is non-deterministic at top level, deterministic inside I/O callbacks
```
**Priority within a tick:** `process.nextTick` queue → Promise microtask queue → continue event loop.
⚠️ Recursive `process.nextTick` can **starve the event loop** (I/O never runs).

## Event loop phases (libuv) — know the order
`timers → pending callbacks → poll → check (setImmediate) → close callbacks`,
with `nextTick` + microtasks drained **between every phase**.

---

# 3. Errors

## Built-in error types
| Type | Thrown when |
|---|---|
| `Error` | Generic base class |
| `TypeError` | Value of wrong type (`null.foo`) |
| `RangeError` | Value out of allowed range (`new Array(-1)`) |
| `ReferenceError` | Using an undeclared variable |
| `SyntaxError` | Invalid JS (often from `JSON.parse`, `eval`) |
| `URIError` | Bad URI encode/decode |
| `AggregateError` | Multiple errors (e.g. `Promise.any` rejection) |

## Anatomy of an Error
```js
err.message;  // human-readable description
err.name;     // "TypeError", etc.
err.stack;    // stack trace string (V8)
err.cause;    // wrapped underlying error (ES2022): new Error("x", { cause: orig })
```

## System errors (from libuv / OS) carry extra fields
```js
err.code;     // "ENOENT", "EACCES", "ECONNREFUSED", "EADDRINUSE", ...
err.errno;    // numeric
err.syscall;  // "open", "connect", ...
err.path;     // offending path (fs errors)
```
> **Check `err.code`, not `err.message`** — messages aren't a stable API; codes are.

## Operational vs Programmer errors (key conceptual distinction)
- **Operational** — expected runtime failures (file missing, network down, bad input). *Handle them.*
- **Programmer** — bugs (undefined is not a function, bad logic). *Fix them; don't try/catch around bugs.*

## Custom errors
```js
class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = "ValidationError";
    this.field = field;
    Error.captureStackTrace?.(this, ValidationError); // clean stack (V8)
  }
}
```

## Error handling patterns by style
```js
// Error-first callback (classic Node)
fs.readFile("x", (err, data) => { if (err) return handle(err); use(data); });

// Promises / async-await
try { const data = await fs.promises.readFile("x"); }
catch (err) { handle(err); }

// Promise combinators
await Promise.allSettled(tasks);  // never rejects; inspect each result
await Promise.any(tasks);         // first fulfilled; rejects w/ AggregateError if all fail
```
> An unhandled rejection inside async code that nobody `await`s → `unhandledRejection`.
> Always attach a `.catch` or `await` in a `try`.

---

# 4. File System (`fs`)

## Three API flavors — know all three
```js
const fs  = require("node:fs");            // callback + sync
const fsp = require("node:fs/promises");   // promise-based (preferred for async)

fs.readFileSync("f.txt", "utf8");          // SYNC — blocks event loop ⚠️
fs.readFile("f.txt", "utf8", (e, d) => {});// CALLBACK — error-first
await fsp.readFile("f.txt", "utf8");       // PROMISE — cleanest with async/await
```
> Sync APIs block the **entire** event loop → only acceptable at startup/CLI scripts, never
> in a request handler.

## Common operations (promise flavor)
```js
await fsp.readFile(path, "utf8");
await fsp.writeFile(path, data);        // overwrites
await fsp.appendFile(path, data);
await fsp.unlink(path);                 // delete file
await fsp.mkdir(dir, { recursive: true });
await fsp.rm(path, { recursive: true, force: true });
await fsp.readdir(dir);                 // list entries
await fsp.stat(path);                   // size, mtime, isFile(), isDirectory()
await fsp.access(path, fs.constants.F_OK); // existence/permission check
await fsp.rename(from, to);
await fsp.copyFile(src, dest);
```

## File descriptors
A **file descriptor (fd)** is an integer handle to an open file from the OS.
`fs.open` gives you one; you must `close` it (the high-level `readFile` etc. manage fds for you).
```js
const fd = await fsp.open("f.txt", "r");  // FileHandle object
const { bytesRead } = await fd.read(buf, 0, 100, 0);
await fd.close();
```

## Flags
`"r"` read · `"w"` write (truncate/create) · `"a"` append · `"r+"` read+write ·
`"w+"` read+write(truncate) · `"a+"` read+append · add `"x"` to fail if exists.

## Streams + backpressure (CRITICAL for large files)
`readFile` loads the **whole file into memory** — bad for big files. Use streams:
```js
const rs = fs.createReadStream("big.log");
const ws = fs.createWriteStream("out.log");
rs.pipe(ws);  // handles backpressure automatically

// modern, with error handling:
const { pipeline } = require("node:stream/promises");
await pipeline(fs.createReadStream("big.log"), fs.createWriteStream("out.log"));
```
> **Backpressure** = when the writable can't keep up with the readable. `pipe`/`pipeline`
> pause the source until the destination drains, preventing unbounded memory growth.
> Manually: `write()` returns `false` when the buffer is full → wait for the `"drain"` event.

## Watching
`fs.watch` (efficient, OS-backed, occasionally fires twice/misses) vs
`fs.watchFile` (polling, more portable, heavier). Prefer `fs.watch` + debounce for tooling.

---

# 5. Path  *(see dedicated file for full depth)*

```js
const path = require("node:path");
```

**`join` vs `resolve` — the #1 question:**
```js
path.join("a","b");      // "a/b"               relative, just concatenates+normalizes
path.resolve("a","b");   // "/cwd/a/b"           always absolute (right→left, cwd fallback)
path.join("/x","/y");    // "/x/y"               '/y' is just a segment
path.resolve("/x","/y"); // "/y"                 '/y' absolute → discards everything left
```

**Decomposition:**
```js
path.basename("/a/b/c.js");      // "c.js"
path.basename("/a/b/c.js",".js");// "c"
path.dirname("/a/b/c.js");       // "/a/b"
path.extname("/a/b/c.js");       // ".js"  (".gitignore" → "", "file." → ".")
path.parse(p);  // { root, dir, base, name, ext }
path.format(obj);                // inverse of parse
```

**Other essentials:**
```js
path.normalize("a//b/../c");     // "a/c"  (no fs access, not absolute)
path.isAbsolute(p);              // boolean
path.relative(from, to);         // relative path between two locations
path.sep;        // "/" or "\\"
path.delimiter;  // ":" or ";"   (PATH var separator)
path.posix.* / path.win32.*      // force a platform's rules
```
> `__dirname`/`__filename` come from the **CJS wrapper**, not `path`. In ESM use
> `import.meta.dirname` / `import.meta.filename` (or `fileURLToPath(import.meta.url)`).
> `path` never touches the filesystem — pure string logic.

---

# 6. Crypto

```js
const crypto = require("node:crypto");
```

## Hashing (one-way, fixed-length digest)
```js
crypto.createHash("sha256").update("data").digest("hex");
```
Use for integrity/checksums. **Not** for passwords directly (too fast → brute-forceable).

## HMAC (hash + secret key → authenticity)
```js
crypto.createHmac("sha256", secret).update("data").digest("hex");
```
Verifies a message wasn't tampered with *and* came from someone holding the key (webhooks, JWT HS256).

## Random
```js
crypto.randomBytes(16);                 // Buffer of CSPRNG bytes (async variant available)
crypto.randomUUID();                    // RFC 4122 v4 UUID string
crypto.randomInt(0, 100);               // unbiased random int
```
> Use `crypto` randomness for anything security-sensitive — **never `Math.random()`**.

## Password hashing (slow + salted, on purpose)
```js
// scrypt (recommended) — memory-hard
const salt = crypto.randomBytes(16);
crypto.scrypt(password, salt, 64, (err, derivedKey) => { /* store salt + key */ });
// or pbkdf2
crypto.pbkdf2(password, salt, 100_000, 64, "sha512", (err, key) => {});
```
Store `salt` + derived key. On login, derive again and compare with **timing-safe** equality.

## Symmetric encryption (same key both ways)
```js
const key = crypto.randomBytes(32);            // AES-256
const iv  = crypto.randomBytes(16);            // unique per message
const cipher = crypto.createCipheriv("aes-256-gcm", key, iv);
const enc = Buffer.concat([cipher.update("secret","utf8"), cipher.final()]);
const tag = cipher.getAuthTag();               // GCM integrity tag
// decrypt: createDecipheriv(...), setAuthTag(tag), update+final
```
> Prefer **authenticated** modes (`aes-256-gcm`) over `aes-256-cbc` — GCM detects tampering.
> IV must be unique per encryption; never reuse with the same key.

## Asymmetric (key pair: public encrypts / verifies, private decrypts / signs)
```js
const { publicKey, privateKey } = crypto.generateKeyPairSync("rsa", { modulusLength: 2048 });
const sig = crypto.sign("sha256", Buffer.from("msg"), privateKey);
const ok  = crypto.verify("sha256", Buffer.from("msg"), publicKey, sig);
```

## Timing-safe comparison (prevents timing attacks)
```js
crypto.timingSafeEqual(Buffer.from(a), Buffer.from(b)); // both buffers must be equal length
```
> Never compare secrets/tokens with `===` — it short-circuits and leaks length/position via timing.

---

# 7. Child Process

```js
const { spawn, exec, execFile, fork } = require("node:child_process");
```
Runs **separate OS processes** (own memory + V8). Used for CLIs, external binaries, offloading work.

## The four methods — when to use which
| Method | Returns data via | Shell? | Use for |
|---|---|---|---|
| `spawn` | **streams** | no (default) | long-running / large output (stream stdout) |
| `exec` | **buffer** (callback) | yes | short commands, convenient, small output |
| `execFile` | buffer | no | run a binary directly (safer/faster than exec) |
| `fork` | **IPC channel** | n/a | spawn another **Node** script w/ message passing |

```js
// spawn — stream output, no shell
const child = spawn("ls", ["-la"]);
child.stdout.on("data", d => process.stdout.write(d));
child.on("close", code => console.log("exited", code));

// exec — buffered, has a shell
exec("ls -la | grep js", (err, stdout, stderr) => {});

// fork — Node-to-Node IPC
const worker = fork("./worker.js");
worker.send({ task: 1 });
worker.on("message", msg => {});  // in worker: process.on("message"), process.send()
```

## Critical gotchas
- **`exec` runs a shell → command-injection risk.** Never interpolate untrusted input into
  an `exec` string. Use `spawn`/`execFile` with an **args array** instead.
- `exec` buffers all output in memory (`maxBuffer`, default ~1MB) → can throw on big output.
  Stream with `spawn` for large output.
- Child processes are **heavy** (full process). For CPU work in-process, see Worker Threads.

---

# 8. Cluster

```js
const cluster = require("node:cluster");
const os = require("node:os");
```
Scales a **network server across CPU cores** by forking the process. Each worker is a full
Node process; they **share the same listening port**.

```js
if (cluster.isPrimary) {
  for (let i = 0; i < os.availableParallelism(); i++) cluster.fork();
  cluster.on("exit", (worker) => cluster.fork()); // auto-restart
} else {
  require("node:http").createServer(handler).listen(3000); // each worker listens
}
```

## How port sharing works (interview depth)
The **primary** creates the listening socket and distributes incoming connections to workers.
Default scheduling is **round-robin** (except on Windows). Workers don't actually bind the port
themselves — the primary hands them connections.

## Cluster vs Worker Threads (常 asked)
| | Cluster | Worker Threads |
|---|---|---|
| Unit | Separate processes | Threads in one process |
| Memory | Isolated per process | Shared (SharedArrayBuffer) |
| Best for | **I/O-bound** servers (scale across cores) | **CPU-bound** tasks |
| Comm | IPC (serialize messages) | MessagePort + shared memory |
| Crash blast radius | One worker dies, others live | Thread crash can affect process |

> **State caveat:** workers don't share memory → in-memory sessions break across workers.
> Use a shared store (Redis) or **sticky sessions** for stateful connections (e.g. WebSockets).

---

# 9. Worker Threads

```js
const { Worker, isMainThread, parentPort, workerData } =
  require("node:worker_threads");
```
**True parallelism for CPU-bound work** inside one process. Each worker has its own V8 isolate
+ event loop, but they can **share memory** via `SharedArrayBuffer`.

```js
// main.js
const worker = new Worker("./heavy.js", { workerData: { n: 40 } });
worker.on("message", result => console.log(result));
worker.on("error", err => {});
worker.on("exit", code => {});

// heavy.js
const { parentPort, workerData } = require("node:worker_threads");
const result = fib(workerData.n);   // blocking CPU work, off the main thread
parentPort.postMessage(result);
```

## Communication
- **`postMessage` / `on("message")`** — structured-clone copy of data (deep copied).
- **Transferables** — move (not copy) an `ArrayBuffer`: `postMessage(buf, [buf])` (zero-copy, source becomes unusable).
- **`SharedArrayBuffer` + `Atomics`** — genuinely shared memory between threads; `Atomics`
  gives lock-free synchronization. The only way to share mutable state.
- **`MessageChannel`** — dedicated bidirectional port pair.

## When to use which (the whole decision tree)
- **CPU-bound** (hashing, image/video processing, big computation) → **Worker Threads**.
- **Scale an I/O server across cores** → **Cluster**.
- **Run an external program / Node script as isolated process** → **Child Process**.
- **I/O-bound** (DB, network, file reads) → you usually need **none** — the async event loop
  already handles concurrency. Threads/processes won't help and add overhead.

> Rule of thumb: *Node is single-threaded for **your JS**, but I/O is already async.*
> Reach for workers only when **JS execution itself** is the bottleneck (CPU), not when waiting on I/O.

---

## Cross-cutting interview one-liners

- **Is Node single-threaded?** Your JS runs on one main thread; libuv uses a **thread pool**
  (default 4, `UV_THREADPOOL_SIZE`) for fs, dns, crypto, zlib. I/O is async, not threaded JS.
- **`setImmediate` vs `setTimeout(fn,0)`?** `setImmediate` = check phase; inside an I/O callback
  it reliably runs first. At top level the order is non-deterministic.
- **`process.nextTick` vs `Promise.then`?** nextTick queue drains first, then microtasks.
- **CommonJS vs ESM bindings?** CJS copies values; ESM exports are **live bindings**.
- **Why streams?** Constant memory + backpressure for data larger than RAM.
- **`exec` vs `spawn`?** exec = shell + buffered (injection risk, size limit); spawn = no shell + streamed.
- **Cluster vs Worker Threads?** Processes for I/O scaling; threads for CPU parallelism + shared memory.

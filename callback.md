# Callbacks — Synchronous vs Asynchronous

A **callback** is a function passed to another function to be called later.
"Sync vs async" is about **when** that "later" happens.

---

## Synchronous callbacks

Run **immediately**, in the same call stack, **before** the outer function returns.

```js
[1, 2, 3].map(x => x * 2);   // arrow fn called right now
[1, 2, 3].forEach(x => {});  // finishes before the next line
[3, 1, 2].sort((a, b) => a - b);
[1, 2, 3].filter(x => x > 1);
```

- Nothing is deferred — by the time `map` returns, every callback already ran.
- Blocks the following code until done.
- Errors propagate normally → a surrounding `try/catch` **catches** them.

---

## Asynchronous callbacks

Queued to run **later**, after current sync code finishes, via the **event loop**.

```js
fs.readFile("f.txt", (err, data) => {});  // runs when I/O completes
setTimeout(() => {}, 0);                  // runs later, even at 0ms
server.on("request", (req, res) => {});   // runs each time the event fires
process.nextTick(() => {});
```

- The outer function returns **first**; the callback runs at a future tick.
- Does **not** block the code after it.
- A `try/catch` around the call does **NOT** catch errors from inside → use the
  **error-first** argument (`err`) or `.catch`.

---

## Side-by-side

| | Synchronous | Asynchronous |
|---|---|---|
| Runs | Immediately, same stack | Later, via event loop |
| Blocks following code? | Yes | No |
| Examples | `map`, `forEach`, `sort`, `filter`, `reduce` | `fs.readFile`, `setTimeout`, event handlers |
| `try/catch` catches errors? | ✅ Yes | ❌ No (use error-first arg / `.catch`) |

---

## The interview gotcha

```js
// ❌ This try/catch CANNOT catch a readFile error
try {
  fs.readFile("missing.txt", (err, data) => {
    // callback runs AFTER the try block already exited
  });
} catch (e) {
  // never reached for the async error
}

// ✅ Async errors arrive through the first argument (error-first pattern)
fs.readFile("missing.txt", (err, data) => {
  if (err) return handle(err);
  use(data);
});
```

---

## Key takeaway

Being a callback says **nothing** about sync vs async.
`map`'s callback is synchronous; `readFile`'s callback is asynchronous.
**The API decides**, not the callback itself.

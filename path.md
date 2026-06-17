# Node.js `path` Module — Interview Reference

> Import first. Use the `node:` prefix (modern convention, makes "builtin not npm package" explicit).
>
> ```js
> const path = require("node:path"); // CommonJS
> import path from "node:path"; // ESM
> ```
>
> Every function here takes and returns `string` (or `ParsedPath` for `parse`/`format`).
> `@types/node` types it all — full autocomplete in TypeScript, nothing to configure.

---

## The big interview question: `join` vs `resolve`

This is THE one. Understand the **model**, don't memorize outputs.

|                    | `path.join()`               | `path.resolve()`                         |
| ------------------ | --------------------------- | ---------------------------------------- |
| What it does       | Glues segments + normalizes | Builds an **absolute** path              |
| Direction          | Left → right                | **Right → left**, stops when absolute    |
| Cares about `cwd`? | No                          | Yes — prepends `process.cwd()` if needed |
| Always absolute?   | No                          | Yes                                      |

```js
path.join("a", "b", "c"); // "a/b/c"               (still relative)
path.resolve("a", "b", "c"); // "/current/cwd/a/b/c"  (always absolute)
```

**The trap — an absolute segment in the middle:**

```js
path.join("/foo", "/bar"); // "/foo/bar"   → '/bar' is just another segment
path.resolve("/foo", "/bar"); // "/bar"       → '/bar' is absolute, everything left is discarded
```

> **Why:** `resolve` processes **right-to-left** and stops the moment it has enough to be
> absolute. Hold that one rule and both functions become 100% predictable.

**When to use which:**

- `join` → combine a known base with relative pieces: `path.join(__dirname, "views", "home.html")`
- `resolve` → you specifically need a guaranteed-absolute path, often relative to `cwd`

---

## Decomposition methods (string surgery)

```js
const p = "/home/sarkhan/app/index.js";

path.basename(p); // "index.js"          → last segment
path.basename(p, ".js"); // "index"             → strip a known suffix
path.dirname(p); // "/home/sarkhan/app" → everything except last segment
path.extname(p); // ".js"               → extension (incl. the dot)
```

**`extname` edge cases interviewers love:**

```js
path.extname(".gitignore"); // ""   → leading dot is NOT an extension
path.extname("file."); // "."  → trailing dot returns just "."
path.extname("a.b.c"); // ".c" → only the LAST extension
path.extname("noext"); // ""
```

---

## `parse` and `format` (inverses)

```js
path.parse("/home/sarkhan/app/index.js");
// {
//   root: "/",
//   dir:  "/home/sarkhan/app",
//   base: "index.js",
//   name: "index",
//   ext:  ".js"
// }
```

`format` rebuilds a path from those parts — great for swapping ONE piece:

```js
const { dir, name } = path.parse("/home/sarkhan/app/index.js");
path.format({ dir, name, ext: ".ts" }); // "/home/sarkhan/app/index.ts"
```

> Mental model: `base = name + ext`. If you pass `base`, it overrides `name`+`ext`.

---

## `normalize` — clean up a messy path

```js
path.normalize("a//b/../c"); // "a/c"
path.normalize("/foo/bar//baz/.."); // "/foo/bar"
path.normalize("./a/./b"); // "a/b"
```

> Resolves `.`, `..`, and duplicate separators — but does **NOT** make the path absolute
> and does **NOT** touch the filesystem (it never checks if the path exists).

---

## `isAbsolute` — boolean check

```js
path.isAbsolute("/home/sarkhan"); // true
path.isAbsolute("app/index.js"); // false
path.isAbsolute("C:\\app"); // true  on Windows, false on POSIX
```

---

## `relative(from, to)` — "how do I get from A to B?"

```js
path.relative("/home/sarkhan", "/home/sarkhan/app/index.js");
// "app/index.js"

path.relative("/home/sarkhan/app", "/home/sarkhan/data/users.json");
// "../data/users.json"   → goes UP and OVER
```

> **Common confusion:** `relative` returns a real relative path (with `..`, separators).
> `basename` returns only the leaf name — it has thrown away all location info.
>
> | Concept       | Example                      | Tells you                        |
> | ------------- | ---------------------------- | -------------------------------- |
> | Absolute path | `/home/sarkhan/app/index.js` | Full location from root          |
> | Relative path | `../app/index.js`            | Location _from another point_    |
> | Basename      | `index.js`                   | Just the final name, no location |

---

## Cross-platform: `sep`, `delimiter`, `posix`, `win32`

This is the **"why does `path` even exist?"** answer: separators and rules differ per OS.

```js
path.sep; // "/"  on POSIX,  "\\"  on Windows  → path segment separator
path.delimiter; // ":"  on POSIX,  ";"   on Windows  → PATH env-var separator

process.env.PATH.split(path.delimiter); // portable way to split $PATH
```

**Force a specific platform's behavior regardless of where code runs:**

```js
path.win32.join("c:", "users", "sarkhan"); // "c:\\users\\sarkhan"
path.posix.join("/var", "log", "app.log"); // "/var/log/app.log"
```

> Use case: a Linux server generating Windows-style paths (or vice versa).

---

## `__dirname` / `__filename` — NOT part of `path`

A frequent interview "gotcha." These are **injected by the CommonJS module wrapper**,
not from the `path` module. No import needed (in CJS).

```js
// CommonJS — available automatically
console.log(__dirname); // "/home/sarkhan/app"        (absolute dir of current file)
console.log(__filename); // "/home/sarkhan/app/index.js" (absolute path of current file)
```

**They DON'T exist in ESM** — using them throws `ReferenceError`. ESM equivalents:

```js
// Node 20.11+ / 21.2+ — simplest
import.meta.dirname;
import.meta.filename;

// portable (works on all ESM versions)
import { fileURLToPath } from "node:url";
import path from "node:path";

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);
```

> `import.meta.url` is a `file://` URL string, NOT a filesystem path —
> that's why you need `fileURLToPath` to convert it.

---

## Quick-fire cheat sheet

```js
path.join(a, b, ...)      // join + normalize, stays relative
path.resolve(a, b, ...)   // build absolute path (right-to-left)
path.normalize(p)         // clean up . .. and // (no fs, not absolute)
path.basename(p[, ext])   // last segment, optionally strip suffix
path.dirname(p)           // parent directory
path.extname(p)           // extension incl. dot
path.parse(p)             // → { root, dir, base, name, ext }
path.format(obj)          // inverse of parse
path.isAbsolute(p)        // boolean
path.relative(from, to)   // relative path from → to
path.sep                  // "/" or "\\"
path.delimiter            // ":" or ";"
path.posix.*              // force POSIX behavior
path.win32.*              // force Windows behavior
```

---

## Common interview Q&A

**Q: Difference between `join` and `resolve`?**
`join` concatenates + normalizes and stays relative; `resolve` always returns an absolute
path, processing right-to-left and falling back to `process.cwd()`. Killer example:
`join("/a","/b") → "/a/b"` but `resolve("/a","/b") → "/b"`.

**Q: Does `path.normalize` check if the file exists?**
No. `path` never touches the filesystem — it's pure string manipulation. Use `fs` for that.

**Q: Are `__dirname`/`__filename` from the `path` module?**
No. They come from the CommonJS module wrapper. They don't exist in ESM (use
`import.meta.dirname`/`import.meta.filename` or derive from `import.meta.url`).

**Q: How do you write cross-platform path code?**
Never hardcode `/` or `\`. Use `path.join`/`path.resolve`, `path.sep`, `path.delimiter`.
Force a platform with `path.posix` / `path.win32` when needed.

**Q: `basename` vs `relative`?**
`basename` = leaf name only (no location). `relative(from, to)` = a real relative path
describing how to travel between two locations (can include `..`).

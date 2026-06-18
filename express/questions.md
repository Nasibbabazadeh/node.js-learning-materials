# Express & Layered Architecture — Q&A

A consolidated reference covering Express fundamentals, the middleware pattern, layered architecture, validation, and error handling.

---

## 1. What three problems does Express solve that you'd otherwise handle manually with the `http` module?

With the raw `http` module you get a single request callback `(req, res)` and nothing else. Express layers conveniences on top of it. The three big ones:

**1. Routing.** With `http` you have one handler and you manually branch on `req.url` and `req.method`:

```js
const server = http.createServer((req, res) => {
  if (req.method === 'GET' && req.url.startsWith('/users/')) { /* parse id yourself */ }
  else if (req.method === 'POST' && req.url === '/users') { /* ... */ }
  else { res.writeHead(404); res.end(); }
});
```

Express gives you a declarative router with method + path matching and path params:

```js
app.get('/users/:id', handler);   // req.params.id is parsed for you
```

**2. The middleware pipeline.** `http` has no concept of a request-processing chain. Express lets you compose cross-cutting concerns (logging, auth, body parsing, CORS) as an ordered stack of functions connected by `next()`. You can't express "run this for every request, then continue" cleanly with raw `http`.

**3. Request/response helpers + body parsing.** With `http`, `req` is a raw stream — to read a JSON body you collect chunks yourself:

```js
let body = '';
req.on('data', chunk => body += chunk);
req.on('end', () => { const data = JSON.parse(body); /* ... */ });
```

And responses are manual (`res.writeHead`, `res.end`). Express adds `express.json()` body parsing plus ergonomic helpers: `req.params`, `req.query`, `req.body`, `res.status()`, `res.json()`, `res.send()`.

> Honorable mentions Express also handles: static file serving, content negotiation, and view/template rendering.

---

## 2. What are the three parameters every middleware function receives? What does calling `next()` do?

Signature: **`(req, res, next)`**

- `req` — the incoming request object
- `res` — the response object
- `next` — a function that passes control to the next middleware in the stack

`next()` hands off to the next middleware/handler in order. Key behaviors:

- `next()` → continue to the next matching middleware/route.
- `next(err)` → skip all remaining *regular* middleware and jump straight to *error-handling* middleware.
- `next('route')` → skip the rest of the current route's handlers (advanced, route-level).
- If you neither call `next()` nor end the response (`res.send`/`res.json`/`res.end`), **the request hangs** — the client waits until it times out.

---

## 3. How is error-handling middleware different from regular middleware? What parameters does it take?

Error-handling middleware takes **four** parameters: **`(err, req, res, next)`**.

```js
app.use((err, req, res, next) => {
  console.error(err);
  res.status(err.status || 500).json({ message: err.message });
});
```

Differences from regular middleware:

- **Arity is the signal.** Express identifies an error handler by `fn.length === 4`. You *must* declare all four params (even if you don't use `next`) or Express treats it as regular middleware.
- **It only runs when an error is passed.** It's triggered by `next(err)` (and in Express 5, also by a thrown/rejected error in an async handler). Normal requests skip it.
- **Registration order matters.** Error handlers should be registered **last**, after all routes and other middleware, so they catch errors from everything above them.

---

## 4. What are the three layers in layered architecture? What does each layer "know about"?

| Layer | Also called | Knows about | Does NOT know about |
|-------|-------------|-------------|---------------------|
| **Presentation** | Controller / HTTP layer | HTTP (req/res, status codes, parsing input, formatting output); calls the Service layer | The database / persistence |
| **Service** | Business logic / domain | Business rules, use cases, orchestration; calls Repositories | HTTP (no `req`/`res`), specific DB drivers |
| **Repository** | Data access / DAL | Persistence — queries, ORM/driver, how to talk to the DB | Business rules, HTTP |

The mental model: **each layer knows only about the layer directly below it.** The controller talks HTTP and delegates to services; services hold the "what should happen" logic and delegate persistence to repositories; repositories know only "how to store and fetch."

---

## 5. Why must dependencies flow downward (Presentation → Service → Repository)? What happens if a service imports a controller?

**Why downward.** Lower layers should be independent of higher ones so your core business logic doesn't depend on the *delivery mechanism* (HTTP) or *infrastructure* (a specific DB). Benefits:

- **Testability** — you can unit-test a service without spinning up Express.
- **Reusability** — the same service can be driven by an HTTP controller, a CLI, a queue worker, or a cron job.
- **Stable core** — the valuable business logic doesn't churn when you swap frameworks or databases.

**If a service imports a controller, three things break:**

1. **Circular dependency** — the controller imports the service and the service imports the controller. This cycle can produce `undefined` exports at load time depending on resolution order (a classic CommonJS footgun).
2. **HTTP coupling** — the service now transitively depends on Express/HTTP concerns, so it can no longer run outside an HTTP context (CLI, worker, tests).
3. **Inverted architecture** — the dependency arrow now points *upward*, which is exactly what layering forbids. The business core becomes a hostage of the delivery layer.

---

## 6. You're switching from MongoDB to PostgreSQL. Which layer(s) need to change in a properly layered application?

**Only the Repository / data-access layer** (plus connection/config setup). Controllers and services depend on the repository's *interface*, not the concrete database, so they stay untouched.

The catch — this clean swap only holds if the repository **returns domain objects / plain data** and doesn't leak DB-specific types upward. If Mongo's `ObjectId`, BSON quirks, or Mongoose document instances have leaked into your service or controller code, you'll have to change those too. That leakage is itself a symptom of imperfect layering — and a good argument for keeping repositories returning clean, framework-agnostic data.

---

## 7. Where should input validation happen in a layered architecture? Why not in the service layer?

**At the edge — the presentation/controller layer**, typically as validation middleware that runs *before* the controller logic:

```js
router.post('/users', validate(CreateUserSchema), userController.create);
```

Why at the edge:

- **Fail fast** — reject malformed requests before they reach business logic.
- **Clean services** — the service can assume it receives valid, well-typed input. It shouldn't be cluttered with "is this a string? is this a valid email format?" checks.
- **Reusability** — keeping HTTP-shaped validation out of the service keeps the service focused and portable.

**Important nuance — two kinds of validation:**

- **Input/shape validation** (syntactic: "is `email` a non-empty valid email string?", "is `age` a number?") → belongs at the **edge**.
- **Business-rule validation / invariants** (semantic: "is this email already taken?", "is the user allowed to do this?", "does this product have enough stock?") → legitimately belongs in the **service** layer, because it needs business context and data access.

So it's not that the service does *no* validation — it does *business* validation, not *input shape* validation.

---

## 8. What's the main advantage of Zod over Joi for TypeScript projects?

**Zod is TypeScript-first: you define the schema once and infer the static type from it** — a single source of truth for both runtime validation and compile-time types.

```ts
import { z } from 'zod';

const CreateUserSchema = z.object({
  email: z.string().email(),
  age: z.number().int().positive(),
});

type CreateUser = z.infer<typeof CreateUserSchema>;
// { email: string; age: number }  — inferred, never hand-written
```

With Joi, the types are bolted on after the fact and you typically maintain your TypeScript interfaces *separately* from the validation schema — so the two can drift out of sync. Zod eliminates that duplication: change the schema, the type updates automatically. (Joi remains a fine, battle-tested choice in plain JS projects; the TS inference story is where Zod wins.)

---

## 9. You have a route handler that throws an error. How do you ensure Express catches it and passes it to error-handling middleware?

It depends on **synchronous vs. asynchronous** and on your **Express version**:

**Synchronous throws** — Express catches them automatically and forwards to error middleware:

```js
app.get('/x', (req, res) => {
  throw new Error('boom'); // auto-caught → error handler
});
```

**Async handlers in Express 4** — a rejected promise / thrown error inside an `async` function is **NOT** caught automatically. You must catch it and call `next(err)`:

```js
app.get('/x', async (req, res, next) => {
  try {
    const data = await service.doThing();
    res.json(data);
  } catch (err) {
    next(err); // hand it to error-handling middleware
  }
});
```

To avoid repeating try/catch everywhere, use a wrapper (`express-async-handler` or your own):

```js
const asyncHandler = fn => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

app.get('/x', asyncHandler(async (req, res) => {
  const data = await service.doThing(); // any rejection → next(err) automatically
  res.json(data);
}));
```

**Async handlers in Express 5** — rejected promises from async middleware/handlers are forwarded to error handlers **automatically**, so you can just `throw` / let it reject without the wrapper.

**Bottom line:** the mechanism is always `next(err)`. Sync errors trigger it for you; in Express 4 you trigger it for async (manually or via a wrapper); in Express 5 it's automatic for both.

---

## 10. Your service layer needs to signal "user not found." Should it throw `new Error('Not found')` or return `res.status(404).json(...)`? Why?

**Throw — and preferably a typed/domain error, not a generic `Error`.** Never touch `res` from the service.

```ts
// errors.ts
export class NotFoundError extends Error {
  status = 404;
  constructor(message = 'Resource not found') {
    super(message);
    this.name = 'NotFoundError';
  }
}

// service
async function getUser(id: string) {
  const user = await userRepo.findById(id);
  if (!user) throw new NotFoundError('User not found');
  return user;
}

// centralized error-handling middleware (controller side)
app.use((err, req, res, next) => {
  if (err instanceof NotFoundError) return res.status(404).json({ message: err.message });
  res.status(500).json({ message: 'Internal server error' });
});
```

Why **not** `res.status(404).json(...)` in the service:

- **It couples the service to HTTP.** `res` is an Express concept. A service that writes HTTP responses can't be reused from a CLI, worker, or test — it breaks layering (see Q4/Q5).
- **It scatters response logic.** HTTP status mapping should live in one place (the error handler), not be duplicated across every service method.

Why **not** a bare `new Error('Not found')`:

- A generic `Error` carries no reliable signal — to map it to 404 the error handler would have to **string-match the message** (`if (err.message === 'Not found')`), which is brittle and easy to break.
- A custom error class lets the handler do a clean `instanceof` check and attach the correct status. The service stays HTTP-agnostic; the controller/middleware owns the translation from *domain error* → *HTTP status*.

---

### One-line summary of the architecture principles running through these answers

> Keep HTTP at the edge, business logic in the middle, persistence at the bottom — dependencies point downward, errors bubble upward as domain objects, and each layer is replaceable without disturbing the others.

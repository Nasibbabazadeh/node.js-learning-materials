# Layered Architecture in Express.js

A practical walk-through of *why* architecture matters, built by evolving one tiny feature — creating a blog post — from a messy route handler into clean, separated layers. All examples are in TypeScript.

---

## Why Do We Need Architecture?

"Architecture" sounds like ceremony you add to impress reviewers. It isn't. Architecture is the set of decisions about **where code lives and what it is allowed to know about**. Good architecture has one job: to make change cheap.

Software is never finished. The database changes. The web framework changes. A "send an email" requirement becomes "send an email *and* a push notification." If those changes ripple through your whole codebase, every change is risky and slow. If they're contained to one file, change is boring — and boring is exactly what you want in production.

The core idea behind layered architecture is **separation of concerns**: each piece of code should have exactly one reason to change. When you respect that, you can swap out a database, a framework, or a notification provider without rewriting your business rules.

We'll prove this by building the *same* feature three times.

---

## The Problem: Creating a Post

Here's the feature, in plain language:

> When a user creates a post, validate the input, save it to the database, send the author a confirmation email, and return the saved post.

Four responsibilities hiding in one sentence: **validation**, **persistence**, **notification**, and **HTTP response**. Watch what happens to those four as the design improves.

---

## Version 1: No Architecture

Everything in the route handler. This is how most of us start, and there's nothing shameful about it — it *works*.

```typescript
import express, { Request, Response } from "express";
import { MongoClient } from "mongodb";
import nodemailer from "nodemailer";

const app = express();
app.use(express.json());

const client = new MongoClient("mongodb://localhost:27017");

app.post("/posts", async (req: Request, res: Response) => {
  const { title, content, authorEmail } = req.body;

  // 1. Validation
  if (!title || title.length < 3) {
    return res.status(400).json({ error: "Title must be at least 3 characters" });
  }
  if (!content) {
    return res.status(400).json({ error: "Content is required" });
  }

  // 2. Persistence
  await client.connect();
  const db = client.db("blog");
  const result = await db.collection("posts").insertOne({
    title,
    content,
    authorEmail,
    createdAt: new Date(),
  });

  // 3. Notification
  const transporter = nodemailer.createTransport({
    host: "smtp.example.com",
    port: 587,
    auth: { user: "noreply@example.com", pass: "secret" },
  });
  await transporter.sendMail({
    from: "noreply@example.com",
    to: authorEmail,
    subject: "Your post is live",
    text: `Your post "${title}" was published.`,
  });

  // 4. HTTP response
  return res.status(201).json({ id: result.insertedId, title, content });
});

app.listen(3000);
```

### What's Wrong Here?

Nothing — *until you have to change it*. The problems are structural, not functional:

- **The route handler knows everything.** It knows MongoDB's API, SMTP details, validation rules, and HTTP status codes. That's four concerns in one function.
- **It's untestable in isolation.** To test the validation logic, you need a running Mongo instance and an SMTP server. You can't check "does a 2-character title get rejected?" without booting the world.
- **It can't be reused.** If you later create posts from a CLI script or a queue worker (not HTTP), you'd copy-paste this whole block, because the logic is welded to `req` and `res`.
- **Every change touches this file.** New database? Edit here. New framework? Edit here. New notification channel? Edit here. The file has *many* reasons to change — the exact opposite of what we want.

---

## Version 2: Simple Refactoring

The obvious first improvement: pull the logic into standalone functions.

```typescript
// post-functions.ts
import { MongoClient } from "mongodb";
import nodemailer from "nodemailer";

const client = new MongoClient("mongodb://localhost:27017");

export function validatePost(title: string, content: string): string | null {
  if (!title || title.length < 3) return "Title must be at least 3 characters";
  if (!content) return "Content is required";
  return null;
}

export async function savePost(title: string, content: string, authorEmail: string) {
  await client.connect();
  const db = client.db("blog");
  const result = await db.collection("posts").insertOne({
    title, content, authorEmail, createdAt: new Date(),
  });
  return { id: result.insertedId, title, content };
}

export async function sendConfirmation(authorEmail: string, title: string) {
  const transporter = nodemailer.createTransport({
    host: "smtp.example.com",
    port: 587,
    auth: { user: "noreply@example.com", pass: "secret" },
  });
  await transporter.sendMail({
    from: "noreply@example.com",
    to: authorEmail,
    subject: "Your post is live",
    text: `Your post "${title}" was published.`,
  });
}
```

```typescript
// routes.ts
import { Request, Response } from "express";
import { validatePost, savePost, sendConfirmation } from "./post-functions";

export async function createPost(req: Request, res: Response) {
  const { title, content, authorEmail } = req.body;

  const error = validatePost(title, content);
  if (error) return res.status(400).json({ error });

  const post = await savePost(title, content, authorEmail);
  await sendConfirmation(authorEmail, title);

  return res.status(201).json(post);
}
```

### Is This Better?

Yes — the route handler is now readable, and `validatePost` is testable on its own. This is real progress, and for a small app it might be *enough*.

But look closely: we've moved the code, not *decoupled* it. `savePost` still hard-codes MongoDB. `sendConfirmation` still hard-codes SMTP. The functions are organized by **the feature** ("post stuff") rather than by **the kind of concern** (data, rules, transport). That distinction is what gets tested next.

### The Test: What If Requirements Change?

Run three change requests through Version 2 and see how much code each one disturbs:

1. **"We're moving from MongoDB to PostgreSQL."** You open `savePost`, which is mixed in the same file as email logic. You rewrite it, and you risk breaking unrelated code in the same module.
2. **"We need to also create posts from a background worker, not just HTTP."** The functions are loose enough to reuse — but they're scattered, and there's no single object that represents "the post workflow," so the worker has to re-assemble the validate → save → notify sequence itself, duplicating orchestration.
3. **"Add a push notification alongside the email."** You edit `createPost` in `routes.ts` to add another call — meaning your **HTTP routing file now knows about notification channels**. The concern leaked upward.

Version 2 reduced the mess but didn't assign clear *responsibilities*. Each function does one thing, but nothing enforces *which layer is allowed to know what*. That's the gap layered architecture closes.

---

## Version 3: Layered Architecture

### What is Three-Layered Architecture?

We split the code into three layers, each with a strict job and a strict rule about what it may depend on:

```
   HTTP request
        │
        ▼
┌─────────────────────┐
│  Presentation Layer │  Controller — talks HTTP. Knows req/res.
│   (Controller)      │  Knows nothing about the database.
└─────────┬───────────┘
          │ plain data in / plain data out
          ▼
┌─────────────────────┐
│ Business Logic Layer│  Service — the rules & orchestration.
│     (Service)       │  Knows nothing about HTTP or the DB engine.
└─────────┬───────────┘
          │ plain data in / plain data out
          ▼
┌─────────────────────┐
│  Data Access Layer  │  Repository — talks to the database.
│    (Repository)     │  Knows nothing about HTTP or business rules.
└─────────┬───────────┘
          │
          ▼
      Database
```

**The dependency rule points one direction only: downward.** A controller may call a service; a service may call a repository. Never the reverse. Each layer exposes plain data (objects, arrays) and hides its implementation details from the layer above.

This is what makes change cheap: each layer can be replaced as long as it keeps the same shape (interface) toward its neighbor.

### Data Access Layer (Repository)

The repository's only job is to talk to the database. It translates between "the rest of the app" and "this specific storage engine." The app above it should not be able to tell whether you're using Mongo, Postgres, or an in-memory array.

We start with an **interface** — the contract — then a Mongo implementation of it.

```typescript
// post.types.ts
export interface Post {
  id: string;
  title: string;
  content: string;
  authorEmail: string;
  createdAt: Date;
}

export interface CreatePostData {
  title: string;
  content: string;
  authorEmail: string;
}

// The contract. Anything that fulfills this can be a post repository.
export interface PostRepository {
  create(data: CreatePostData): Promise<Post>;
  findById(id: string): Promise<Post | null>;
}
```

```typescript
// post.repository.mongo.ts
import { Collection, MongoClient, ObjectId } from "mongodb";
import { Post, CreatePostData, PostRepository } from "./post.types";

export class MongoPostRepository implements PostRepository {
  private collection: Collection;

  constructor(client: MongoClient) {
    this.collection = client.db("blog").collection("posts");
  }

  async create(data: CreatePostData): Promise<Post> {
    const doc = { ...data, createdAt: new Date() };
    const result = await this.collection.insertOne(doc);
    return { id: result.insertedId.toString(), ...doc };
  }

  async findById(id: string): Promise<Post | null> {
    const doc = await this.collection.findOne({ _id: new ObjectId(id) });
    if (!doc) return null;
    return {
      id: doc._id.toString(),
      title: doc.title,
      content: doc.content,
      authorEmail: doc.authorEmail,
      createdAt: doc.createdAt,
    };
  }
}
```

Notice the repository returns a clean `Post` — it converts Mongo's `_id: ObjectId` into a plain `id: string`. The Mongo-ness stops here. Nobody above this file ever sees an `ObjectId`.

### Business Logic Layer (Service)

The service holds the **rules** and **orchestrates the workflow**. Validate, then save, then notify. It depends on *abstractions* (the `PostRepository` interface and a `Notifier` interface), not on concrete Mongo or SMTP classes. This is dependency injection: the service is *given* its collaborators rather than constructing them itself.

```typescript
// notifier.types.ts
export interface Notifier {
  send(to: string, subject: string, message: string): Promise<void>;
}
```

```typescript
// post.service.ts
import { PostRepository, CreatePostData, Post } from "./post.types";
import { Notifier } from "./notifier.types";
import { ValidationError, NotFoundError } from "./errors";

export class PostService {
  constructor(
    private readonly posts: PostRepository,
    private readonly notifier: Notifier,
  ) {}

  async createPost(data: CreatePostData): Promise<Post> {
    // Business rules live here — not in the controller, not in the repo.
    if (!data.title || data.title.length < 3) {
      throw new ValidationError("Title must be at least 3 characters");
    }
    if (!data.content) {
      throw new ValidationError("Content is required");
    }

    const post = await this.posts.create(data);

    await this.notifier.send(
      post.authorEmail,
      "Your post is live",
      `Your post "${post.title}" was published.`,
    );

    return post;
  }

  async getPost(id: string): Promise<Post> {
    const post = await this.posts.findById(id);
    if (!post) throw new NotFoundError(`Post ${id} not found`);
    return post;
  }
}
```

The service has no idea what a `Request` is, and no idea what MongoDB is. It speaks in plain data and throws plain errors. You could unit-test `createPost` with a fake repository and a fake notifier — no database, no SMTP server, no HTTP. That testability is the payoff of the dependency rule.

### Presentation Layer (Controller)

The controller's only job is to translate between HTTP and the service. Read the request, call the service, map the result (or the error) to a status code. It contains **no business rules and no database code**.

```typescript
// post.controller.ts
import { Request, Response, NextFunction } from "express";
import { PostService } from "./post.service";

export class PostController {
  constructor(private readonly service: PostService) {}

  // Arrow methods so `this` stays bound when passed to the router.
  createPost = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const post = await this.service.createPost({
        title: req.body.title,
        content: req.body.content,
        authorEmail: req.body.authorEmail,
      });
      res.status(201).json(post);
    } catch (err) {
      next(err); // hand errors to the central error middleware
    }
  };

  getPost = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const post = await this.service.getPost(req.params.id);
      res.status(200).json(post);
    } catch (err) {
      next(err);
    }
  };
}
```

Finally, **composition** — the one place where the concrete pieces are wired together. This is your "main" file:

```typescript
// app.ts
import express from "express";
import { MongoClient } from "mongodb";
import { MongoPostRepository } from "./post.repository.mongo";
import { EmailNotifier } from "./notifier.email";
import { PostService } from "./post.service";
import { PostController } from "./post.controller";
import { errorHandler } from "./error.middleware";

const client = new MongoClient("mongodb://localhost:27017");

// Wire the layers: repo → service → controller.
const repository = new MongoPostRepository(client);
const notifier = new EmailNotifier();
const service = new PostService(repository, notifier);
const controller = new PostController(service);

const app = express();
app.use(express.json());

app.post("/posts", controller.createPost);
app.get("/posts/:id", controller.getPost);

app.use(errorHandler); // central error handling, registered last

client.connect().then(() => app.listen(3000));
```

This wiring file is the *only* place that names `MongoPostRepository` and `EmailNotifier` directly. That single fact is what makes the next section possible.

---

## The Benefits: What Changes Now?

Re-run the same three change requests from Version 2 — but against the layered design.

### Scenario 1: Switch from MongoDB to PostgreSQL

Write one new file that implements the same `PostRepository` interface:

```typescript
// post.repository.postgres.ts
import { Pool } from "pg";
import { Post, CreatePostData, PostRepository } from "./post.types";

export class PostgresPostRepository implements PostRepository {
  constructor(private readonly pool: Pool) {}

  async create(data: CreatePostData): Promise<Post> {
    const result = await this.pool.query(
      `INSERT INTO posts (title, content, author_email, created_at)
       VALUES ($1, $2, $3, NOW()) RETURNING *`,
      [data.title, data.content, data.authorEmail],
    );
    const row = result.rows[0];
    return {
      id: row.id.toString(),
      title: row.title,
      content: row.content,
      authorEmail: row.author_email,
      createdAt: row.created_at,
    };
  }

  async findById(id: string): Promise<Post | null> {
    const result = await this.pool.query(`SELECT * FROM posts WHERE id = $1`, [id]);
    const row = result.rows[0];
    if (!row) return null;
    return {
      id: row.id.toString(),
      title: row.title,
      content: row.content,
      authorEmail: row.author_email,
      createdAt: row.created_at,
    };
  }
}
```

Then change **one line** in `app.ts`:

```typescript
// const repository = new MongoPostRepository(client);
const repository = new PostgresPostRepository(pgPool);
```

The service, the controller, and every test of your business rules stay **completely untouched**, because they only ever depended on the `PostRepository` interface — never on Mongo.

### Scenario 2: Switch from Express to Fastify

Only the presentation layer talks HTTP, so only the controller and wiring need to change. The service and repository — where your actual logic and data lives — don't move at all.

```typescript
// post.controller.fastify.ts
import { FastifyRequest, FastifyReply } from "fastify";
import { PostService } from "./post.service";

export class FastifyPostController {
  constructor(private readonly service: PostService) {}

  createPost = async (req: FastifyRequest, reply: FastifyReply) => {
    const body = req.body as { title: string; content: string; authorEmail: string };
    const post = await this.service.createPost(body);
    reply.status(201).send(post);
  };
}
```

```typescript
// server.fastify.ts
import Fastify from "fastify";
const app = Fastify();
const controller = new FastifyPostController(service); // same service as before
app.post("/posts", controller.createPost);
```

Swapping the web framework — usually a terrifying, codebase-wide migration — collapses into "rewrite the thin HTTP adapter." Everything below it is framework-agnostic.

### Scenario 3: Add push notifications alongside email

The service depends on the `Notifier` interface, not on email specifically. Add a push implementation, then send through both — and **the controller and routes never learn that anything changed.**

```typescript
// notifier.email.ts
import { Notifier } from "./notifier.types";
export class EmailNotifier implements Notifier {
  async send(to: string, subject: string, message: string) {
    // ...nodemailer SMTP logic...
  }
}

// notifier.push.ts
export class PushNotifier implements Notifier {
  async send(to: string, subject: string, message: string) {
    // ...push provider logic...
  }
}

// notifier.composite.ts — send through several channels at once
export class CompositeNotifier implements Notifier {
  constructor(private readonly notifiers: Notifier[]) {}
  async send(to: string, subject: string, message: string) {
    await Promise.all(this.notifiers.map((n) => n.send(to, subject, message)));
  }
}
```

Wire it in `app.ts` — again, one place:

```typescript
const notifier = new CompositeNotifier([new EmailNotifier(), new PushNotifier()]);
const service = new PostService(repository, notifier);
```

Recall that in Version 2 this same change leaked notification logic into the routing file. Here, the leak is impossible by design: the controller has no path to reach a notifier.

---

## Error Handling Across Layers

A recurring beginner mistake is sprinkling `res.status(400)` throughout the service. But the service shouldn't know HTTP status codes exist — that's a presentation concern. So how does a validation failure deep in the service become a `400` at the edge?

**Layers throw meaning; the presentation layer translates meaning into HTTP.** The service throws a `ValidationError`. It doesn't say "400" — it says "this input was invalid." One central piece of middleware maps each error *type* to a status code.

### The Pattern: Custom Error Classes

Define a small hierarchy of domain errors. Each carries the *semantic* status it corresponds to, but the classes live independently of Express.

```typescript
// errors.ts
export class AppError extends Error {
  constructor(message: string, public readonly statusCode: number) {
    super(message);
    this.name = this.constructor.name;
  }
}

export class ValidationError extends AppError {
  constructor(message: string) {
    super(message, 400);
  }
}

export class NotFoundError extends AppError {
  constructor(message: string) {
    super(message, 404);
  }
}

export class UnauthorizedError extends AppError {
  constructor(message: string) {
    super(message, 401);
  }
}
```

The service throws these freely (you saw it throw `ValidationError` and `NotFoundError` earlier). Controllers don't handle them individually — they just call `next(err)`. One Express error-handling middleware catches everything:

```typescript
// error.middleware.ts
import { Request, Response, NextFunction } from "express";
import { AppError } from "./errors";

export function errorHandler(
  err: Error,
  _req: Request,
  res: Response,
  _next: NextFunction, // the 4-arg signature is what makes Express treat this as error middleware
) {
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({ error: err.message });
  }

  // Unknown error: don't leak internals to the client.
  console.error("Unexpected error:", err);
  return res.status(500).json({ error: "Internal server error" });
}
```

Now error handling is **defined once** instead of repeated in every route. Add a new error type, give it a status code, and it works everywhere automatically. The service stays HTTP-ignorant, and the mapping rule lives in exactly one file.

---

## Suggested Project Structure

Organize folders by **feature first, layer second**. This keeps everything about "posts" together, which scales far better than one giant `controllers/` folder holding 50 unrelated controllers.

```
src/
├── app.ts                      # composition root: wires everything together
├── server.ts                   # starts the HTTP server
│
├── shared/
│   ├── errors.ts               # AppError, ValidationError, NotFoundError...
│   └── error.middleware.ts     # central error handler
│
├── notifications/
│   ├── notifier.types.ts       # Notifier interface
│   ├── notifier.email.ts       # EmailNotifier
│   ├── notifier.push.ts        # PushNotifier
│   └── notifier.composite.ts   # CompositeNotifier
│
└── posts/
    ├── post.types.ts           # Post, CreatePostData, PostRepository interface
    ├── post.repository.mongo.ts    # data access layer
    ├── post.service.ts             # business logic layer
    ├── post.controller.ts          # presentation layer
    ├── post.routes.ts              # route definitions for posts
    └── post.service.test.ts        # tests the service with fake repo + notifier
```

Rules of thumb that the structure encodes:

- **Interfaces (`*.types.ts`) define the contracts between layers.** Implementations depend on interfaces, never the other way around.
- **`app.ts` is the only file that imports concrete implementations** (`MongoPostRepository`, `EmailNotifier`). Everywhere else, depend on the interface. This is the "composition root."
- **A new feature = a new folder** following the same repository / service / controller pattern. Consistency makes the codebase navigable.

When you later move to NestJS, this maps almost one-to-one: NestJS's modules, providers, and controllers are this exact pattern with a dependency-injection container doing the wiring that `app.ts` does by hand here.

---

## Summary

We built one feature three times and watched the cost of change shrink:

| | Where do business rules live? | Cost to swap the DB | Cost to swap the framework | Cost to add a channel |
|---|---|---|---|---|
| **V1 — No architecture** | In the route handler | Rewrite the handler | Rewrite the handler | Rewrite the handler |
| **V2 — Simple refactoring** | In loose functions | Edit a shared file, risk collateral | Edit the route file | Leaks into routing |
| **V3 — Layered** | In the service, isolated | One new file + one line | Rewrite the thin HTTP adapter | One new file + one line |

The principles that got us there:

1. **Separation of concerns** — each layer has exactly one reason to change. HTTP changes hit the controller; storage changes hit the repository; rule changes hit the service.
2. **The dependency rule** — dependencies point downward only (controller → service → repository), and every layer depends on *interfaces*, not concrete implementations.
3. **Dependency injection** — collaborators are passed in, not constructed inside. This is what makes layers swappable and code unit-testable without a real database or network.
4. **Errors carry meaning, not transport** — the service throws typed domain errors; one middleware translates them into HTTP status codes.

Layered architecture is not about adding files for their own sake. It's about drawing boundaries so that when — not if — requirements change, the change has only one place it's allowed to land.

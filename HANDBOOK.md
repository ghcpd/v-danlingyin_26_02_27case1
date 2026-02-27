# Middleware Toolkit

A small, composable middleware pipeline implementation for processing request-style contexts.

This library exposes a lightweight `MiddlewarePipeline` that can run an ordered chain of middleware functions (similar to Express/Koa) with built-in timeout handling and error callbacks.

---

## Quick Start

```ts
import { MiddlewarePipeline, MiddlewareContext } from "./middlewarePipeline";

const pipeline = new MiddlewarePipeline({
  timeoutMs: 2000,
  onError: (err, ctx) => {
    console.error("Pipeline error", err, ctx);
  },
});

pipeline.use(async (ctx, next) => {
  console.log("start", ctx.path);
  await next();
  console.log("end", ctx.path);
});

pipeline.use((ctx) => {
  ctx.state.processed = true;
});

const ctx: MiddlewareContext = {
  path: "/api/hello",
  method: "GET",
  headers: { "content-type": "application/json" },
  state: {},
};

await pipeline.execute(ctx);
```

---

## Core Concepts

### MiddlewarePipeline

`MiddlewarePipeline` is the main class in `middlewarePipeline.ts`. It manages an ordered stack of middleware functions and provides methods to manage and execute the pipeline.

Key methods:

- `use(fn)` – register a middleware function.
- `execute(ctx)` – run the pipeline for a given context.
- `count()` – return how many middleware functions are registered.
- `clear()` – remove all registered middleware.
- `branch()` – create a new `MiddlewarePipeline` instance that shares the same `PipelineOptions` (timeoutMs/onError) and can be mounted as middleware.

### MiddlewareContext

`MiddlewareContext` is the shape of the object passed to every middleware function. It contains information about the request and a mutable `state` object.

Fields:

- `path: string` – the request path.
- `method: string` – the request method (e.g., `GET`, `POST`).
- `headers: Record<string, string>` – HTTP-style headers.
- `body?: unknown` – optional request body.
- `state: Record<string, unknown>` – shared state that middleware can read and update.

### MiddlewareFn

`MiddlewareFn` is the function signature for middleware registered via `pipeline.use()`. Each middleware receives the current `MiddlewareContext` and a `next()` callback.


### PipelineOptions

When creating a new `MiddlewarePipeline`, you can provide `PipelineOptions`:

- `timeoutMs` – maximum time (in milliseconds) a pipeline execution can take. If exceeded, the execute call rejects with a `Pipeline timeout` error and the configured `onError` is invoked.
- `onError` – callback invoked when a middleware or pipeline timeout throws an error. It receives the `Error` and current `MiddlewareContext`.


### NextFunction

`NextFunction` is the type of the `next` argument passed into middleware functions. It returns `Promise<void>` or `void`, allowing both sync and async middleware.

---

## Notes

- The pipeline runs middleware in registration order (FIFO).
- Middleware must call `await next()` to continue to the next middleware.
- If middleware does not call `next()`, the pipeline stops early.
- The pipeline enforces the `timeoutMs` setting on the entire execution.

This documentation is based on the public exports in `middlewarePipeline.ts`.

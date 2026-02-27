# Middleware Toolkit

Composable middleware for request processing.

This handbook covers the public API exposed by `middlewarePipeline.ts` and provides guidance
for authors who want to build a chain of middleware functions that operate on a request-like
context object.

> **Source**: all examples and API information come from `middlewarePipeline.ts` in the
> repository root.

## Key Types and Interfaces

### `NextFunction`

A simple alias for a function that advances the pipeline. Middleware implementations call
`await next()` (or return it directly) to invoke the next item in the stack.

```ts
export type NextFunction = () => Promise<void> | void;
```

### `MiddlewareContext`

The structure passed from the caller to every middleware function. Fields are:

- `path`: the request path being handled (string).
- `method`: HTTP verb or similar (string).
- `headers`: key/value map of request headers.
- `body?`: optional request body (unknown).
- `state`: mutable record for sharing data across handlers.

```ts
export interface MiddlewareContext {
  path: string;
  method: string;
  headers: Record<string, string>;
  body?: unknown;
  state: Record<string, unknown>;
}
```

### `MiddlewareFn`

A middleware function has the signature `(ctx, next)`. It may perform synchronous or
asynchronous work before and/or after calling `next()`.

```ts
export type MiddlewareFn = (
  ctx: MiddlewareContext,
  next: NextFunction
) => Promise<void> | void;
```

### `PipelineOptions`

Configuration options for a `MiddlewarePipeline` instance.

- `timeoutMs` (number): maximum time in milliseconds the pipeline is allowed to run
  before a `Pipeline timeout` error is thrown. Defaults to `5000`.
- `onError` (error, ctx) callback: invoked when an error occurs or the pipeline times out.
  Receives the thrown `Error` and the `MiddlewareContext` at the time of failure. Default is
  a no-op.

```ts
export interface PipelineOptions {
  timeoutMs: number;
  onError: (error: Error, ctx: MiddlewareContext) => void;
}
```

```ts
const defaultOptions: PipelineOptions = {
  timeoutMs: 5000,
  onError: () => {},
};
```

## `MiddlewarePipeline` Class

The main export of the module. You create a pipeline, register middleware with `.use()` and
run it with `.execute(ctx)`.

```ts
export class MiddlewarePipeline {
  private stack: MiddlewareFn[] = [];
  private options: PipelineOptions;
  
  constructor(options?: Partial<PipelineOptions>) { ... }
  use(fn: MiddlewareFn): this { ... }
  async execute(ctx: MiddlewareContext): Promise<void> { ... }
  count(): number { ... }
  clear(): void { ... }
  branch(): MiddlewarePipeline { ... }
}
```

### Constructor

`new MiddlewarePipeline(options?)` accepts a partial `PipelineOptions` object. Any values
not provided are filled in from defaults.

### `.use(fn)`

Add a middleware function to the end of the pipeline. Returns the pipeline instance to allow
chaining.

### `.execute(ctx)`

Run every middleware in registration order using the supplied `MiddlewareContext`. An
internal `NextFunction` is created to advance through the stack. If the pipeline takes longer
than `timeoutMs`, the `onError` callback is invoked and the promise rejects with an error.

Errors thrown by middleware also trigger `onError` before propagating.

### `.count()`

Returns the current number of middleware functions in the pipeline.

### `.clear()`

Removes all middleware from the pipeline, resetting the stack to empty.

### `.branch()`

Creates and returns a new `MiddlewarePipeline` instance configured with the same options but
with an independent middleware stack. The new pipeline can be mounted as middleware in another
pipeline by calling `.use(subPipeline.execute.bind(subPipeline))`.

## Usage Example

```ts
import {
  MiddlewarePipeline,
  MiddlewareContext,
  MiddlewareFn,
  PipelineOptions,
} from "./middlewarePipeline";

const pipeline = new MiddlewarePipeline({ timeoutMs: 2000 });

pipeline.use(async (ctx, next) => {
  console.log("first handler", ctx.path);
  await next();
});

pipeline.use((ctx, next) => {
  ctx.state.processed = true;
  return next();
});

const ctx: MiddlewareContext = {
  path: "/api/data",
  method: "GET",
  headers: {},
  state: {},
};

await pipeline.execute(ctx);
console.log("pipeline finished");
```

## Reference & Further Reading

See the `middlewarePipeline.ts` source file for implementation details. All public types and
the class documented above are exported from that module.

---

*This handbook now provides a complete overview of the toolkit's interface, including types,
configuration options, methods, and an illustrative example.*

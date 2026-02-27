# Middleware Toolkit

Composable middleware for request processing.

## Overview

The Middleware Toolkit provides a flexible, chainable system for processing requests through a pipeline of middleware functions. It allows you to build complex request-handling workflows by composing simple, single-purpose middleware functions that can inspect, modify, or control the flow of request processing.

## Core Concepts

### Middleware Function
A middleware function is a unit of request processing logic that receives a context and a `next` function. It can:
- Inspect the request context (path, method, headers, body)
- Modify the context state
- Call `next()` to pass control to the next middleware
- Skip calling `next()` to terminate the pipeline early

### Pipeline
A pipeline is an ordered collection of middleware functions executed sequentially. When executed, the pipeline creates a chain of responsibility where each middleware decides whether to continue to the next function.

## Types & Interfaces

### `NextFunction`
```typescript
type NextFunction = () => Promise<void> | void;
```
A function passed to each middleware that, when called, executes the next middleware in the pipeline. Can be async or synchronous.

### `MiddlewareContext`
```typescript
interface MiddlewareContext {
  path: string;
  method: string;
  headers: Record<string, string>;
  body?: unknown;
  state: Record<string, unknown>;
}
```
The context object passed to each middleware function. Contains request information and a shared state object for passing data between middleware.

**Properties:**
- `path` — The request path (e.g., `/api/users`)
- `method` — The HTTP method (e.g., `GET`, `POST`)
- `headers` — Request headers as a key-value object
- `body` — Optional request body data
- `state` — A shared object for storing context-specific data across middleware

### `MiddlewareFn`
```typescript
type MiddlewareFn = (
  ctx: MiddlewareContext,
  next: NextFunction
) => Promise<void> | void;
```
The signature for a middleware function. Takes a context and next function, and can be async or synchronous.

### `PipelineOptions`
```typescript
interface PipelineOptions {
  timeoutMs: number;
  onError: (error: Error, ctx: MiddlewareContext) => void;
}
```
Configuration options for the pipeline.

**Properties:**
- `timeoutMs` — Maximum time (in milliseconds) allowed for pipeline execution. Default: 5000. If exceeded, the pipeline throws a "Pipeline timeout" error.
- `onError` — Error handler callback invoked when an error occurs during pipeline execution. Receives the error and the context.

## MiddlewarePipeline Class

The main class for creating and managing middleware pipelines.

### Constructor

```typescript
constructor(options?: Partial<PipelineOptions>)
```

Creates a new pipeline with optional configuration. If options are not provided, defaults are used.

**Example:**
```typescript
const pipeline = new MiddlewarePipeline({
  timeoutMs: 10000,
  onError: (error, ctx) => console.error(`Error: ${error.message}`)
});
```

### Methods

#### `use(fn: MiddlewareFn): this`

Adds a middleware function to the pipeline and returns the pipeline instance (for chaining).

**Example:**
```typescript
pipeline
  .use(async (ctx, next) => {
    console.log(`Incoming ${ctx.method} ${ctx.path}`);
    await next();
  })
  .use(async (ctx, next) => {
    ctx.state.timestamp = Date.now();
    await next();
  });
```

#### `execute(ctx: MiddlewareContext): Promise<void>`

Executes all middleware functions in the pipeline with the given context. The pipeline enforces the timeout and error handling.

**Example:**
```typescript
const context: MiddlewareContext = {
  path: '/api/users',
  method: 'GET',
  headers: { 'content-type': 'application/json' },
  state: {}
};

try {
  await pipeline.execute(context);
  console.log('Pipeline completed successfully');
} catch (error) {
  console.error('Pipeline failed:', error);
}
```

#### `count(): number`

Returns the number of middleware functions currently registered in the pipeline.

**Example:**
```typescript
const pipeline = new MiddlewarePipeline();
pipeline.use(() => {}).use(() => {});
pipeline.count(); // Returns 2
```

#### `clear(): void`

Removes all middleware functions from the pipeline, resetting it to an empty state.

**Example:**
```typescript
pipeline.clear();
pipeline.count(); // Returns 0
```

#### `branch(): MiddlewarePipeline`

Creates a new sub-pipeline with the same options as the current pipeline. Useful for creating nested middleware chains.

**Example:**
```typescript
const mainPipeline = new MiddlewarePipeline({ timeoutMs: 10000 });
const subPipeline = mainPipeline.branch();

mainPipeline.use(async (ctx, next) => {
  subPipeline.use((ctx) => { ctx.state.nested = true; });
  await subPipeline.execute(ctx);
  await next();
});
```

## Complete Usage Example

```typescript
import {
  MiddlewarePipeline,
  MiddlewareContext,
  MiddlewareFn
} from './middlewarePipeline';

// Create a pipeline with error handling
const pipeline = new MiddlewarePipeline({
  timeoutMs: 5000,
  onError: (error, ctx) => {
    console.error(`Error in ${ctx.path}:`, error.message);
  }
});

// Add logging middleware
pipeline.use(async (ctx, next) => {
  const startTime = Date.now();
  console.log(`→ ${ctx.method} ${ctx.path}`);
  await next();
  const duration = Date.now() - startTime;
  console.log(`← ${ctx.method} ${ctx.path} (${duration}ms)`);
});

// Add authentication middleware
pipeline.use(async (ctx, next) => {
  if (!ctx.headers.authorization) {
    throw new Error('Unauthorized');
  }
  ctx.state.user = { id: 1, name: 'John' };
  await next();
});

// Add request validation middleware
pipeline.use(async (ctx, next) => {
  if (!ctx.path.startsWith('/api/')) {
    throw new Error('Invalid path');
  }
  await next();
});

// Execute the pipeline
const context: MiddlewareContext = {
  path: '/api/users',
  method: 'GET',
  headers: { authorization: 'Bearer token123' },
  state: {}
};

await pipeline.execute(context);
console.log('Execution completed, user:', context.state.user);
```

## Key Features

- **Composability**: Chain middleware functions together fluently
- **Error Handling**: Built-in error callback and timeout protection
- **State Sharing**: Pass data between middleware via the context state object
- **Branching**: Create sub-pipelines for nested processing logic
- **Async Support**: Full support for both synchronous and asynchronous middleware

## Source Code

For implementation details, see [middlewarePipeline.ts](middlewarePipeline.ts).
